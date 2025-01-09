---
layout: post
title: "The MMTk post that needs a better name"
author:
- Octave Larose
---

This is a blog post about the garbage collection framework [MMTk](https://github.com/mmtk/mmtk-core) and my experience implementing it into our own Rust-based intepreters.
If you only care about the MMTk implementation bits, you can probably jump directly to [this section](#implementation-mmtking) and it will not hurt my feelings.

## what am i reading?

This is part of a series of blog posts relating my experience pushing the performance of programming language interpreters written in Rust. For added context, read the start of [my first blog post]({% post_url 2024-05-29-to-do-inlining %}).

In short: we optimize AST (Abstract Syntax Tree) and BC (Bytecode) Rust-written implementations of a Smalltalk-based research language called [SOM](http://som-st.github.io/), in hopes of getting them fast enough to meaningfully compare them with other SOM implementations. The ultimate goal is seeing how far we can push the AST's performance, the BC being mostly to be a meaningful point of comparison (which means its performance needs to be similarly pushed).

As a general rule, all my changes to the original interpreter (that led to speedups + don't need code cleanups) are present [here](https://github.com/OctaveLarose/som-rs/tree/best)...

...and benchmark results are obtained using [Rebench](https://github.com/smarr/ReBench), then I get performance increase/decrease numbers and cool graphs using [RebenchDB](https://github.com/smarr/ReBenchDB). In fact, you can check out RebenchDB in action and all of my results for yourself [here](https://rebench.stefan-marr.de/som-rs/), where you can also admire the stupid names I give my git commits to amuse myself.

## what have i been doing?
It's been ages since the previous blog post (six months / 7 years or so in dog years), so a lot of things. Most of which are described in hundreds of commits.

The performance of both our AST and BC interpreters is much, much better: something like 100% (yeah!) for the AST and 70% for the BC. I could show them here but I'd rather hold off on addressing them for now, since there's potentially very interesting results for my own research I'd like to expand on in the future (and/or check if I'm messing something up somewhere before drawing conclusions). But at this present moment, AST and BC performance are very similar.

Both interpreters have been worked on extensively and even undertook major reworks, which are explained by this blog post's subject: we now do garbage collection using MMTk, instead of relying on reference counting.

I can motivate this whole blog post by showing the performance we got from adopting tracing GC using MMTk, though!

![bde96ba315406117f87c3246310f6e990df557d4..8aa557da960c81b13750d1cdb3fb9e8928693bc2](/assets/mmtk/total_mmtk_perf.png)

*Double* the speed, for both our AST and BC intepreters. As the ancient nugget of wisdom goes: "good shit".

Though this data can easily be misleading! Importantly: this needed *major* interpreter reworks to function, so a *lot* of changes were made and not all of them are strictly related to MMTk itself or tracing GC. What do you do when you're getting annoyed after a lot of GC debugging? You procrastinate by working on other features in your interpreter[^procrastination]. As a random example, I realized that I'd been using [`Vec::split_off`](https://doc.rust-lang.org/stable/std/vec/struct.Vec.html#method.split_off) in a very common case instead of the better-suited [`Vec::drain`](https://doc.rust-lang.org/stable/std/vec/struct.Vec.html#method.drain). That's a fun fact for you.

I should mention that the majority of such theoretically unrelated changes were *made possible* by changing our pointer representation to accommodate for tracing GC, though, so it is at least related.

But what did we change, and why?

## reference counting: the existing, naive approach
Our interpreters manage a lot of data that we need to be able to reference somehow. Rust offers several pointer types, be it standard references with `&T` (pointers which abide by Rust's safety guarantees), ugly raw pointers `*const T` (bad boys which disregard safety guarantees), or smart pointers, most notably `Box<T>`, `Rc<T>` and `RefCell<T>`. We were using smart pointers a lot since they also tend to *own* the data they reference, which tends to be a lot more convenient than trying to pass references around everywhere.

The basic case is a `Box<T>`: it's just a pointer to the heap, prettified and wrapped by Rust. OK, say in our interpreters any pointer to a `Class` object will be a `Box<Class>`. Instances of this class need a `Class` pointer to read static variables from it and - oh wait, `Box<T>` uniquely owns the data, so these instances can't all own the same `Box<T>`.
So `Rc<T>` (which stands for *reference counting*) comes to the rescue: data can be referenced in several places, since we keep track of its users through a counter. When that counter reaches 0, the final user of that `Rc<T>` has sadly passed away, the data is no longer needed and so it's discarded.

And we also want to be able to write to those static variables, so we need mutable access to `T`, which is what `RefCell<T>` allows. So the final pointer type we choose: `Rc<RefCell<T>>`. Very nice.

We can access and modify pesky types like `Class` from many different places, Rust manages all the memory for us and is happy, and we're happy because Rust is happy and automatically manages all the memory for us and because Rust is happy. Everyone and everything is happy.

Except the runtime performance, which - as a reminder - is the thing we actually care about. Every single time we want to access e.g. a `Class`, we need to A) check that it's not already mutably borrowed by somebody else, B) increment the reference count by 1; then every time we drop a `Rc<RefCell<T>>` after using it, we need to decrement the reference count by 1.

So pointer access is *slow*, and we do a lot of it.
But we need some sort of garbage collection to manage our memory, and if reference counting doesn't cut it, what can we do instead?

## tracing garbage collection
Tracing garbage collection is another form of automatic memory management. It looks through all our objects to determine which ones should be discarded.
Our objects are all a big graph that starts with several *roots*, and GC *traces* it to see which objects are *reachable*: if you can no longer be found in the graph, you're no longer in use and in the garbage you go.

Reference counting has to check whether or not to discard an object everytime we stop using a reference to it.
Tracing GC just waits until we run out of memory, in which case we stop the world to stop running our code so that we can safely trace the object graph without it being modifiable mid-analysis[^stop-the-world].

Added benefit, which turns out to be a major performance win in our case: we manage our data on our own heap, not letting Rust do it.
This is obviously harder to work with since we ditch [ownership](https://doc.rust-lang.org/book/ch04-01-what-is-ownership.html) entirely, which is a core Rust feature. But now allocation much faster by default: we always keep a pointer to the next free element in the heap, and allocation is just bumping it up by the requested amount.

<a name="bump-allocator"></a>
```rust
let new_cursor = self.alloc_bump_ptr.cursor + size;
if new_cursor < self.alloc_bump_ptr.limit {
    let addr = self.alloc_bump_ptr.cursor;
    self.alloc_bump_ptr.cursor = new_cursor;
    addr
} else {
    panic!("out of space, we need to collect garbage first")
}
```
(code simplified from [an example in the MMTk docs](https://docs.mmtk.io/portingguide/perf_tuning/alloc.html#option-3-embed-the-fast-path-struct))

And as it turns out, even without implementing any garbage collection algorithm, we get a lot of performance[^nogc-performance] just from ditching reference counting and having faster allocation! If we manage our heap, but we set no upper limit to it and we keep increasing its size when it becomes full, then...
![bde96ba315406117f87c3246310f6e990df557d4..be7d7c13cd789f2b977aaa66817c2690ec3c005f](/assets/mmtk/nogc-speedup.png)

But we've got no garbage collection happening. This means to function for any program, we need an infinitely-sized heap, and right now our interpreters keep requesting more and more memory until the [OOM killer](https://linuxhandbook.com/oom-killer/) comes in with the steel chair. And to be fair, I *think* [RAM prices are going down](https://pcpartpicker.com/trends/price/memory/), so maybe 1 billion gigs of RAM will be a reasonable ask in 2-3 years.

But until then, we need to actually periodically collect all that accumulating garbage in the heap, and that's when MMTk comes in.

## what's mmtk then?

The [MMTk](https://www.mmtk.io/) project has been around since 2004. Tracing garbage collection is notoriously hard, so MMTk aims to provide a language-agnostic GC framework to do a lot of the heavy lifting. [Here's a short video introduction](https://www.youtube.com/watch?v=0mldpiYW1X4).

Through MMTk's API and bindings, you can incorporate any of several pre-existing *plans* (garbage collection algorithms - mark and sweep, semispace, etc.) into your VM, or create your own plan. So you can easily support various GC algorithms, as opposed to tightly coupling your VM to any chosen GC approach; and all this in any language, AND while being fast. It's really, really cool stuff.

It's got sturdy bindings for [OpenJDK](https://github.com/mmtk/mmtk-openjdk), [V8](https://github.com/mmtk/mmtk-v8), among others like for [Julia](https://github.com/mmtk/mmtk-julia) or [Ruby](https://github.com/mmtk/mmtk-ruby), so it clearly works with much larger virtual machines than mine.

My reasoning behind choosing MMTk for `som-rs` was:
- I want to be able to easily switch between different GC strategies, because modularity makes me happy and so that I have the ability to evaluate the performance impact of different algorithms on my work if I want.
- I want to *not* have to implement these GC strategies myself, since I *know* this would take me ages. I'm more used to [metacompilation systems](https://stefan-marr.de/papers/oopsla-larose-et-al-ast-vs-bytecode-interpreters-in-the-age-of-meta-compilation/) giving me GC for free than me implementing it myself...
- It is written in Rust! The previously mentioned bindings needed to do [FFI](https://en.wikipedia.org/wiki/Foreign_function_interface), but *we* sure don't.

Though with regards to it not taking ages to implement, software is war[^software-is-war] and all warfare is based on deception: therefore I unfortunately deceived myself with unreasonably short time estimates.

So this last point does not account for the fact that implementing GC did in fact take me forever (oops). This is a major motivation for this blog post: sharing my experience and my code, to help others get up to speed faster than I did.

All warfare is also based on planning, so the game plan was:
- get a basic *mark-and-sweep* GC working: it traverses the graph, finds and marks the garbage, sweeps it.
- once that implementation is sturdy, move up to a better-but-more-complex *semispace* plan: heap split in two big chunks, garbage collection copies only live objects from one chunk to the other, that other chunk becomes the new "main" heap until garbage collection occurs again and we do it all the other way around.

Then once I've got a moving GC off the ground, I can also theoretically easily start using a better MMTk-provided GC plan like [Immix](https://www.cs.cornell.edu/courses/cs6120/2019fa/blog/immix/). But I don't actually need a crazy well performing GC, just one that's *good enough* for my interpreters not to be much slower than they could be, in which case no meaningful conclusions could be drawn from their performance. So semispace should be good enough.

## implementation: the new pointer type
For starters, the most obvious change to our existing design is our pointer representation. It's no longer a `Rc<RefCell<T>>` type, but instead just a wrapper around a raw pointer to our own object heap.

```rust
pub struct Gc<T>(*mut T);
```

This mean we don't leverage Rust's safety guarantees at all: it's raw pointer city, and that *sucks*.
There's many blog posts on garbage collection out there that describe smarter approaches than mine: you can read about Saoirse's [shifgrethor](https://without.boats/blog/shifgrethor-iii/),
Core Dumped's [Emacs Lisp VM](https://coredumped.dev/2022/04/11/implementing-a-safe-garbage-collector-in-rust/)
or kyren's [gc-arena](https://kyju.org/blog/rust-safe-garbage-collection/).

An obvious improvement their work features is that they make clever use of lifetimes, while my pointers are always raw, which is a real shame.
The reason is that since it was already a *major* hurdle to get my interpreters working correctly with tracing GC, I wanted the simplest solution... this interpreter is just a solo dev research project.
If myself or others continue improving `som-rs` in the future though, I would very much want this addressed.

Anyway, we've now got our own heap pointer wrapper. Thanks to Rust's type system, we can implement `Deref`/`DerefMut` on it to be able to access its underlying data as if it were a `&T`/`&mut T`, avoiding a lot of boilerplate code.

```rust
impl<T> Deref for Gc<T> {
    type Target = T;

    fn deref(&self) -> &T {
        debug_assert_valid_heap_ptr!(self);
        unsafe { &*self.0 }
    }
}
```

We've obviously been moving away from normal Rust and all its guarantees, which is annoying, but there's another nice example of how we can still leverage the type system to our advantage. A wrapper has the clear benefit of providing explicit, meaninful differentiation between an arbitrary pointer and a pointer to an object in our heap.
But it's got an another major advantage for debugging: we can check that our pointers are valid whenever we dereference them.
Our `debug_assert_valid_heap_ptr` macro verifies that we're indeed pointing into our heap and not to random data, so that we didn't mess up many many cycles ago.

This is especially a godsend when using garbage collection plans that can move your data around: if our safety check sees you're pointing to data that has since been moved, you forgot to update that pointer when moving, and you've got a bug.
More on moving GC later.
(Note for MMTk devs: it would then be a forwarding pointer, but the indirection would be a slowdown, so we want everything copied onto the new heap)

<!--
We've now got full control over our object heap, so we can really do whatever. As an example, since we've also got immutable lists in our interpreter, nothing is stopping us from turning a `Gc<Vec<T>>` into a custom `GcSlice<T>` type:
```rust
pub struct GcSlice<T>(Gc<T>);
```
...which is really just a raw pointer to consecutive `T` types and provides methods to access them. [TODO: do i keep this or does it raise more questions?] -->

## implementation: mmtk'ing

MMTk provides the very helpful [DummyVM](https://github.com/mmtk/mmtk-core/tree/master/docs/dummyvm) to showcase interactions with its API ([described here](https://docs.mmtk.io/api/mmtk/memory_manager/index.html)), which I used as a base - and you probably should too.
It assumes we want to do FFI, and so has a lot of e.g. `extern "C"`, but we don't even need that since we never exit Rust.
This won't be a thorough overview since I don't need to fully leverage MMTk's features, more like a starting point.

To interact with MMTk, we need to create an MMTk instance. For that, we create a builder with our chosen options and our pre-existing `"MarkSweep"` plan, then we call `mmtk_init::<SOMVM>(&builder)` to get a nice `MMTK<SOMVM>` instance.

We also need to declare a *mutator*, a per-thread data structure to manage allocations, so a pretty major one.
We call `mmtk_bind_mutator` to ask MMTk to please give us a `Mutator<SOMVM>` for our current and only execution thread, which we'll store and use to allocate memory. And now we're all sorted.

But that `SOMVM` didn't come out nowhere, did it? That's the name we've given to our VM in our big [`VMBinding`](https://docs.mmtk.io/api/mmtk/vm/trait.VMBinding.html) with MMTk, a binding which looks like this:

```rust
impl VMBinding for SOMVM {
    type VMObjectModel = object_model::VMObjectModel;
    type VMScanning = scanning::VMScanning;
    type VMCollection = collection::VMCollection;
    type VMActivePlan = active_plan::VMActivePlan;
    type VMReferenceGlue = reference_glue::VMReferenceGlue;
    type VMSlot = SOMSlot;
    type VMMemorySlice = mmtk::vm::slot::UnimplementedMemorySlice<SOMSlot>;

    /// Allowed maximum alignment in bytes.
    const MAX_ALIGNMENT: usize = 1 << 6;
}
```

And now there's a lot going on. Here's what everything means:

- `VMObjectModel`: self-explanatory, how objects are represented.
  - The defaults here worked well for me. My main change was implementing the `copy` function, since semispace needs to copy objects.
- `VMScanning`: pretty self-explanatory again, how to scan the object graph. Boils down to implementing two of the most important and complex functions, which we'll elaborate on [in a little bit](#scanning):
  - the `scan_object` function to scan any object in your VM.
  - ...and the `scan_roots` function to tell MMTk what the roots are. Very nice
- `VMCollection`: methods for garbage collection. The only needed ones are:
  - `block_for_gc`: stop the world to let a collection happen. For us, it's pretty much just using a [`CondVar`](https://doc.rust-lang.org/std/sync/struct.Condvar.html) to stop our main and only mutator thread.
  - `stop_all_mutators`: stop all mutator threads. We've only got one and `block_for_gc` has already stopped it, so we just tell MMTk to go ahead.
  - `spawn_gc_thread`: spawn a thread that MMTk will used for GC. In our case, just a glorified `std::thread::spawn` call.
  - `resume_mutators`: notify that same `CondVar` that GC has occurred, and that we can go back to executing our code.
- `VMActivePlan`: methods for the current plan [Question for MMTK authors: I find that a bit unclear - I think the name confuses me. How are mutators related to the plan in specific ways?].
  - just has methods to tell MMTk how to access mutators, so we happily provide it our stored `Mutator<SOMVM>` reference.
- `VMReferenceGlue`: for reference processing (like weak references) and finalizers. I never needed that [Question for MMTk authors: is me not needing it indicative of a VM design flaw?]
- `VMSlot`: we'll talk more about slots [later on](#slots). MMTk provides `SimpleSlot` which works for most cases, but we did need to make our own `SOMSlot` here.
- `VMMemorySlice`: to represent abstract memory slices. As the "unimplemented" indicates, I never implemented that either, though.
- ...and I guess the `MAX_ALIGNMENT` variable for the maximum [alignment](https://en.wikipedia.org/wiki/Data_structure_alignment), and the default is fine by me.

All done. An allocation then looks like:
```rust
let new_ptr: usize = mmtk_alloc(&mut mutator,
                                size_requested,
                                GC_ALIGN, // (alignment): 8 for us
                                GC_OFFSET, // (offset): 0
                                AllocationSemantics::Default);
```

When we request too much memory and GC gets triggered, we've provided everything we need for MMTk to stop the world, scan our object graph and clean up our mess.

Additionally, we say `AllocationSemantics::Default` to indicate we've got no special requirement for this object, but we've got other options such as requesting it to be `Immortal` so that it's never collected, `NonMoving` to never be moved, or `Los` to allocate large objects specifically [TODO: for performance reasons?].
[List of options here](https://docs.mmtk.io/api/mmtk/plan/enum.AllocationSemantics.html).

We can also make allocation faster such as with that [bump allocator](#bump-allocator) example earlier, instead of ever calling `mmtk_alloc` directly.

## scanning


## slots

## GC bugs (oh no)
talk about bugs i had

## More GC bugs (please stop)
People talk about the issues that come from having to scan the stack. I know about these issues because I have read about GC before.
This did not stop me from forgetting all about these issues and losing too much time being confused by those bugs

And that's why I'm grateful I don't handle the GC logic at least.
[TODO READ https://manishearth.github.io/blog/2015/09/01/designing-a-gc-in-rust/]


## in conclusion
Hopefully this shows MMTk might be an interesting choice for your own VM.

My implementation could use a lot more work, but can serve as a starting point. You can find the latest version of it [here](https://github.com/OctaveLarose/som-rs/tree/master/som-gc).

wooo

---

[^procrastination]: Annoying, I know, since it makes potentially interesting data not so usable, and I *might* write a more thorough analysis of each experiment of mine in the future if I can make it interesting. And look, you yourself also procrastinate by implementing new features instead of fixing bugs. We are kindred spirits, you and I.
[^stop-the-world]: Some GCs don't stop the world, but that's a whole other level of complexity so we don't do it. I am but one little guy. There's no reason you couldn't use MMTk for that, though. [Question for MMTk authors: can you give an example, please? Or am I mistaken somehow?]
[^nogc-performance]: Actually unsure why the AST benefited less, but I would assume that's due to it being a lot slower at the time, so GC solving only -some- of its performance problems. [TODO: proofreaders please am i making sense?]
[^software-is-war]: On a metaphysical level, everything is war and love - probably. I had philosophy classes back in high school, so I must know what I'm talking about.