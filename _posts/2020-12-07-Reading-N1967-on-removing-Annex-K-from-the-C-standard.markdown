---
layout: post
mathjax: true
comments: true
title:  "Reading N1967 on removing Annex K from the C standard"
author: John Z. Li
date:   2020-12-07 19:00:18 +0800
categories: c programming
tags: memory-safety, c-standard
---
I’ve just finished reading the document named
“Updated Field Experience with Annex K — Bounds Checking Interfaces”
by Carlos O'Donell and Martin Sebor
(the N1967 document as it is referred by the C standard committee).
I found it a very interesting piece of technical writing.
The document points out,
contrary to its alleged technical merits,
the so-called Annex K of C11, also known as  bounds-checking interfaces introduced in C99,
didn’t achieve what it promised.
It turns out, correct error handling is a subtle matter of programming.
Some seemingly useful constructs don’t solve the very problems
they aim to solve, even making the situation even worse.

It is well known that some C standard library functions
are not “safe” in terms of possible buffer overflow.
This leads to potential security vulnerabilities.
Annex K, was proposed as “safer” replacements
to some “unsafe” C standard library functions.
For example, the strcpy with the following signature
```c
    char* strcpy (char* restrict s1, const char* restrict s2);
```
is supposed to replaced by the following alternative one:
```c
    errno_t strcpy_s (char* restrict s1, rsize_t s1max, const char* restrict s2);
```
With `strcpy_s`, the programmer is supposed to explicitly
specify the buffer size of the destination buffer,
so that it can be checked inside the function,
and explicitly check return values of the function
to correctly handle possible errors.
It surely seems a good idea at first glance.
(Visutal Studio has been promoting its implementation of a subset of Annex K ever since.)
And I used to take it for granted that those functions
ended with a suffix “_s” must be safer.
However, field experience showed a quite counter-intuitive story.

According to the document, besides many things,
like
1. inefficiencies introduced by repeated bound checking
when one check should suffice,
2. more verbose code than that with older standard function calls,
3. being intrusive while refactoring legacy code,
4. awkward error handling in a multi-threaded context, etc. ,
Below are what I believe lies at the core of the problem.

First, the kind of errors that are supposed to be prevented
by using Annex K functions are logical errors.
This means, no programmer in their right mind would introduce
such an error intentionally.
A consequence of this is that when testing code,
he normally does not know how to trigger an error case
where the runtime constraints violation will lead to execution of error handling code.
It is only natural that If you don’t know you have made a mistake,
you can’t replicate that mistake.
To make things worse, the error handling code may also use
these Annex K functions that themselves need to be tested.
But there is a second order effect here,
that one is trying to **trigger an error in error handling code to test the
error handling code that is supposed to handle the kind of error in the first place**.
On the other hand, if a programmer has found an error of the type,
he could just fix the error.
After doing so, both the error handling code and the error
prevention functionality of the Annex K functions just become dead code.
They are there for errors that you know will not happen.

Secondly, A runtime constraint violation is very different
from input data errors.
The former, is a logical error,
a fatal one that the program can not recover from.
In face of such errors, the sensible thing to do
is aborting the program as soon as possible.
But input data errors are expected errors.
It is unreasonable that the program
just crashes in face of an incorrect user input.
If this is the case, the behavior itself will be
a source of security vulnerabilities.
The correct thing to do is always validate user input.
And the validation logic is often way more complicated
than what Annex K has provided.
And it surely can be done in an orthogonal fashion
with respect to the standard library functions.

Thirdly, there are better non-intrusive tools now.
Notable ones are Clang Address Sanitizer,
which can detect most cases of memory corruption with some runtime overhead.
And there is Intel Pointer Checker which has minimum to none
(on Intel CPUs with Memory Protection Extension) runtime cost,
and it can catch all incorrect array or pointer usage.
The latter approach should be the correct way to go with Intel CPUs.
While with non-Intel CPUs,
the former can be used along with some static analysis tools,
many of which are integrated to compilers nowadays.

