---
layout: post
title: "AST interpreter inline caching (in Rust) (gone wrong) (TODO: get a better title)"
author:
- Octave Larose
---

This is part of a series of blog posts relating my experience pushing the performance of programming language interpreters written in Rust. For added context, read the start of [last week's blog post]({% post_url 2024-05-29-to-do-inlining %}).

### inline caching?
Inline caching is an extremely widespread interpreter optimization, that's highly beneficial for Smalltalk-based systems that do a ton of method calls. Basically, say you have this code:

```
    someclass := getSomeClass: "lol".
    x :=
```