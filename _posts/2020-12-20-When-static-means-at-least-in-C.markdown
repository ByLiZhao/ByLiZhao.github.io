---
layout: post
mathjax: true
comments: true
title:  "When static means at least in C"
author: John Z. Li
date:   2020-12-20 19:00:18 +0800
categories: c programming
tags: static
---
When arrays are used as function parameters in C,
they can be qualified with the keyword `static`.
It means that the array has a specified minimum length, for example,
```c
    void fun(char ca[static 100]);
```
defines a function, that takes a char array as its parameter,
and the caller will guarantee that when the function is called,
the array has at least 100 elements.
This also introduces a optimization opportunity,
because now inside the function,
the pointer decayed from array type is guaranteed not to be a `NULL` pointer.
As a result,
null checking is not necessary, and the compiler is free to remove such code.
```c
    void fun(char ca[static 100]){
    if(ca){// this null checking code can be removed by the compiler.
    }
    }
```
A useful idiom is using
```c
    void fun( struct a[static 1]);
```
instead of
```c
    void fun(struct a *);
```
to indicate that you don’t intend to pass a null pointer to the function.

