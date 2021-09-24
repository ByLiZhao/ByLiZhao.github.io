---
layout: post
mathjax: true
comments: true
title:  "A cost benefit analysis of the 'const' keyword in C++"
author: John Z. Li
date:   2021-09-24 19:00:18 +0800
categories: c++
tags: const, constness, const_cast, immutability
---
While reading real-world C++ code in some open-source projects, I noticed the following
minor issues appearing again and again:

## A `const`object in C++ is only head-const
A non-trivial C++ class usually contains pointers as member variables. For example,
A `std::string` object contains a `char *` pointer which points to a buffer that holds
the content of the string. When applying the `const` keyword to a C++ object, that
object is head-const only. For a pointer, if the pointer itself is `const` but not the
chunk of memory pointed to by the pointer, we say that this pointer is head-const.
In contrast, if the chuck of memory pointed to is `const` but not the pointer itself,
we say that this pointer is tail `const`. When working directly with pointer, C++
allows one specify at which level the `const` keyword is applied to a pointer.
For example, `char const* const*`, `char * const*` and `char const**` are three
different types. But we don't have such fine-grained control when working with objects.
For example, `const std::string s;` does not guarantee that data pointer by `s.data()`
is not altered (you know how).

**Cost**: because a `const` object is not really immutable, the compiler cannot optimize
by assuming the value of the object does not change during its lifetime.

## `const` is not automatically propagated when calling a member function of an object through a pointer.
This is related with the above-mentioned lacking fine-grained control over constness
when working with objects containing pointers as member variables, but happens when
one wants to call a member function of the pointed object through a member pointer.
We know that with C++, member functions can be overloaded with respect to the constness
of `*this` object. Consider the following example:
```cpp
struct A{
    //...
	void do_sth() const;
	void do_sth();
};

struct B{
	//...
	B* bptr;
	void proxy_do_sth(
		// if bptr is const, call the const version of do_sth
		// if bptr is not const, call the non-const version of do_sth
	)
};
```
As the example shows, class `A` has two overloaded member function `do_sth` based
on the constness of `*this` of an object of type `A`. Class `B` holds a pointer
to an object of `A`, and we want the following being done automatically.
```
const B b;
b.proxy_do_sth(); // disptach to the const version of do_sth();
B b;
b.proxy_do_sth(); // dispathces to the non-const version of do_sth()
```
With this small example, of course we can manually make things write by writing
some boilerplate. But this approach obviously does dot scale. If we have a shared
pointer to type `T`, and we want to access the pointed object as if it is `const T`,
we need ugly converting function like `std::const_pointer_cast`. The problem is
whenever you introduces a new pointer-like class, you need to provide a function
like `std::const_pointer_cast`. Even if one would like do that, for non-pointer-like
objects, say, that object 1 holds a pointer to object 2, which holds a pointer to
object 3, ..., and so on to a certain depth N. We can not say that if object 1 is
`const`, only `const` member functions along the calling chain can be called.
To ensure `const` correctness, the boilerplate needed grows  as N grows.

**Cost**: Ensuring `const` correctness involves write a lot of boilerplate code.
Even for thins as simple as a "getter" for a member variable of a class, one
needs to provide a `const` version and a non-const version.

## A `const` member function can modify global variables
