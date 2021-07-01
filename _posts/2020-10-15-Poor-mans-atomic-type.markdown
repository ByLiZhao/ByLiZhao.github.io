---
layout: post
mathjax: true
comments: true
title:  "Poor man's atomic type"
author: John Z. Li
date:   2020-10-03 19:00:18 +0800
categories: c programming
tags: memory-order, atomic
---
Assuming that we are working on a 32-bit machine,
is it safe to use properly aligned 32-bit variables as poor man’s atomic variables
in a multi-threaded context?
The answer to this question is of course NO for obvious reasons.
We all know the tricky things about memory order.

Now let us refine the original question as

If we only need to use a properly aligned 32-bit variable (say a int)
as a 32-bit channel-buffer between two threads,
that means one thread writing to it and another thread reading from it,
and we know in advance that our program will be run on a single-core CPU,
is it safe to write code like below?
```c
    If(sahred_int){//in the reading thread
      do_something(shared_int);
    }
```
The answer is still NO. (To my surprise before I read [link 1](https://lwn.net/Articles/793253/),
[link 2](https://lwn.net/Articles/508991/), and [link 3](https://github.com/google/ktsan/wiki/READ_ONCE-and-WRITE_ONCE)).

One thing could happen is the so-called “load tearing”,
which means the CPU might split loading of `shared_int` into two 16-bit load.
If the value of `shared_int` gets changed between these two loads,
the reading thread might get garbage value of `shared_int`. For example, if the writing thread performs the following:
```c
    //in the writing thread
    shared_int = 0;
    shared_int = 0xFFFFFFFF;
```
A possible execution flow is (in pseudo code):
```
    shared_int = 0; //by the writing thread;

    the reading thread reads first two LSBs of shared_int

    shared_int = 0xFFFFFFFF;

    the reading thead reads last two MSBs of shared_int;
```
resulting a garbage value of shared_int being `0XFFFF0000`, which is never intended.

Correspondingly, there is also “store tearing”, that is,
the action of a single 32-bit store is split into two separate 16-bit store instructions.

The C language standard does not guarantee that accessing a word is atomic.
Because of this, many kinds of optimization might kick in while compiling source code,
resulting in subtle and hard-to-debug errors.

If full-fledged atomic types is not desired for some reasons, the `ACCESS_ONCE()` macro (introduced in Linux kernel in `<linux/compiler.h>`),
which is defined as
```c
    #define ACCESS_ONCE(x) (*(volatile typeof(x) *)&(x))
```
combined with the cache coherence and memory coherence protocol of x86/amd64 CPUs
(this guarantees that any store to a shared variable will be visible for other threads),
is sufficient to many use cases as a lightweight replacement for atomic types.
