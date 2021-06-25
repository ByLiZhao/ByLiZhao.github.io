---
layout: post
mathjax: true
comments: true
title:  "Empty classes and zero length arrays in C++: Part 2"
author: John Z. Li
date:   2020-09-15 19:00:18 +0800
categories: c++ programming
tags: empty-classes, zero-length-arrays
---
In the [previous post](https://bylizhao.github.io/c++/programming/2020/09/07/Empty-classes-and-zero-length-arrays-in-C++-Part-1.html) we talked about empty classes and zero-length arrays in C++. The language standard mandates that an empty class has a non-zero size. So the following class B has a size larger than size of int.
```cpp
    class A{};
    Class B{
        A a;
        int i;
    }
    assert(sizeof(B) > sizeof(int)); //always true
```
The increase of the size of a class by introducing an empty class as a data member is not only counter-intuitive, but also a waste of memory space.
Because of alignment and padding, an empty data member may introduce
more than 1 byte of extra memory consumption, even if the size of an empty class is only 1 byte.
One way to avoid this is to take advantage of the so-called
Empty Base Optimization,  That is, when an empty class is inherited,
compilers are required to optimize memory they occupy away. For example, the following class `C` is as large as an `int`.
```cpp
    class C; public A{
        int i;
    }
    assert(sizeof(C) = sizeof(int)); // a standard compliant compiler must not fail this assert.
```
**Notice that for Empty Base Optimization to kick in, the empty base class must appear as the first base class in the base class list, and the derived class should not itself be
an empty class.**

 But problems arise when one tries to inherit from multiple empty classes. In this
 case, does Empty Base Optimization optimize away both empty base classes?
This is actually implementation defined.
For example, for the following code, where a class inherits from two different empty classes.
Whether
 `get_ap()` and `get_bp()` return pointers with the same memory address
is implementation defined.
If the two functions return the same memory address, because base class `A` is required
by the language standard to be optimized away, it means that the compiler also optimize
the second empty base class away. This is the case with GCC.
```cpp
    #include <iostream>
    struct A {};
    struct B {};
    struct C : A, B {
      int i;
      A *get_ap() { return this; }
      B *get_bp() { return this; }
    };

    int main() {
      C c;
      assert(c.get_ap() == c.get_bp());// implementation defined.
      std::cout << c.get_ap() << ", " << c.get_bp() << std::endl;
      return 0;
    }
```

What if a class inherits from the same class twice? In this case, the language standard requires that pointers to the two subobjects  are of different addresses. The precise wording is (Chap. 10, 5.10)

> A base class subobject may be of zero size (Clause 9); however, two subobjects that have the same class type and that belong to the same most derived object must not be allocated at the same address .

When inheriting in class template ,
if one does not pay attentions, unexpected things might happen.
```cpp
    template <class Empty1, class Empty2, class X>
    struct Class_with_empty_base : Empty1, Empty2
    {
       int i;
    };
```
The size of derived class `Class_with_empty_base` will be different for the case where
it is instantiated with two different empty classes, and the cases where it is instantiated using the same empty twice. In other words, the following might be true:
```cpp
struct A {};
struct B {};
using AA = struct Class_with_empty_base<A, A>
using AB = struct Class_with_empty_base<A, B>
assert (sizeof(AA) != sizeof(AB) ); // possibly true;
```
