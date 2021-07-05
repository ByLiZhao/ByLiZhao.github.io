---
layout: post
mathjax: true
comments: true
title:  "Inline variables in C++17"
author: John Z. Li
date:   2021-01-30 19:00:18 +0800
categories: c++ programming
tags: inline, ODR
---
One guideline of the C++ language design is that macros should be used
as less as possible.
With C, if one wants to define a compile constant, he may do it by
```c
    #define pi (3.141592653589793238462643383279502884)
```
The shortcomings of C macros are well known, for they are nothing
but text substitution.
If we want to define pi as a real constant,
we CAN NOT simply add the following in a header file
```c
    const double pi = 3.141592653589793238462643383279502884;
```
because a (file-scoped) global variable in C has external linkage regardless
the `const` qualifier, and the compiler will complain
that ODR (one definition rule) is violated.
The way to fix this in C is by putting the following in a header file
```c
        extern const double pi; // in some_header.h
        // extern can be omitted here.
```
and define pi in a separate C file
```c
        #include "some_header.h" //optional
        const double pi = 3.141592653589793238462643383279502884;
```
This solves the problem of violating ODR, but has the unpleasant effect that even if
`pi` is a compile time constant, its definition is in another compilation unit which is only accessed during link time.

C++ solves this particular problem by decreeing that a `const`
variable in the global scope has internal linkage.
Great, keeping this information in mind, we now can just put the following in a header file
```cpp
// in a cpp header
// constexpr in this case works too
    const double pi = 3.141592653589793238462643383279502884;
```
and use it without worrying about ODR. But wait, what if we want define a global object like
```cpp
     const X x; // in a header called x.h
```
where X denotes a class with its own constructor. If we include the header `x.h`
into two (or more) different cpp files,
we don't really get a global, but really TWO (or more) independent objects.
(if you omit the `const` specifier, you actually get undefined behavior
because of violating ODR).
This, in most cases, is not what the programmer wants,
especially when X's constructor has side effects,
and the intention is that the constructor shall be only executed once.
(Otherwise, the constructor will be executed
twice along with all its side effects, and the order of execution is unspecified.)

If we really think about it, it is obvious that C++ has solved the wrong problem.
The real problem that needs to be solved is that the programmer wants a way to express his intention like

>    Hi, compiler, don't mind ODR for this object.

Instead, by altering the linkage rule of global constants,
the problem is only accidentally avoided in an ad hoc way.
It soon occurs to people that C++ already has the semantic element,
namely the `inline` keyword to do just what really needs to be done, namely relaxing the ODR rule.

Recall that you can put the definition of a class in a header file
in C++ and the compiler won't complain a thing.
The reason is that the member functions of the
class is implicitly marked as inline by the compiler.
It is like a directive to the compiler, which says

>    Hi, compiler, I know the following definitions will appear multiple times,
>    but don't worry, they are all identical. So, just compile the program without complaining.

It is kind of funny that the problem hadn't been solved until C++17. With C++17, we finally can simply put
```cpp
     inline const X x;
```
in a header file and things will happen as you expect.
Notice that the `inline` specifier here actually cancels the effect of `const` specifier on x's linkage.
Also notice that now it is fine to also write `inline X x;` in a header file,
which can also be placed in a namespace.
And if you do put a `inline const X x;` in a namespace in a header file, it now has external linkage as it should.

Another usage of the inline keyword in C++17 is to decorate static member
variables in a class.
Before C++17, static member variables of a class suffer from the same problem
as global variables as mentioned above.
If a class has a static member variable,
you have to define it in a separate cpp file while other parts of the class is in a header file
(unless the static member variable is a constant integer value).
With C++17, we can just go with
```cpp
    class X{
        inline static int count = 0;
    };
```
Curiously enough, the same problem didn't happen for
function-local static objects even in C++03,
making you wonder why the language designer didn't apply
the same principle consistently also to statically member variables.

