---
layout: post
title: "Inline caching in our AST interpreter"
author:
- Octave Larose
---

This is part of a series of blog posts relating my experience pushing the performance of programming language interpreters written in Rust. For added context, read the start of [last week's blog post]({% post_url 2024-05-29-to-do-inlining %}).

In short: we optimize AST (Abstract Syntax Tree) and BC (Bytecode) Rust-written implementations of a Smalltalk-based research language called [SOM](http://som-st.github.io/), in hopes of getting them fast enough to meaningfully compare them with other SOM implementations.

As a general rule, all my changes to the original interpreter (that led to speedups + don't need cleaning up) are present [here](https://github.com/OctaveLarose/som-rs/tree/best). This week's code is [in its own branch](https://github.com/OctaveLarose/som-rs/tree/f9ba61bcc740cafd32b0b1be517a71ecfd9b3bbb), since it relies on ugly code so I don't want it on the main branch (explanations further below).

...and benchmark results are obtained using [Rebench](https://github.com/smarr/ReBench), then I get performance increase/decrease numbers and cool graphs using [RebenchDB](https://github.com/smarr/ReBenchDB). In fact, you can check out RebenchDB in action and all of my results for yourself [here](https://rebench.stefan-marr.de/som-rs/), where you can also admire the stupid names I give my git commits to amuse myself.

### inline caching?

Inline caching is a very widespread optimization in dynamic programming language implementation. If you're reading this kind of blog, chances are you've already heard of it. Say you have this code:

```
someClass := getSomeClassSomehow.
someClass doSomethingWith: someArg.
```

Since we're working with a dynamic language, `someClass` can be any class at all: it can be `True`, it can be `Integer`, it can be `Whatever`, etc. Whenever you've got a `doSomethingWith` method call (a.k.a a "message": sending the receiver the message "please execute your `doSomethingWith` method"), you can't make any assumptions at compile-time about what class the method is going to get invoked on, therefore what version of `doSomethingWith` you should use: who knows what `getSomeClassSomehow` did and what it returned, really. You only find that out when it's actually executed.

This is annoying since that means that at every method call site, once you know what the class is, you need to look up the method in that class to be able to then execute it. That takes a bit of time for every single call, and basically everything under the sun is a call in SOM, so that's kinda slow.

So why not cache the result of those lookups, so that they don't have to be done every single time? That'd make it so that whenever we call anything, we check our cached method and we invoke that instead of looking it up.

Obvious caveat: if the method call eventually has a different receiver class, the method is wrong (it belongs to another class). So we need to invalidate the cache...
- ...or better: we make it so that our cache has several entries, and cache all possible receivers, so that we account for both the old and new receivers!

What if there's so many receivers that caching is impractically expensive?
- this is not so much of an issue in practice! As it turns out, a method call is rarely invoked with that many different classes: most calls are *monomorphic*, a fancy term for saying "only one possible caller"..
  - We wrote a paper in 2022 on the behavior of Ruby codebases, [which you can read here](https://stefan-marr.de/downloads/dls22-kaleba-et-al-analyzing-the-run-time-call-site-behavior-of-ruby-applications.pdf), in which we observed about 98% of the call-sites in large benchmarks to be monomorphic. Granted this is for the Ruby language and not SOM, but since they're both highly dynamic and similar in terms of features (e.g. objects and inheritance, lambdas/closures, non-local returns), we argue they're comparable.

- most inline caching implementations add a way to declare a callsite as *megamorphic* (fancy term for "an impractically large amount of possible callers"): if we've observed way too many receivers in the past, we stop caching and looking up entries entirely, only doing a generic lookup from now on.

Cool. And those caches may as well be stored at the call sites themselves, therefore be *inline*, hence the name.

### inline caching in our bytecode interpreter

We've already implemented in the bytecode interpreter, and looks like this:

```rust
fn resolve_method(frame: &SOMRef<Frame>, class: &SOMRef<Class>, signature: Interned, bytecode_idx: usize) -> Option<Rc<Method>> {
    let mut inline_cache = unsafe { (*frame.borrow_mut().inline_cache).borrow_mut() };

    let maybe_found = unsafe { inline_cache.get_unchecked_mut(bytecode_idx) };

    match maybe_found {
        Some((receiver, method)) if *receiver == class.as_ptr() => {
            Some(Rc::clone(method))
        }
        place @ None => {
            let found = class.borrow().lookup_method(signature);
            *place = found.clone().map(|method| (class.as_ptr() as *const _, method));
            found
        }
        _ => class.borrow().lookup_method(signature),
    }
}
```

Every bytecode has its own associated inline cache. It's empty most of the time - you don't need to cache anything for a `POP` bytecode - but will get entries for `SEND` bytecodes. Whenever you resolve a method, you check whether the inline cache has an entry for this bytecode, and what the cached receiver and methods are: if the cached receiver is the same as the current receiver, then we can just call the method. Otherwise, we do a lookup and put the method in the cache.

Note that this is an inline cache of size one. That's because [adding more entries was not that beneficial to performance on our benchmarks](https://github.com/Hirevo/som-rs/pull/31). We'll try varying the number of entries in the AST implementation, and see if it's the same as in the BC.

Rust side note: this is also a good example of us using pointers + `unsafe` to sneak in some performance:
1. `*frame.borrow_mut().inline_cache`: inline caches are stored in `Method` structs directly. Frames keep a pointer to them for fast access, and dereferencing a pointer is unsafe.
We could theoretically use `&` (i.e. a standard Rust reference) instead, but those come with the burden of informing the compiler about lifetimes, which is far from straightforward in this case; we the (very) smart programmers know that a method and its inline cache will definitely outlive any frame that relies on them.
1. `inline_cache.get_unchecked_mut(bytecode_idx)`: we know there's as many inline cache entries as there are bytecodes, so we may as well avoid the safety check a regular `get()` call would induce.

### rust hates AST interpreters

Now we want to implement it in the AST. Unfortunately for us, this may be a case where the choice of the Rust language may not be the best.

Our AST is currently immutable. It just so happens that [self-optimizing ASTs are key to good performance](https://dl.acm.org/doi/10.1145/2384577.2384587), so we've got an issue.

To implement inline caching, we'd want to implement the cache _inline_ in the AST directly, so e.g. have a `MethodCallNode` that we modify to cache stuff directly in it. That requires said node to have a mutable state; and if you know anything about Rust, you can foresee how that could easily cause issues.

Easy fix in theory: nodes get invoked with `fn evaluate(&self, ...)` - instead let's make them use `fn evaluate(&mut self, ...)` so that they can modify their content when executed! And we change many things around and we run it aaaaand:

```rust
thread 'main' panicked at somewhere/in/my/code
already mutably borrowed: BorrowError
```

...aaand that makes a lot of sense since Rust ownership is all about either having many immutable borrows or ONE mutable borrow. If whenever we call a method, we borrow it mutably, then a simple recursive call from inside that method will need to also mutably borrow the same method, and we're already toast - and that's one of several potential `BorrowError` we could see.

Changing the design of the AST interpreter so that it can be self-modifying has been in my todo list since the start of this project. It's hard if just to wrap my head around how it would be best to go about it. So let's circumvent that for now: the current quick """fix""", is... using a lot of raw pointers instead of safe `Rc<RefCell<T>>` types, and peppering the code with a million uses of `unsafe`. Is this bad? Yeah! Does it work? Also yeah!

No more Rust compiler guarantees though, and that's kind of Rust's whole thing, so we might as well be working with C at this point. Shame. This is absolutely something that I want to fix, and will fix in the future - and it should make for a decent blog post when I do.

### unsafe is actually good?

Making the interpreter unsafe is a mild speedup:

![unsafe-interp](/assets/inline-caching/unsafe-interp.png)

...very likely from avoiding safety checks and no longer having to increase reference counters. This makes me think that even if I stop using this horribly unsafe AST interpreter, there's potential for some mindful usage of `unsafe` to get little speedups like this.

As time goes on, I'm probably going to have to lean more and more on the `unsafe` side to get more performance out of our systems. It'll be interesting to find out how much, exactly: would Rust even still be that interesting a language choice if we end up having to omit most of its safety guarantees?

My expectation is that I shouldn't care at all, and that I'll never get to the point that most of the codebase uses `unsafe`. In the work of [Yi Lin et al. on high performance garbage collection](https://www.steveblackburn.org/pubs/papers/rust-ismm-2016.pdf), they found that few uses of `unsafe` were necessary to still get high performance. This is for garbage collection and not PL implementation like we're doing, but it's similar enough to make me believe that good software engineering can circumvent abusing ugly unsafe code.

### rust semantics also hate self-optimizing nodes

The way I wanted to implement inline caching mirrors how we do it in our other AST interpreters: replacing an unoptimized `NormalMessageNode` with a *specialized*, optimized `CachedMessageNode`. Something like this:

```rust
#[derive(Debug, Clone, PartialEq)]
pub enum MessageEnum {
    Uninitialized(Message),
    Cached(CachedMethodDef, ClassIdentifier, Message)
}
```

Turning normal `Uninitialized` messages into `Cached` ones, which are the same but with a cached method and class, and still able to call the generic case with their `GenericMessage` entry.

So to replace an uninitialized node, we'd do something like this:
```rust
*self = ast::Message::Cached(method_def, class_identifier, self.message);
```

Sounds good, right? We replace it and transfer ownership of the messag... wait this code doesn't work because I can't transfer ownership in Rust actually oops

Really, it should be allowed: I want to tell Rust to transfer ownership of the `GenericMessage` from the node to its new, optimized version. My understanding is that this fails because creating this new node and assigning it to `self` are two distinct operations: we create a new node that takes ownership of `self.message` and THEN make `self` that node, when really `self` keeps ownership of the message throughout the whole thing.

If anyone knows how this is achievable, I'd love to know. Is there a way to use `std::mem::replace` somehow, or has someone made some crate that addresses this?

### rust-friendly working version

Self-replacement is a no-go, so we're no longer replacing the `Message` itself. Instead, a `Message` is now bundled with its cache in a big ol' `MessageCall` :

```rust
#[derive(Debug, Clone, PartialEq)]
pub struct MessageCall {
    pub message: Message,
    pub inline_cache: Option<CacheEntry>
}
```

`CacheEntry` is just a pointer to a receiver + a pointer to a method. `Option<...>` is the default Rust option type, meaning "either something or nothing at all": there can be some cache, or there can be none.

Before looking up a method, we check whether there's an entry in the inline cache, and whether it matches that method. If it does, great! We invoke it from the pointer to it that we stored. Otherwise, we look up the method the boring and slow way, and we then store that lookup result in the cache if it's empty.

![alt text](/assets/inline-caching/one-entry.png)

Neat! Mild speedups!

Though we do have got one outlier in the form of this grey little dot: `NBody`. I've no idea why. In my experience, adding to basic data structures in the interpreter like this can prevent some compiler optimizations on some benchmarks. `CacheEntry` is the size of two pointers - one for the class, one for the method. A `Box<CacheEntry>` (Rust heap pointer type) is only the size of one pointer. Let's use that type instead, because why not:

![alt text](/assets/inline-caching/one-entry-boxed.png)

No more outlier! Fixing this bug after not a minute of thinking makes me feel like I'm starting to master arcane arts. I'm going to need to grow a longer beard to become a proper senior developer (i.e. wizard) though[^beard].

It doesn't show very well, but boxing is a very minor slowdown for basically all other benchmarks, though, since boxing actually needs to allocate some memory. Oh well.

One entry is good, but several cache entries *should* be better:

```rust
pub inline_cache: Box<[Option<CacheEntry>; INLINE_CACHE_SIZE]>
```

We'll start with an `INLINE_CACHE_SIZE` of 2.

From now on, results are compared not with a baseline version of the interpreter without inline caching, but with the previous version of the interpreter with a different version of inline caching (and we use micro benchmarks instead: there's more of them, so more data points, so slightly more legible results). So in this case, we're comparing the older 1-entry-cache version with the newer 2-entries-cache version:

![alt text](/assets/inline-caching/two-entries.png)

...ok, that's a rough 1-2% speedup on the AST. Though those results aren't very conclusive: the BC interpreter itself is getting a mild speedup. Our BC interp does rely on the AST generated by the parser (which it just turns to bytecode and then discards), so maybe changing the type of the `inline_cache` allowed the Rust compiler to do some optimizations that made parsing slightly faster, somehow. Either way, not convinced the cache had a major impact.

...but another inline cache entry will totally change things: comparing the 2-entry one with a new one with an `INLINE_CACHE_SIZE` of 3 giiives...

![alt text](/assets/inline-caching/three-entries.png)

...nothing really, just noise. Seven entries?

![alt text](/assets/inline-caching/seven-entries.png)

Slowdown. But I bet ONE THOUSAND entries will do the trick:

![alt text](/assets/inline-caching/1k-entries.png)

Darn. No one could have predicted allocating 1000 entries per callsite would be a slowdown. I think I should try with 2000 (just in case), but for now I think we're done.

<!--
### linked lists

...are a bad idea in Rust, [say smart people](https://rust-unofficial.github.io/too-many-lists/index.html#an-obligatory-public-service-announcement). But I'm also occasionally smart[^smart]

but in our case it sounds smart!

```rust
#[derive(Debug, Clone, PartialEq)]
pub struct CacheEntry {
    class_ptr: usize,
    method_ptr: usize,
    next: Option<Box<CacheEntry>>,
}

#[derive(Debug, Clone, PartialEq)]
pub struct MessageCall {
    pub message: Message,
    pub inline_cache: Option<Box<CacheEntry>>,
}
```

![alt text](/_drafts/linked-list.png)

Only a minor speedup, interestingly.
-->


### done and dusted

OK, an inline cache of 2 is the final choice, then, I guess.

![alt text](/assets/inline-caching/two-entries-base-compare.png)

All our tweaks with caches of varying sizes really didn't improve much. Results also aren't insanely good, which is actually expected: when it was implemented it for the bytecode interpreter, [we got similar numbers](https://github.com/Hirevo/som-rs/pull/13). Speaking of the BC interpreter, our experiments today show that increasing the size of its cache by 1 might be beneficial like in the AST, but I'm not doing that just for a potential 1-2% speedup.

I originally had a section where I implemented inline caching as a linked list instead of an array. That felt like a more natural choice since that would make it a lazy cache: the length of the inline cache is X entries (each one a node) if it's received X receivers, which can be as small or big as it needs. But this wasn't a speedup, probably because our benchmarks never need big caches anyway.

### what have we learned?
- you might have learned how inline caching works? But I didn't learn anything since I already knew that. You've effectively ripped me off.
- monomorphic callsites everywhere! That might be unexpected to some, and makes inline caching not be as beneficial performance-wise as you'd think.
- we've seen Rust doesn't like self-referential, self-modifying tree structures. Which is a pain for me, and advice is more than welcome (tell me on [Twitter](https://twitter.com/OctaveLarose)).
  - my understanding is that managing my own heap/arena of AST nodes may solve it?
  - [Rc::new_cyclic](https://doc.rust-lang.org/stable/std/rc/struct.Rc.html#method.new_cyclic) might be helpful here as well? I'm not entirely sure.
- shoutout to good benchmark running/visualizing software to allow me to do such a thorough comparison of various slightly different versions of my system!
  - I know my supervisor designed Rebench and RebenchDB, but I swear I'm not getting paid to praise them (though I wish I were)

thanks for reading, goodbye

---

[^beard]: A colleague recently shaved his massive beard and I commented on that being essentially him truncating his computer science skills, which he admitted to himself. We all hope his GNU/Linux knowledge comes back alongside his beard