---
layout: post
mathjax: true
comments: true
title:  "What is wrong with C++'s string class"
author: John Z. Li
date:   2020-12-24 19:00:18 +0800
categories: c++ programming
tags: string, small-string-optimization
---
By the title, I do not intent to nitpick on C++’s string class
 by complaining things like
- strings in C++ are not a built-in type, many roll it own string class,
- it has too few member methods,
- it should specifies an encoding, etc..
While these complaints are valid points more or less,
but, hey, it is C++, programmers’ convenience is not on its
priority list from the beginning.
What I am going to talk about in this post are performance related issues.
After all, C++ takes “zero overhead abstraction” very seriously
(as its first design principle).
I have three points to make:

First, Strings initialized from string literals should not
incur runtime cost. With C, the following code runs cheaply as it can be:
```c
    const char[] = "some string literal";
```
With C++, the following code does not
only make an extra copy of the string literal,
but also leads to a heap allocation if the string literal is too big
to be optimized away by the so called Small String Optimization (SSO):
```cpp
    const std::string s = "some string literal";
```
This means that C++ strings are not a replacement for C-stype strings.
A programmer either choose C strings for this kind of stuff,
at the cost of losing some functionalities of C++’s string class provides,
or he goes with C++ strings knowing that they are not really of zero overhead.
C++17 introduced `string_view` that might mitigate this problem,
but also introduced some new issues.

Secondly, there should be a way to express the idea that a string is a constant.
And I am not saying that you should mark your strings with `const` whenever possible.
The following code does not guarantee the string will not be altered during its lifetime:
```cpp
    const std::string s = "some string literal that does not fit SSO";
```
The reason for this is that C++ does not propagate constness
of an object along the chain of pointers, that is, with the following code
```cpp
    struct A {char* p;};
    const A a;
```
The language only guarantee that the value of `p` won’t be
changed during its lifetime (provided constness is not cast away),
but the content of the on-heap memory chunk pointed by `p` might be
changed (you know how).
This prohibits a whole load of optimizations that might otherwise kick in.
This may not be a big issue for other types,
but strings are used so frequently in programming,
there must be a price tag attached to it.
I don’t think there is easy fix for this without breaking backward compatibility.

Thirdly, strings should be promoted to stack strings when escape analysis show that the string is only short-lived.
Though I don’t have concrete numbers,
my estimation is that most of strings used in a typical program are short-lived
small strings.
This is why Small String Optimization (SSO) can help with performance.
But SSO is too restrictive, strings that are larger than 24 (with GCC),
or 32 (with Clang) chars can not be optimized with it.
Modern OSes have large thread stacks.
A Linux thread on 64-bit platforms have 8MB stack memory.
This means, if a program is constantly exchanging small strings
varying from several bytes long to several hundreds bytes long with the external world,
it can have a large performance boost by eliminating heap allocation of strings.
**Many short-lived strings should not  incur heap allocation at all**.
If Escape Analysis alone is not enough, programmers should be given the option to
mark a string as short-lived and small, so that the compiler can make the
string reside on the stack.
If this is the case, SSO will not be necessary,
because SSO now becomes a special case of the stack string optimization.
To make this optimization possible, if I am not mistaken,
requires C++ makes an exception for stack strings to temporally disable stack unwinding,
so that when a function returns
a stack string created inside the function is still accessible to the calling function.
The string is deallocated via stack unwinding only when its lifetime ends.

