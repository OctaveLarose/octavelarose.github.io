---
layout: post
title: "unsafe code is fast (and other shocking revelations)"
author:
- Octave Larose
---

This is part of a series of blog posts relating my experience pushing the performance of programming language interpreters written in Rust. For added context, read the start of [last week's blog post]({% post_url 2024-05-29-to-do-inlining %}).

In short: we optimize AST (Abstract Syntax Tree) and BC (Bytecode) Rust-written implementations of a Smalltalk-based research language called [SOM](http://som-st.github.io/), in hopes of getting them fast enough to meaningfully compare them with other SOM implementations.

As a general rule, all my changes to the original interpreter (that led to speedups + don't need code cleanups) are present [here](https://github.com/OctaveLarose/som-rs/tree/best). This week's code is [in its own branch](https://github.com/OctaveLarose/som-rs/tree/f9ba61bcc740cafd32b0b1be517a71ecfd9b3bbb), since it relies on ugly code so I don't want it on the main branch (explanations further below).

...and benchmark results are obtained using [Rebench](https://github.com/smarr/ReBench), then I get performance increase/decrease numbers and cool graphs using [RebenchDB](https://github.com/smarr/ReBenchDB). In fact, you can check out RebenchDB in action and all of my results for yourself [here](https://rebench.stefan-marr.de/som-rs/), where you can also admire the stupid names I give my git commits to amuse myself.

# what have i been doing

I've been doing fixing bugs. I've been informed that having failing tests in your interpreter is in fact not advisable, so I had to be a good guy and fix those. I'm not done... But I'm doing something called "performance procrastination" where instead of eating my veggies, I work on the fun bits of my work instead: squeezing more performance out of my systems. I fixed some major oversights in the AST that gave us like 40% median performance - that's pretty damn cool (and pretty indicative that it does not get as much love as the BC interp for now).

I've been doing profiling to investigate which parts of our BC interpreter are slower than they could be. I focused on the BC for this job since I think it's much better than the AST interpreter, which I know for a fact is slower than it could be. And for the most part, there weren't really any clear answers: overall, we just seem to be spending too much time in the interpreter loop.

Related to some work of his, my supervisor recently added a bunch of [the microest benchmarks](https://github.com/SOM-st/SOM/pull/121), designed to test out very basic operations: `ArgRead` does nothing but read arguments over and over again, `FieldReadWrite` just reads/writes to fields, this kind of stuff. And even on basic operations like these, som-rs is quite slow: an `ArgRead` has a median runtime of 3.37ms on our fastest bytecode interpreter, versus... 49.23ms on som-rs. Something's way wrong. So I set out to optimize argument reads specifically.

I made a new branch `why-are-we-so-slow` and I started experimenting. I did a bunch of small changes that yielded a few % of speedups: for instance, I changed
```rust
let bytecode = unsafe { (*self.current_bytecodes).get(self.bytecode_idx) };
```
...to:
```rust
let bytecode = *(unsafe { (*self.current_bytecodes).get_unchecked(self.bytecode_idx) });
```

We were using `unsafe` since we were storing a pointer to the current bytecodes in the bytecode loop for fast access. Which made me think "we're already using `unsafe`, might as well also call `get_unchecked` for more performance": this code never fails as the bytecode index never increases more than the number of bytecodes in a function. That's because the final bytecode is always a `RETURN_LOCAL`.

Theoretically a `JUMP` could increase the bytecode index to an incorrect value, say 100000... but here's how I draw the line for optimizations: **my bytecode compiler is trustworthy**, and unsafe code is a valid choice based on that. I might be shooting myself in the foot: I'm only human and there may be bugs in my compiler, but I think this is a fair assumption to make. [TODO justify better]

Which means that when we currently look up an argument with this code:
```rust
pub fn lookup_argument(&self, idx: usize) -> Option<Value> {
    self.args.get(idx).cloned()
```

...if we **know** that the bytecode we emitted is correct, then this `get` will never fail: we will **always** get an argument.

So it now becomes:

```rust
pub fn lookup_argument(&self, idx: usize) -> Value {
    unsafe { self.args.get_unchecked(idx).clone() }
```

...and code that looked like `lookup_argument(0).unwrap()` can now ditch the `unwrap()`. We're no longer using `Option<Value>`, but `Value` directly.

So that was a very simple change I had little faith in. The new code is obviously faster, but I didn't think it would be *much* faster - 1% at best, maybe. Turns out I was wrong:

![d55a2d116d21f1ea4c83410246281de3fd2f5a41..49a98d57883b86a137f36bc4130bfe37446d8dad](lookup-arg.png)

That's a 5% median speedup from changing, like, 3 lines of code. Damn.


So why stop there?

![bbc03a8bfd4a76cedc15d71626e27ad0b4fddb20..30b66e2855ca804d232d470724b972ce2dd5fe6d](more-unsafe-accesses.png)

TODO POP

TODO more operations (to actually implement)

These `get_unchecked_mut` are nice, but they also make Rust crash unceremoniously, so I'd like to have the option to keep the original not-unsafe code. Rust offers the `debug_assertions` preprocessor directive to check whether we are in a debug (my normal use case) or release (fully optimized, the version we check performance on) version of the interpreter. So through syntactic sugar, I can do this:

```rust
pub fn assign_local(&mut self, idx: usize, value: Value) {
    match cfg!(debug_assertions) {
        true => { *self.locals.get_mut(idx).unwrap() = value; },
        false => unsafe { *self.locals.get_unchecked_mut(idx) = value; }
    }
}
```

Neat. Final numbers: [TODO]

### what have we learned?

`unsafe` code is faster. That's a no brainer, but I did not expect it to be to that extent.