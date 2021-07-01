---
layout: post
mathjax: true
comments: true
title:  "NULL is not what I thought it were"
author: John Z. Li
date:   2020-11-16 19:00:18 +0800
categories: c programming
tags: NULL
---
I used to believe that a C compiler would
guarantee type correctness of `NULL`,
so that whenever `NULL` is used,
I can just assume it is of the type `void *`.
This turns out to be false.

The C language standard defines NULL as below:

>    [6.3.2.3] An integer constant expression with the value 0,
>    or such an expression cast to type void *, is called a null pointer constant.

So, `NULL` can be any “integer constant expression” that evaluates to 0.
But what is an “integer constant expression”? By the language standard, it is defined as:

>    [6.6.6] An integer constant expression shall have integer type and shall
>    only have operands that are integer constants,
>    enumeration constants, character constants,
>    sizeof expressions whose results are integer constants,
>    _Alignof expressions, and floating constants
>    that are the immediate operands of casts.
>    Cast operators in an integer constant expression
>    shall only convert arithmetic types to integer types,
>    except as part of an operand to the sizeof or _Alignof operator.

That is, in short, any expression that is of integer type and can be,
in principle, evaluated into a constant during the compilation phase.
It means `false`, `‘\0’`, `(short)0`, `(int)0.0f`
are all valid candidates for NULL.
I have no idea why the language standard is
so permissive in this regard. Maybe, `nullprt_t`
of C++ should be ported into C.

