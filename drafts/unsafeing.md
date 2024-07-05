---
layout: post
title: "rust unsafe code is faster (and other shocking revelations)"
author:
- Octave Larose
---

This is part of a series of blog posts relating my experience pushing the performance of programming language interpreters written in Rust. For added context, read the start of [last week's blog post]({% post_url 2024-05-29-to-do-inlining %}).

In short: we optimize AST (Abstract Syntax Tree) and BC (Bytecode) Rust-written implementations of a Smalltalk-based research language called [SOM](http://som-st.github.io/), in hopes of getting them fast enough to meaningfully compare them with other SOM implementations.

As a general rule, all my changes to the original interpreter (that led to speedups + don't need code cleanups) are present [here](https://github.com/OctaveLarose/som-rs/tree/best). This week's code is [in its own branch](https://github.com/OctaveLarose/som-rs/tree/f9ba61bcc740cafd32b0b1be517a71ecfd9b3bbb), since it relies on ugly code so I don't want it on the main branch (explanations further below).

...and benchmark results are obtained using [Rebench](https://github.com/smarr/ReBench), then I get performance increase/decrease numbers and cool graphs using [RebenchDB](https://github.com/smarr/ReBenchDB). In fact, you can check out RebenchDB in action and all of my results for yourself [here](https://rebench.stefan-marr.de/som-rs/), where you can also admire the stupid names I give my git commits to amuse myself.

## what have i been doing

I've been fixing bugs. I've been informed that having failing tests in your interpreter is in fact _not advisable_, so I had to be a good guy and fix those. I'm not done... But I'm doing something called "performance procrastination" where instead of eating my veggies, I work on the fun bits of my work instead: squeezing more performance out of my systems. I fixed some major oversights in the AST that gave us like 40% median performance. What I thought were deep-rooted, hard to fix design flaws weren't that bad a change, so that's pretty damn cool for me.

## why are we so slow

Related to some work of his, my supervisor recently added a bunch of [the microest benchmarks](https://github.com/SOM-st/SOM/pull/121), designed to test out very basic operations: `ArgRead` does nothing but read arguments over and over again, `FieldReadWrite` just reads/writes to fields, this kind of stuff. And even on basic operations like these, som-rs is super slow: an `ArgRead` has a median runtime of 3.37ms on our fastest bytecode interpreter, versus... 49.23ms on som-rs. Something's way wrong.

So I've been doing profiling to investigate which parts of our BC interpreter are slower than they could be. I focused on the BC for this job since I think it's much better than the AST interpreter, which I know for a fact is slower than it could be. And for the most part, there weren't really any clear answers: we mostly seem to just be spending too much time in the interpreter loop.

I made a new branch `why-are-we-so-slow` and I started experimenting. I did a bunch of small changes that yielded a few % of speedups: for instance, I changed
```rust
let bytecode = unsafe { (*self.current_bytecodes).get(self.bytecode_idx) };
```
...to:
```rust
let bytecode = *(unsafe { (*self.current_bytecodes).get_unchecked(self.bytecode_idx) });
```

We were using `unsafe` since we were storing a pointer to the current bytecodes in the bytecode loop for fast access. Which made me think "we're already using `unsafe`, might as well also call `get_unchecked` for more performance": this code never fails as the bytecode index never increases past the number of bytecodes in a function. That's because the final bytecode in a method or a block is always a `RETURN` of some kind: if a method doesn't return anything explicitly, it returns itself, and we return to the previous function.

Theoretically a `JUMP` could increase the bytecode index to an incorrect value, say 100000... but here's how I draw the line for optimizations: I choose to believe that **my bytecode compiler is trustworthy**, and that any code that is unsafe based on that assertion is a valid choice. I could be shooting myself in the foot since there may be bugs in my compiler, but I think trusting it is a fair assumption to make: it's been proving sturdy enough, and if the bytecode I generate is incorrect, my code will fail miserably anyway or act very weirdly. That's why we have tests for our implementation.

## unsafe frame accesses

Trusting the bytecode compiler means that whenever our bytecode requests a given argument, we know that this argument does in fact exist.  Which means that when we currently look up an argument with this code:

```rust
pub fn lookup_argument(&self, idx: usize) -> Option<Value> {
    self.args.get(idx).cloned()
```

...if we **know** that the bytecode we emitted is correct, then this `get` will never fail: we will _always_ get an argument.

So it now becomes:

```rust
pub fn lookup_argument(&self, idx: usize) -> Value {
    unsafe { self.args.get_unchecked(idx).clone() }
```

...and code that used this function, which looked like `lookup_argument(0).unwrap()`, can now ditch that `unwrap()`. We're no longer using `Option<Value>`, but `Value` directly.

So that was a very simple change I had little faith in: "it's just avoiding a couple of minor runtime checks", I thought. The new code would obviously be faster, but I didn't think it would be *much* faster - like 1% at best, maybe. Turns out I was wrong:

![d55a2d116d21f1ea4c83410246281de3fd2f5a41..49a98d57883b86a137f36bc4130bfe37446d8dad](lookup-arg.png)

That's a 5% median speedup from changing, like, 3 lines of code. Damn. Apparently those `.unwrap()` and `get()` calls add up.

Which opens a lot of possibilities! Why stop at arguments? Let's make local variable reads/writes get the same treatment, and same for literal constants (e.g. accessing string literals from a method). Here's the speedup compared to the branch that already has the argument change:

![bbc03a8bfd4a76cedc15d71626e27ad0b4fddb20..30b66e2855ca804d232d470724b972ce2dd5fe6d](more-unsafe-accesses.png)

Sick. I feel like a proper Rust programmer, using `unsafe` as God definitely did not intend (but he seems powerless to stop me).

I also optimized field accesses in the same way, but that wasn't much of a speedup overall since those are less common.

## unsafe bytecodes

If we're doing `unsafe` stuff, we might as well optimise some bytecodes. For that, I need to find out the most interesting bytecodes to optimize. So I did [some basic instrumenting using the `measureme` crate](https://github.com/OctaveLarose/som-rs/commit/d55a2d116d21f1ea4c83410246281de3fd2f5a41) to be able to print a summary of which bytecodes take up the most execution time.

The biggest offenders are `SEND_X` bytecodes: pretty much everything is a method send, so those get invoked a lot. Not sure they can benefit enormously from `unsafe` though.

Since we're a stack based interpreter, we do a lot of calls to `POP`. It looks like this:

```rust
Bytecode::Pop => {
    self.stack.pop();
}
```

`stack` is a `Vec`, and the code for `Vec::pop()` looks like:

```rust
pub fn pop(&mut self) -> Option<T> {
    if self.len == 0 {
        None
    } else {
        unsafe {
            self.len -= 1;
            core::hint::assert_unchecked(self.len < self.capacity());
            Some(ptr::read(self.as_ptr().add(self.len())))
        }
    }
}
```

Unsurprisingly, it does several things we don't need:
- it checks that the length is not 0: we trust our bytecode, and so this never happens in our world.
- it checks that the length is inferior to its capacity: ...I'm not sure why? It's not like the capacity could ever be inferior to its length. At least I don't know how our code could ever produce that.
- it wraps the output in a `Some` type: we know there will always be an output, we don't need an `Option` type.
- it returns an output in the first place: ...we don't even need to know the output!

So really what we want is just:

```rust
Bytecode::Pop => {
    unsafe {self.stack.set_len(self.stack.len() - 1);}
}
```

I'd do `self.len -= 1` like they do, but `len` is unsurprisingly not public. That would be an odd software engineering decision if it was.

Speedup from that:
![30b66e2855ca804d232d470724b972ce2dd5fe6d..5853a7543fd996a9b02f0d5d79cf2fc866e08237](pop.png)

Hey, we take those. We can do a similar thing for our `DUP` bytecode, which duplicates the last stack element.

```rust
Bytecode::Dup => {
    let value = self.stack.last().cloned().unwrap();
    self.stack.push(value);
}
```

We trust our bytecode etc. etc., there will always be a value on top of the stack, so we end up with this:

```rust
Bytecode::Dup => {
    let value = unsafe { self.stack.get_unchecked(self.stack.len() - 1).clone() };
    self.stack.push(value);
}
```

Man, I wish I had a `last_unchecked()` function, that'd look prettier. Oh well. Speedup from that:

![4c6fa4df8afc7272b1969783f52a64291689d4a1..9d692b2334718b5b16956fc3c62ff05699a9e018](dup.png)

Not much at all, but we also take those. I also spent a bit optimizing random bytecode that had very similar stack operations with peppered uses of `unsafe`. Speedup from that: pretty much none (oops).

## unsafe frame accesses in AST also

That was all in our bytecode interpreter, but we can use `unsafe` similarly in our AST interpreter. In fact, we got some speedups from `unsafe` in the AST [in my last post]({% post_url 2024-06-06-ast-inline-caching %})

So I implemented all those unsafe frame accesses in our AST as well:

![5859af1174f64ce78f300f488321dd5b2bed183f..cd616e5c8ff1daf46ac1616cd29a399d47d9766e](ast.png)

It looks underwhelming, but I assume it brought a similar speedup. It's just that the AST is much slower than the bytecode interpreter at the moment, and so that it wasn't as big a win comparatively.

## final minor cleanups

These `get_unchecked_mut` are nice, but they also make Rust crash unceremoniously, so I'd like to have the option to keep the original not-unsafe code. Rust offers the `debug_assertions` preprocessor directive to check whether we are in a debug version (my normal use case as a developer) or release version (fully optimized, the version we assess performance on) of the interpreter. So through cool Rust syntactic sugar, I can do this:

```rust
pub fn assign_local(&mut self, idx: usize, value: Value) {
    match cfg!(debug_assertions) {
        true => { *self.locals.get_mut(idx).unwrap() = value; },
        false => unsafe { *self.locals.get_unchecked_mut(idx) = value; }
    }
}
```

If we're in the debug version, we call `unwrap()`, otherwise we call `get_unchecked_mut`. Really the debug version should have better error handling than just a dump `unwrap`, but I don't want too much complexity from managing two versions of the interpreter at once.

Neat. Final numbers: [TODO]

And here's where our Rust interpreters currently are compared to the others:


### what have we learned?

So yeah, to no one's surprise, `unsafe` code is faster. That's a no brainer, but I did not expect it to be to that extent from such minimal changes: an `unwrap()` might be extremely cheap, but a million `unwrap()` calls aren't.

I've not explored every possibility in this article. Hoping to find more as I go on profiling my systems and realize "hey, what if we just used a pointer there instead" or