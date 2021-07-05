---
layout: post
mathjax: true
comments: true
title:  "A taxonomy of coroutines"
author: John Z. Li
date:   2021-01-26 19:00:18 +0800
categories:  programming
tags: coroutine
---
Depending on how coroutines are designed in a programming language,
we can classify coroutines into 8 (2 times 2 times 2 times 2) different types.

1. Symmetric / Asymmetric coroutines

With respect to control flow,
a symmetric coroutine is able to transfer control
freely to its peers.
The current coroutine determines which coroutine to be invoked next.
All symmetric coroutines are equal in the sense that
no coroutine has control over other coroutines
other than transferring control to it.

With respect to control flow, when an asymmetric coroutine is suspended,
the control is always returned to the invoker of it.
In a sense, the invoker "owns" the (asymmetric) coroutine it invokes.
The relationship of them is analogous with that of the caller
and the callee of a subroutine.

2. First-class / constrained coroutines.

If coroutines in a programming language are always supposed to
use with/within some specific language constructs.
Iterators and generators in many programming languages
are examples of constrained coroutines.
Usually, they are only supposed to loop over or traverse a
collection of data objects.
They are used to abstract away concrete representations of
these collection and provide programmers with a
unified interface to traverse collections without
bothering their underlying implementations.

OTOH, If coroutines are first-class citizens of a programming language.
It means, the programmer can use it anywhere as he wants,
This often means, the programmer can compose coroutines freely
with other language features to explore the full power of expressivity of coroutines.

3. Stackful /stackless coroutines

A stackful coroutine can be suspended and resumed within nested function calls.
While a stackless coroutine can only suspend its execution
when the current active stack frame is the one where it has been invoked.
Though the term stackful, or stackless is somehow misleading.
For one thing, it conflates conceptual properties with implementation techniques.
Typically, a stackful coroutine has its own runtime stack,
make subsequent function callings within it happen on that stack,
while a stackless coroutine usually uses the same stack that the invoker
residing on (often OS thread stacks),
but this is just one way to implement them.
For another, it confuse semantics with optimization,
most of the times, the calling chain within a coroutine
is of limited length that can be determined at compile time,
thus the stack associated with a stackful coroutine
might be optimized away.
Maybe better names are colorless coroutines for stackful coroutines,
and colorful coroutines for stackless coroutines ([What Color is Your Function?](https://journal.stuffwithstuff.com/2015/02/01/what-color-is-your-function/)
).

4. Schedule-able / non-schedule-able coroutines

Schedule-able coroutines are often called fibers,
which is also known as "threads in user space" except that fiber only
working with cooperative context switching while kernel threads can be preemptively scheduled.
Like kernel threads, the lifespan of a fiber can be longer than that of its invoker.
When a fiber yields (or more precisely, blocked),
it passes control to the scheduler, which determines which fiber is to be executed next.

When a non-schedule-able coroutine yields,
control is transferred to the invoker
(in the case of asymmetrical coroutines)
or to the designated coroutine (in the case of symmetric coroutines).
Advanced scheduler may also move fibers between different kernel threads,
making it easy to utilize multi-core CPUs.

