---
layout: post
mathjax: true
comments: true
title:  "C++ has the Stop-The-World GC problem too"
author: John Z. Li
date:   2020-10-01 19:00:18 +0800
categories: c++ programming
tags: GC, Stop-the-world
---
At first glance, it seems odd to talk about GC in the context of C++,
as C++ is famous for its lack of a Garbage Collector (GC).
But if we use the term GC in a broader sense, referring to some functionality of
the programming language that can automatically free unused memory blocks,
smart pointers introduced in C++11 is a kind of GC.
Smart pointers in C++ thus can be said to be a ref-counted GC.
(unique pointers semantically can be viewed as shared pointers
but their reference counting never exceeds one.)
From the perspective of symmetry,
ref-counting GC can be viewed as the dual algorithm of mark-and-sweep GC algorithms,
that is, while mark-and-sweep GC algorithms are singling out currently active objects,
ref-counting based GC  algorithms
singles out inactive objects.
At any given moment of program execution,
the set of has-become-inactive objects are the
complement of all currently active objects.

It is well known that GCs have the Stop-The-World problem.
This makes languages such as Java can’t be easily used for hard real-time problems.
A common mis-belief about C++ is that C++ just does not have this problem.
However, as a characteristic shared by all GCs, C++ does have a Stop-The-World GC problem with smart pointers.
This can happen when destructor of a smart pointer triggers the destructor of another smart pointer,
and executing the destructor of the second smart pointer in turn triggers another smart pointer’s destructor to be executed, and so on,
until a very deep recursion of destructor calling being formed.
Suppose that each memory deallocation consumes 1.5us averagely,
and there are 2,000 thousand objects are to be deallocated consecutively,
there will be a 3ms Stop-The-World GC pause.
3ms for 2,000 objects does not seem a lot, but see the below performance statistics of Java’s newest ZGC:

> ZGC

> avg: 1.091ms (+/-0.215ms)

> 95th percentile: 1.380ms

> 99th percentile: 1.512ms

> 99.9th percentile: 1.663ms

> 99.99th percentile: 1.681ms

> max: 1.681ms

With C++, programmers get to determine when deallocation happens.
This does not mean it is free lunch.
A very deep recursive calling of destructors is not only expensive,
but also has the potential to overflow the stack.

