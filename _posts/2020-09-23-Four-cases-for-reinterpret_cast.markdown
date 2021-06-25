---
layout: post
mathjax: true
comments: true
title:  "Four cases for reinterpret_cast"
author: John Z. Li
date:   2020-09-23 19:00:18 +0800
categories: c++ programming
tags: type_cast, reinterpret_cast
---
A previous blog post of mine [Type punning in C++](https://bylizhao.github.io/c++/programming/2020/08/01/type-punning-in-C++.html)
mentioned that using `reinterpret_cast` to do type punning can easily leads to code that
violates strict alias rule, thus undefined behavior, as the following code does
```cpp
    int d = 1234567;
    //don’t do this, undefined behavior
    std::cout << *reinterpret_cast<float *>(&d) <<std::endl;
```
There are quite some posts on the Internet about `reinterpret_cast`
that actually involves examples violating strict alias rule unintentionally.
But when is it safe to use reinterpret_cast?
I will give 4 cases where `reinterpret_cast` can be safely used in this post.
The list is not exhaustive, only including most common cases.

First of all, don’t use C-style cast in C++ code. What C-style cast does is
trying any of `static_cast`, `const_cast`, `reinterpret_cast` or
combinations of the three to get its job done.
It is kind of hard to reason what happens with a C-style cast while
reading C++ code.
When `static_cast` or `dynamic_cast` expresses intention of the programmer better,
one should choose one of them specifically. With that in mind, here we go.

**Case 1**: cast between a pointer to signed integer types (char, short, int, etc.,)
and a pointer to its corresponding unsigned pointer like below
```cpp
    char  a[] = "a char string";
    unsigned char* up = reinterpret_cast<unsigned char *>(a);
```
As a side note, it is also OK to convert a pointer to `std::byte` (introduced in C++17)
from a pointer to (signed or unsigned) char.
This is useful when working with third party interfaces that takes `unsigned char*` but you have `char *`, or `std::byte`.

**Case 2**: cast an object pointer to a pointer to raw bytes. For example, a possible implementation of a string class is as below
```cpp
    class string{
    char * data;
    size_t capacity;
    size_t size;
    };
```
The most significant bit of the most significant byte of the member variable `size`,
when set to 1, indicates that the current string object is a small string,
which is stored in the remaining bytes of the object (including an appending ‘\0’ ).
(Remaining 7 bits of that byte is used to store the size information of a small string.)
Suppose it is a little-endian 64-bit machine ,
a strings with its size being smaller or equal to 22 (3 times 8 minus 2)
does not incur heap allocation.
To check whether a string instance is a small string, one needs to perform something like
```cpp
    //string s;
    bool is_small_string = reinterpret_cast<char*>(&s)[24] & 0x80 ? true : false ;
```
to check and let `front()` refers to `reinterpret_cast<char*>(s)[1]` if it is.

**Case 3**: cast to escape from nested structs, arrays or unions. For examples, with
```cpp
    struct A{int a;};
    struct B{
        A a;
        int b;
    };
    struct C{
        B b;
        int c;
    };
    C c{{{1}, 2}, 3};
    int (& ints)[3] = reinterpret_cast<int(&)[3]>(c);
	// to treat c as an array of three ints
```
object `c`, after compilation, boils down to three contiguous ints.
So we can use `reinterpret_cast` to treat `c` as an array of int of length 3.
This is useful when one works with complex numbers.
No matter how complex number is implemented,
the standard guarantees that for a complex number z, the following is true:
```cpp
// T is the underlying type
    reinterpret_cast<T(&)[2]>(z)[0] == z.real();
    reinterpret_cast<T(&)[2]>(z)[1] == z.imag();
```
and for an array of complex numbers, p, there is
```cpp
    reinterpret_cast<T*>(p)[2*i] == p[i].real();
    reinterpret_cast<T*>(p)[2*i + 1] == p[i].imag();
```

**Case 4**: cast to restore or construct nested structs, arrays or unions.
This is essentially the reverse procedure of Case 3.
It is especially useful when working with matrices (two-dimensional arrays)
and tensors (arrays with dimension equal to or larger than three). For example,
```cppp
    loat fa[16] = {};
    //fb is a reference to a 4-by-4 matrix
    float (& fb)[4][4] = reinterpret_cast<float(&)[4][4]>(fa);
    //fc is a reference to a 4-by-2-by-2 tensor.
    float (& fc)[4][2][2] = reinterpret_cast<float(&)[4][2][2]>(fa);
```
When dealing with matrices or tensors, casting around like this could be handy.
