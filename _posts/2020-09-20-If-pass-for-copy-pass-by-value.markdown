---
layout: post
mathjax: true
comments: true
title:  "If pass for copy, pass by value"
author: John Z. Li
date:   2020-09-07 19:00:18 +0800
categories: c++ programming
tags: copy-elision, move-semantics
---
This post is inspired by the seminal blog post
“Want Speed? Pass by Value” by Dave Abrahams.

An object in C++ is a chuck of memory in RAM.
Theoretically, there are three things that one might want to do with it, that is,
1. to read from it., (to copy bytes from RAM to CPU caches or registers);
2. to write to it, (to copy from CPU caches or registers to overwrite some bytes of the object);
3. to clone a new copy from it, (to allocate new memory in RAM, and copy from the old object to the new object).

With introduction of move semantics,
things get a little bit more  complex in C++.
One way to reasoning move semantics
is to regard it as a form of economic copy
that “breaks” the copied-from object during the process.
With this in mind, there are four things that one might want to do with an object, that is

1. to read from it;
2. to write to it;
3. to clone a new copy from it, leaving the original intact;
4. to clone a new copy from it, leaving the original possibly broken.

When an object is passed into a function, a question can be asked:
> What purpose does the parameter passing is supposed to serve?
This morale of this question is not about performance or optimization,
but about programmers’ intention and semantics.
While there is no language features that help programmers
to explicitly make distinctions between the above mentioned four cases,
there is one thing that the language does provide, that is:
> Passing by value means to copy (case 3 or 4).
So whenever one sees some code like
```cpp
class_name a;

fun(a);
fun(g()); //g() returns an object of type class_name
```
he knows that semantically, a copy  is required.

Now if he wants to copy the passed-in parameter inside the function,
the so-called Copy Elision (CE) can kick in.
The idea behind CE is simple. It goes along the line like this:
> if an object A is supposed to be copied to an object B,
> and A is no longer needed after the copy,
> we can let A just “become” B and omit the copy;
> Similarly, if an object A is supposed to be copied to an object B,
> and B is in turn copied to an object C, we can copy A directly to C,
> and let B an alias to C omitting the second copy.

Clearly, compilers can only optimize unnecessary copies away,
when it can reason  that such a pattern will appear.
This leads to a rule of thumb: **If an object will be copied inside a function,
pass it by value. CE will eliminate redundant copies.**

Since pass by value covers case 3 and case 4,
we can save pass by constant reference for something that
bears more semantic meaning,
that the parameter is read-only (case 1). So that whenever ones sees some code like this
```cpp
    fun(const class_name & a, …);
```
he know that `fun()` or functions called by `fun()` won’t make copies of `a`,
nor write to a (no `const_cast` or other black magic will be performed).
**Notice that this is not guaranteed by the language, we just make it a convention to
better express programmers' intentions.**

As to Case 2, we could let passing by reference bears the extra semantic meaning
that one would not explicitly take its address inside the function, for example, when one sees code like
```cpp
    fun(class_name & a, ...)
```
we know that there is no code like below in `fun` or functions  called by `fun`,
```cpp
    class_name * p = &a;
    p++;
```
And the only way to access to `a` is through `a` or an
lvalue reference binding to `a`,
something like the `__restrict` keyword in C implies, but with stricter restriction.
Then the only way to modify `a` inside fun() is though `a` (or an lvalue reference coming from a)
via public fields and public methods of `a`.
If we need more complex semantics, we should resort
to pointers, all wrap the semantics in a class.

It is a pity there is no way one can enforce these rules.
I would stick to them as part of personal coding style.
It would be helpful for code readability and program correctness.
