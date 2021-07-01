---
layout: post
mathjax: true
comments: true
title:  "Do not redefine type char"
author: John Z. Li
date:   2020-11-01 19:00:18 +0800
categories: c programming
tags: C, C++
---
The C (and C++) language standard doesn’t specify whether the `char` type
is signed or unsigned. This annoys some programmers,
I’ve seen suggestions in stackoverlow that you explicitly define
`char` being signed or unsigned using `#define` directive.

The problem with this is that `char`, `unsigned char`, and `signed char`
by the language standard are three distinctive types,
whether `char` is implemented as signed char or unsigned char
is implementation defined, that is, a problem of representation.
At the level of the type system, internal representation  of a type is
independent from semantics of that type.
So, it is never a good idea to redefine fundamental types of the language.
If you do it, you will hardwire a specific representation of a type into
the type system.
This might cause weird problems while interacting with other code
which have a different idea about signed-ness of the char type.

One should never write code that suppose a certain signed-ness of the `char` type.
Doing so is breaching type constraints, that is, you are assuming things that the type system has
never promised.
If you mean `char`, just use `char`. If, for example,
a third-party library exposes its interface using signed or unsinged char,
use the corresponding type accordingly.
Though I believe it is a mistake to expose interface via signed or unsigned chars.
If, on the other hand, everyone just sticks to type `char`,
given a certain platform and toolchain, everything just works fine.
In case of cross compiling, gcc provides the compiler option `-fsigned-char` or `-funsigned-char`,
which can force the underlying representation of char being signed or unsigned respectively.

Another thing to notice is that `char` doesn’t have to be 8-bit wide.
In C (and C++), char is synchronous with byte,
the width of which is platform dependent.
So, don’t use `signed char` as a replacement of an 8-bit signed integer type, use `int8_t`
instead, or `unsigned char` as a replacement of an 8-bit unsigned integer type, use `uint8_t` instead.
Likewise, don’t assume `char8_t`, newly introduced in C++20 is synchronous with `unsigned char`.
Although the language standard says that the underlying representation of the type is the same with `unsinged char`,
but it doesn’t mean they are the same type. Actually,
a standard compliant implementation should evaluate `std::is_same_v<unsigned char, char8_t>` to be false.
It is good to be honest with the type system.

