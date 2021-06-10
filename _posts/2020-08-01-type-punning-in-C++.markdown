---
layout: post
mathjax: true
comments: true
title:  "Type punning in C++"
author: John Z. Li
date:   2020-08-01 19:44:18 +0800
categories: c++ programming
tags: type-punning
---
Type punning is how a programmer converts an object of one type into another
type in a way that cannot be done by normal type cast.
This means, the type system must be somehow circumvented.
One may think of reinterpret_cast,
but it is usually a bad idea due to it is way too easy to violate the
strict alias rule, which leads to undefined behavior.
For example, the following code snippet triggers undefined behavior for that reason.
```cpp
    float f;
    int d = 1234567;
    std::cout << *reinterpret_cast<float *>(&d) <<std::endl;
```
The C language actually provides a mechanic to do type punning by unions,
which is, forbidden by C++.
The following code is legitimate C code,
but leads to undefined behavior in C++ according to the standard.
```cpp
    union {
     int d;
     float f;
     } u;
     u.d = 1234567;
     printf("%f\n", u.f);	//OK in C99,
     std::cout << u.f << std::endl;  undefined behavior in C++
```
The modern replacement of unions in C++ is std::variant.
It has stricter rules on this.
Trying to do type pruning via std::variant will throw an `std::bad_variant_access`
exception. The following is an example:
```cpp
    std::variant<int, float> v = 12;
    float x = std::get<1>(v); //throw std::bad_variant_access
```
Despite what the language standard might say,
GCC actually implemented an extension to make it (using unions to do type punning)
legal also in C++. Last time I checked, MSVC allows it too.
Anyway, this approach is not standard compliant.
If you want to write portable code, you can always resort to the good old memcpy, as below:
```cppp
     int d = 1234567;
     float y;
     memcpy(&y, &d, sizeof(d));
```
This seems inefficient at first glance. But if you only use the copied-to
object as an read-only object, often times, compilers are smart enough to
eliminate actual memory copy, making it as efficient as it can be.

In case you are not really fond of memcpy, C++20 introduced `bit_cast` for just
that purpose. Using `bit_cast` makes code clearer and shorter by explicitly
stating programmers’ intention, as below:
```cpp
     int d = 1234567;
     std::cout<< std::bit_cast<float>(d) <<std::endl;
```
