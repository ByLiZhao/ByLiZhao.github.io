---
layout: post
mathjax: true
comments: true
title:  "Empty classes and zero length arrays in C++: Part 1"
author: John Z. Li
date:   2020-09-07 19:00:18 +0800
categories: c++ programming
tags: empty-classes, zero-length-arrays
---
The C++ language standard guarantees that objects of an empty class has
non-zero size, so the following code snippet will print out a positive number:
```cpp
    strcut A{};
    A a;
    std::cout<<  sizeof a << std::endl;
```

What the size would be is implementation defined.
In both GCC and MSVC, as I last checked, they will treat A as if it contains
an invisible char as below.
```cpp
    struct A {
      \\char a; \\invisible, but occupy 1 byte.
    }
```
This means that the following code snippet will produce 10:
```cpp
    A a_array[10];
    std::cout << sizeof a_array << std::endl;
```
and `a_array[0]`, `a_array[1]`, …, `a_array[9]` are all legitimate access.

It is worth noticing that it is undefined behavior to declare or define an
array of zero length. The following code is illegal:
```cpp
    struct B{
      char b[0];
    };
```
Here is the precise words about the length of an array in
the standard (ISO/IEC 14882:2003 8.3.4/1:), (emphasis is mine)

> If the constant-expression (5.19) is present, it shall be an integral constant expression
> and its value **shall be greater than zero**.

Despite that, to “new” an array of zero length is actually legal in C++,
so the following code is legitimate in C++,
although trying to de-reference that pointer leads to undefined behavior.
```
    char * p = new char[0]; // this is legal
```
The precise words in the standard is (ISO/IEC 14882:2003 5.3.4/6), (emphasis is mine):

>The expression in a direct-new-declarator shall have integral or enumeration type (3.9.1) with a **non-negative** value.

This can be regarded as a flaw of the language standard.
After all, `struct B` is just an empty class.
I found that MSVC is quite consistent regarding this matter in treating both `struct A` and `struct B`
as empty classes and give identical behavior.
While GCC behave differently. With GCC applying `sizeof` to `struct B` always gets 0.

