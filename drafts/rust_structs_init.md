---
layout: post
title: "Initializing Rust structs makes sense (but not to my brain)"
author:
- Octave Larose
---

This is part of a series of blog posts relating my experience pushing the performance of programming language interpreters written in Rust. But this doesn't matter here: I've nothing new and interesting to talk about, I've just been fixing bugs. Instead I just want to share some gripes I have with the Rust language.

## why is this not valid

I was just minding my own business doing some refactoring in my parser when it broke. Which made me think "darn, I must not be abiding by the ownership rules in some way, I might need to refactor my entire approach". See here:

```rust
let ctxt = AstGenCtxtData {
    [...] // skipping a bunch of other fields
    current_scope: outer.borrow().current_scope + 1,
    universe: mem::take(&mut outer.borrow_mut().universe),
};
```

The issue that arises is that the `universe` field initialization gives us a `already borrowed: BorrowMutError`. Originally, I thought "this code looks fine it should be fine to me, so `outer` must already be mutably borrowed elsewhere in the code". This is NOT the case: the issue comes only from the lines of code shown here.

I then thought `mem::take` must be causing issues somehow, which was not the case. I also thought I might have forgotten to increase the reference count of outer for one of these instructions (throuhg `Rc::clone(&outer)` does), which was also not the case.

I know that struct fields get initialized in order - [that's guaranteed by the compiler](https://stackoverflow.com/a/62001313/10489787). In this case, `current_scope` gets initialized via `outer.borrow().current_scope + 1`, then `universe` via `mem::take(&mut outer.borrow_mut().universe)`. So it being sequential theoretically means that after being done initializing `current_scope`, we're done using/borrowing `outer` and can borrow it again for `universe`. Yet this isn't the case: it considers `outer` to still be borrowed, which is not intuitive to me - I'm clearly done with `current_scope`!

Here's the functioning code:
```rust
let taken_universe = mem::take(&mut outer.borrow_mut().universe);
let ctxt = AstGenCtxtData {
    [...] // skipping a bunch of other fields
    current_scope: outer.borrow().current_scope + 1,
    universe: taken_universe,
};
```

...which compiles fine. We get `taken_universe` ahead of time, which means we're free to borrow `outer` again when initializing a struct. This all means that the borrow checker was complaining because we borrowed the same value to initialize two different fields.

Which makes complete sense since really, the code above is the same as `let ctxt = AstGenCtxtData { [...], current_scope: [...], universe: [...] };` - it's all one line, and Rust ownership works on a line by line basis. So my code not working _MAKES SENSE_, yet it took me a while to understand why it did... The line by line basis idea messes with my head when initializing structs over several lines, even though it is clearly still one line (with one semicolon at the end). The concept that the borrow checker works by treating sequential operations as discrete made me think that sequential operations when initializing a struct would

This is a silly mistake on my part, looking back at it. But I understand why I made it: my code is clearly safe.

[todo find ref for ownership being line by line - steal their term. "expression"?]

I hope this might be of help to others who encounter this same error. Or that more knowledgeable people put forth arguments that I'm missing explaining why this completely makes sense and why the Rust spec shouldn't/couldn't possibly be modified to account for this.
