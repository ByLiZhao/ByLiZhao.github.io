---
layout: post
mathjax: true
comments: true
title:  "What happens to padding bytes during initialization"
author: John Z. Li
date:   2020-08-27 19:44:18 +0800
categories: c++ programming
tags: padding-byptes, value-initiliazation, aggregate-initialization
---

Define a struct like below:
```cpp
    struct A{
            char c;
            int i;
    };
```
The actual memory layout is like
```cpp
    struct A{
            char c;
            //char d[3]  three invisible padding bytes
            int i;
    };
```
because of alignment and padding.

Question: what happens to those padding bytes during initialization according to the C++ standard?

1. Case one
```cpp
    A a{}; //value initialization
```
Zero initialization actually is performed while doing value initialization,
all bytes of he initialized object is set to zero, including padding bytes.

2. Case two
```cpp
    A a2 {'\0', 0}; //aggregate initialization
```
While doing aggregate initialization
with `braced-init-list`,
and the number of elements in the `braced-init-list`
is equal with the number of elements of the struct,
the padding bytes are left untouchedx, that is, padding bytes might contain arbitrary value.

3. Case three
```cpp
    A a3{.c='\0'}; //aggregate initialization
```
This is also aggregate initialization.
Since the number of initializer clauses
is less than the number of members of the struct,
remaining members of the struct, as well as all padding bytes, are zero initialized.

*Note: In the C programming language, wording is different, but the same rule applies.*

