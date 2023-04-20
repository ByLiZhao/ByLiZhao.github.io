---
layout: post
mathjax: true
comments: true
title:  "Conversion operators considered harmful"
author: John Z. Li
date:   2023-04-20 21:00:18 +0800
categories: C++
tags: conversion_operator, conversion_constructor, overloading_resolution
---
There are two ways to define a implicit conversion sequence in C++. (many people
argue that the whole idea of implicit conversion in a programming langauge is bac.
Let us put this aside for this post.) One is via a so called conversion operator.
In any class `A`, if you define an operator in the form of `operator B()` or
`operator B() const`, where `B` is a type, it defines an implicit conversion
sequence from `A` to `B`. Another approach is through a constructor that takes
a single paramter which is not marked as `explicit`, like `A::A(const& B)`. This
kind of constructors are sometimes called conversion constructors. (Again, many
people in the C++ community argue that every constructor should be marked as `explict`.
I have some sympathy to that statement. But again, let us focus on what this post
is about.)

This post tries to argue that conversion operators are a bad idea, and in general
should be avoided. Below is my analysis.

First of all, programming is a social activity. Often times, we don't write code
for our own usage. Instead we produce code so that can be used by someone else.
The implicit social contract here, is that
> A trivial addition to a piece of code should not break code that relies on it.
This is, IMHO, is exactly why conversion operators should be avoid.

Say, you are a C++ programmer, and you published a C++ library which contains the
definition of class `A`. Another C++ programmer, who uses your library as a dependency,
created another class `B`. He thinks that `B` needs to conversion constructor so
that he can construct instances of `B` easily from instances of `A`. So, he simply
wrote down some code like below:
```cpp
class B{
	B(const A&) {
	// create an instance of B out of an instance of A.
	}
};
```
So far so good. After all, it is just some very trivial code.

But sometime later, out of some reason, You, the author of class `A` determined to
add a conversion operator to `B` inside `A`'s definition. Let us say, you just
wrote down something like below:
```cpp
class A{
	operator B(){
	// crate an instance of B out of an instance of A
	}
};
```
After the author of class `B` updated the library that contains the definition of
`A` in this project, he would as usual do a new building. Code like below will
compile without any problem, but now, it calls the conversion operator inside class
`A` instead of the conversion constructor in class `B`.
```cpp
B b = A{};
```
This is not something that is usually expected from the perspective of library
users. If you ever encounter this, you can only hope that you are lucky and the
behavior of the conversion constructor and the conversion operator is exactly the
same. But for any but trivial cases, this is something too much to expect. We
should bear in mind that there are many ways one can define a conversion constructor,
for example, it can be like `B(A&)`. The same applies to conversion operators, one
might define the conversion operator as `operator b() const`. We have not went into
things like whether the operator/constructor is of `noexpect`, and whether they
cause some side-effects inside its body, etc. This is means, when we are lucky,
the compiler will complain ambiguity in overloading resolution. If we are unlucky,
we might be trapped later with very hard to debug isues because that line of code
does not do what you think it is doing.

So, if you are to define a class, let the users of that class determine whether
they want implicit conversion from your type. If they want implicit conversion
in their code, they can add conversion constructors to their classes.

The only expection I can think of is that when converting to types in the standard
library, for exmpale, you want your custom string class implicity converting to
`std::string_view`. But even in this case, it might be a bad idea, because you
can make the conversion explicit by defining a method called `to_string_view`.
It will be a little bit more verbose, but might save you a lot of troubles later.

