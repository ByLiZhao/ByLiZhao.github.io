---
layout: post
mathjax: true
comments: true
title:  "A taxonomy of CPP's inheritance semantics"
author: John Z. Li
date:   2021-01-02 19:00:18 +0800
categories: c++ programming
tags: inheritance
---
What does inheritance mean in C++? It can mean a whole lot of different things.
Below is a brief list of 6 possibilities.
Notice that this does not mean that whenever inheritance is used in C++ code,
it is one of these 6 cases,
because they can be used combinatorically at the same time.

Notice that those names, and their symbols used below are not some standard notation
used in the literature.
They are used to facilitate more intuitive understanding of the matter.

1. Extension,

{A} -> {A, b}

Extending a class by adding more class elements
(instance variables, static members, virtual/non-virtual methods, etc.)
into a class to form a new class,
while the inherited class is embedded into the new one (as a sub-object).

2. Shrink

{A} -> {A/b}

This consists of two cases:

First case, a non virtual member method is effectively deleted by
appending `=delete` at the end of a member method which
shadows the one in the base class. An example is as below:
```cpp
    class A {
    public:
      int f(int a) {
      return a;
      }
    };
    class B : public A {
    public:
        int f(int a) = delete;
    };
```
Second case: the base class have two or more overloaded member function
(which might be virtual),
and the derived class introduce its own member function with the same name
(not signature),
shadowing those member functions with the same name of the base class,
thus effectively shrinks the class. An example is:
```cpp
    class A {
    public:
      virtual int f(int a) {
      return a;
      }
      virtual float f(float a) {
      return a/0.5;
      }
    };
    class B : public A {
    public:
      //this function shadows float f(float)
      // B b; b.f(1.0) will call int f(int);
      virtual int f(int a) override {
      return a + 1;
      }
    };
```
3. Modification

{A} -> {B}, A ~ B.

This happens when some non-virtual member function of the base class get
new definitions in the derived class, `A~B`
means there is one-to-one correspondence between members of the base class and the derived class.
After modification, class invariant of the base class may no longer holds,
meaning it is a logical error to pass an instance of the derived class when an object of the base type is wanted.

4. Implementation

{A} -> {A{B}}

This is what people mean when they “implement an interface”,
that is, inheriting from an abstract class (that is a class with at least one
pure virtual member method).
The derived class has the interface of the base class
and has its customized behavior.
Accessing an object of the derived class through a pointer/reference to
the base class leads to runtime polymorphism.

5. Customization

{A} -> {A or A(B)}

This is often called sub-typing. Like the previous case,
except that the base class’s virtual methods have default behavior
(they are not pure virtual),
and the derived class can choose to implement its own customization behavior,
or it can choose to reuse the base class’s default behavior.

6. Composition

{A}, {B} – {A, B}

This is what happens when one multi-inherit from two or more non-abstract classes.
Those base classes are combined into a new one.

We have not touched access modifiers in this post,
which are orthogonal to inheritance semantics.

