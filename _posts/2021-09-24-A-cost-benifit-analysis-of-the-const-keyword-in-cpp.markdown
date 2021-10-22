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
The perceived benefits of the `const` keyword in C++ can be summarized as below:
1. It protects data that is not supposed to be altered  from being accidentally altered.
For example, for simple objects of fundamental types marked as `const` in global or namespace scope,
after compilation, they might end up in residing in the "READONLY" segment
of the resulted object file. After an object file is loaded into main memory,
the operating system might place the "READONLY" segment into a read-only memory page.
Such a read-only memory page is protected by the OS from being written to. It is
kind of important that the programmer is able to specify
that the content in READONLY memory pages should not be modified after the process starts,
especially in security-sensitive contexts.
2. It allows the compiler to perform more optimization using tricks such as constant propagation.
For example, if the compiler knows that a vector is constant during its lifetime,
while iterating over the vector, it can substitute each call of the `size()`
method with the fixed value of the vector size. It can even unroll the loop into
sequential code.
3. When `const` is applied to function arguments passed by pointers or references,
it documents the programmer's intention that the corresponding object is not mutated
inside the function.

Of course, we know that there exist `const_cast` and `mutable` in C++.
These two, while making the semantics of `const` more complicated
(actually negating some benefits of using `const`),  are sometimes needed nevertheless.
1. For `mutable`, for example, if we have a class that is thread-unsafe,
and we want to make a thread-safe version of it by inheriting from it.
We can achieve that by adding a `mutable` mutex member to the derived class,
and delegating each method call to those of the base class except that the method
body is protected by the mutex.
By making the mutex `mutable`, all the `const` member functions of the base class
can stay `const`. That is, we are extending the base class without breaking its API.
The derived class can thus be used as a drop-in replacement for the base class.
2. `mutable` is also necessary when it comes to lambda functions in C++. Because
lambda  functions in C++ are `const` by default, in case we need mutable lambda functions,
we have to use `mutable` to opt out the default behavior.
3. The `const_cast` operator is useful because sometimes we need to bypass the
type system of C++, especially with working with external libraries.

The drawbacks of C++ `const` are discussed below:
## A `const`object in C++ is only head-const
A non-trivial C++ class usually contains pointers as member variables. For example,
A `std::string` object contains a `char *` pointer which points to a buffer that holds
the content of the string. When applying the `const` keyword to a C++ object, that
object is head-const only. *For a pointer, if the pointer itself is `const` but not the
chunk of memory pointed to by the pointer, we say that this pointer is head-const.
In contrast, if the chuck of memory pointed to is `const` but not the pointer itself,
we say that this pointer is tail-const*.
When working  with pointer, C++
allows one to specify at which level the `const` keyword is applied.
For example, `char const* const*`, `char * const*` and `char const**` are three
different types. But we don't have such fine-grained control when working with objects.
For example, with the following class and an instance of it:
```cpp
class S{
    size_t len;
	char * cp;
};
const S s;
```
The object `s` is marked as `const`, but it only means that `cp` itself is `const`,
but the data pointed by `cp` might still be altered.
If we pass `s` as a `const` reference to a function, for example, `fun(const S&)`,
The compiler can not be sure that `s` will not be altered somewhere along the calling chain,
even if `const_cast` is not used.

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
	A* aptr;
	void proxy_do_sth(
	    // what is needed
		// if aptr points to a const version of an object of type A, call the const version of do_sth
		// Otherwise, call the non-const version of do_sth
	)
};
```
As the example shows, class `A` has two overloaded member function `do_sth` based
on the constness of the `this`  pointer  of type `A`. Class `B` holds a pointer
to an object of `A`, and we want the following being done automatically.
```
const B b;
b.proxy_do_sth(); // disptach to the const version of do_sth();
B b;
b.proxy_do_sth(); // dispathces to the non-const version of do_sth()
```
With this small example, of course we can manually make things work by writing
some boilerplate. But this approach obviously does dot scale. If we have a shared
pointer to type `T`, and we want to access the pointed object as if it is `const T`,
we need ugly converting function like `std::const_pointer_cast`. The problem is
whenever you introduces a new pointer-like class, you need to provide a function
like `std::const_pointer_cast`. And whenever you has a pointer-like data member in your class,
Even if one would like do that, for non-pointer-like,
you need to worry whether the const-correctness will be preserved.
Say, object 1 holds a pointer to object 2, which holds a pointer to
object 3, ..., and so on to a certain depth N. We can not say that if object 1 is
`const`, only `const` member functions along the calling chain can be called.
To ensure `const` correctness, A lot of  boilerplate code is needed.

Note that `std::experimental::propagate_const` might be a fix to the problem. Consider
the following code:
```cpp
#include <iostream>
#include <memory>
#include <experimental/propagate_const>

struct X
{
    void g() const { std::cout << "g (const)\n"; }
    void g() { std::cout << "g (non-const)\n"; }
};

struct Y
{
    Y() : m_ptrX(std::make_unique<X>()) { }

    void f() const
    {
        std::cout << "f (const)\n";
        m_ptrX->g();
    }

    void f()
    {
        std::cout << "f (non-const)\n";
        m_ptrX->g();
    }

    std::experimental::propagate_const<std::unique_ptr<X>> m_ptrX;
};

int main()
{
    Y y;
    y.f();

    const Y cy;
    cy.f();
}
```
This piece of code will print the following
```txt
f (non-const)
g (non-const)
f (const)
g (const)
```
Though the usage of `std::experimental::propagate_const` is contagious. Existing code
is unlike to adopt it. If you changes every data member of `unique_ptr<T>` to
`propagate_const<unique_ptr<T>>`, the API of a library involving unique pointers
effectively changed. We may never see it being widely used.

**Cost**: Ensuring `const` correctness involves write a lot of boilerplate code.
Even for things as simple as a "getter" for a member variable of a class, one
needs to provide a `const` version and a non-const version. Making sure of const-correctness
along a chain of function calls is too tedious to do.

## A `const` member function can modify global variables
A `const` member function of a class is guaranteed not to alter the value of class
data members. But
1. A `const` member function is allowed to modify a `static` data member of the class.
Wait what, `const` member functions will not change the `this` object by changing its
state, but it might change the behavior of **all** the instances of the class by changing
a `static` member of that class.
2. A `const` member function is also allowed to change global variables, or a namespace
scoped variables. I have to idea whey the language is designed this way. There is very
little you can predict what a `const` member function of a class will NOT do by looking
at its signature.
3. You can not mark a free function as `const` to indicate that the free function will
not change the value of any variable in its scope, or in the upper scopes.

**Cost**:  In other words,
there is no way that the programmer can specify a C++ function as "local". Locality is
kind of important for the readability of a programming language.
