---
layout: post
mathjax: true
comments: true
title:  "Const lvalue reference is useless"
author: John Z. Li
date:   2020-12-16 19:00:18 +0800
categories: c++ programming
tags: lvalue-reference, const
---
If an object is `const` qualified,
it means that it should not and perhaps could
not be modified during its lifetime.
After rvalue reference is introduced for
implementing move semantics,
the thing gets a bit complicated.
By the language standard, after an object is moved,
the moved-from object is in a valid but unspecified state,
that is, the object might be modified after being moved.
This contradicts constness of a `const` qualified object.

While designating parameter types of a function,
if the programmer intends to copy an object,
he could simply pass by value or pass by const lvalue reference.
If he intends to move the object, he will pass by rvalue reference.
This is reflected in C++ down to the standard library level,
for example, the `push_back()` member function of the `std::vector`
class has two overloading:
```cpp
    void push_back( const T& value ); //for copying
    void push_back( T&& value ); //for moving
```
What if one passes a const rvalue to `push_back`?
Because a const rvalue  can not bind to non-const rvalue reference type,
the first overloading is selected, that is, the object is copied, instead of being moved.
This, is quite counter-intuitive, though in this case, the const-ness of the object is respected.
```cpp
    const T obj;
    std::vector<T> v;
    v.push_back(std::move(obj)); //this code actually copied the object.
```
Even if an overloading that takes const rvalue reference
is added to the interface of `std::vector`,
the right thing to do is still simply copying the object,
because moving a const object is paradoxical.

It is thus obvious that const rvalue reference is useless?
The only time it is been used in the standard library, as far as I know,
is in the `std::reference_wrapper` class to disable rvalues from
binding to `ref()` and `cref()`.
A type exists sorely for the purpose that functions
taking it as parameter type can be explicitly deleted.
If we think about it, it clearly indicates a design flaw of the language.
If const rvalue reference is disallowed in the language in the first place,
one could simply disable `ref()` and `cref()` on rvalues as below:
```cpp
// note this not how ref and cref are implemented in the standard library.
    template <class T> void ref (T&&) = delete;
    template <class T> void cref (T&&) = delete;
```
No one will miss it for any reason.

