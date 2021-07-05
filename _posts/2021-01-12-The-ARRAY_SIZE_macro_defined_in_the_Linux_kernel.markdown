---
layout: post
mathjax: true
comments: true
title:  "The ARRAY_SIZE macro defined in the Linux kernel"
author: John Z. Li
date:   2021-01-12 19:00:18 +0800
categories: c programming
tags: ARRAY_SIZE
---
In searching for a satisfactory solution to calculating array size, I found the
following useful macro in the Linux kernel.

In file "kernel.h", the macro is defined as below:
```c
#define ARRAY_SIZE(arr) (sizeof(arr) / sizeof((arr)[0]) + __must_be_array(arr))
```
The `sizeof(arr) / sizeof((arr)[0])` part is self explanatory,
but what is `__must_be_array()`?
It is a macro defined in "compiler.h". It is defined as
```c
    #define __must_be_array(a)	BUILD_BUG_ON_ZERO(__same_type((a), &(a)[0]))
```
The `BUILD_BUG_ON_ZERO(e)` macro
(BTW, "BUILD_BUG_ON_TRUE" might be a better name )
triggers a compilation error when the condition `e`
evaluated to a non-zero value.
In this case, it triggers a compilation error when the variable being
tested has the same type with that of
a pointer to its starting element.
For example, if `a` is an array defined as
```c
int a[N];
```
It is of type of `int[N]`, and `&a[0]` is of type of `int *`.
Otherwise, if `a` is
defined as
```c
int * a = int a_[N];
```
variable `a` and `&a[0]= &(*(a+0))` are of the same type, that is, `int *`.
So, If one
accidentally uses the `ARRAY_SIZE()` macro to a pointer, a compilation error will be raised.

The C language itself doesn't provide a way to determine
if two types are equivalent (as what `std::is_same` does in C++ ),
nor does it provides a way to get the type of a variable (as `dcltype` in C++).
But type information is known to compilers anyway.
Compilers have to know types of variables to type check a piece of code.
Some compilers like GCC provides language extensions to achieve this.
With GCC, the `__same_type()` macro is defined as in "compiler_types.h"
```c
    #define __same_type(a, b) __builtin_types_compatible_p(typeof(a), typeof(b))
```
The good thing with this macro is that it works even if the array being tested is
a variable length array,
which means its type can only be known at runtime.You can test it with the following code snippet.
```c
    #include<stdio.h>
    #define BUILD_BUG_ON_ZERO(e) ((int)(sizeof(struct { int:(-!!(e)); })))
    #define __same_type(a, b) __builtin_types_compatible_p(typeof(a), typeof(b))
    #define __must_be_array(a)	BUILD_BUG_ON_ZERO(__same_type((a), &(a)[0]))
    #define ARRAY_SIZE(arr) (sizeof(arr) / sizeof((arr)[0]) + __must_be_array(arr))


    int main(void){
            int n = 100;
            int a[n];
            printf("the size of array is %lu", ARRAY_SIZE(a));
    }
```
