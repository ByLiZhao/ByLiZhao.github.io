---
layout: post
mathjax: true
comments: true
title:  "std::initilizer_list, a wasted opportunity for C++"
author: John Z. Li
date:   2025-11-08 20:00:00 +0800
categories: programming
tags: c++, initializer_list, template
---
Arrays have well-known issues in C++.
Most of them are caused by the fact that arrays decay into pointers when passed as function parameters,
and automatically convert to pointers in assignments.
We can't fix this without breaking a lot of existing code.
Yet, even without breaking backward compatibility, the comunity once had a chance to fix it.
Unfortunately, instead of fixing the real issue, `std::initilizer_list` was added into the language.

When adding new features to  a programming language, we usually have certain use cases for that new feature.
In the case of `std::initializer_list`, the use case is unified initialization.
The standard committee created a new type, that is `std::initializer_list`, for that specific purpose.
To make it work, complex rules about it are hard-coded into the language standard:
- A `std::initilizer_list` has an array of underlying data of the same type (called the backing array).
- Yet, it does not own the underlying data. A instance of `std::initializer_list` is just a lightweight proxy to the backing array.
- The backing array ends its lifetime as if it is a temporary object.
- Yet, initializing an instance of `std::initializer_list` extends the lifetime of the backing array
  in the same way that binding a temporary to a const reference.

When there are so many special rules with a type, that type is really special.
Users of C++ can't make a type of similar properties using standard C++.
The type has to be hard-coded into as a special-case type in the compiler.
When you really think about it, the introduction of `std::initializer_list` is only trying very hard to hide a more fundamental problem of C++.

Thinking in terms of types, there are kinds of array types in C++:
1. `T[N]`, where `T` is a type and `N` is a fixed compile time non-negative integer.
   For example, `char arr[2]` defines an array of `char` with length `2` named `arr`.
2. `T[]`, runtime arrays. The length of a runtime array is only known at runtime, usually created with operator `new[]`.

But there is actually a third array kind: *Arrays with compile-time known sizes but they are only known when the array is actually constructed*.
The need for `std::initializer_list` is the result of not explicitly spell out the existence of such arrays.
A constructor, or any other functions/methods, takes a `std::initiaizer_list` is nothing but one that takes
such an array as paramter. The size of the array for each call is a compile time constant,
yet the actual size is only known when the compiler sees the code that invoking the constructor.

Had such a type been introduced into C++, assuming the syntax is `T[*]`, then a constructor that takes such an array can be written as
```cpp
class C{
	// the array size is a compile time constant, but we do not know it at function definitin site
	// it is only known when this function is actually called
	C(const T (&arr)[*]){ // take the array by const reference
		constexpr size = sizeof arr / sizeof (T); // a compile time constant
	}
};
```
No special rules would be needed for the new type `T[*]`, it follows the same rules as `T[N}`.
The language standard would only need to define such a type, plus that any `T[N]` type can be promoted to `T[*]`.
Since it is a single type, a lot of code can be simplied in C++.
C++ are littered with template specialization to take `T[N]` as paramter, often in the form of
```cpp
template< class T, std::size_t N >
constexpr ReturnType do_sth( const T (&array)[N] );
```
Depending on context, the type of `T` is often already known, but a function template instead of a simple function
has to be used.

An example is `std::format_string` (introduced in std::basic_format_string in C++ 20).
If we call `std::print` with only one parameter which is a string literals.
But,
```cpp
std::print("A"); // The constructor of std::format_string deduces that the passed-in type is char[2]
std::print("AB"); // the deduced passed-in type is char[3]
std::print("ABC"); // deduced type char[4]
\\ ....
```
For each unique length `N`, there is a `std::format_string<char, N>` constructor template instantiation
If we had the new `T[*]` type, it would be done by a single constructor taking `const char (&)[*]` as parameter.
`std::initializer_list` can't do this, because it is not an array type, and the backing array is deliberately kept from programmers.
Despite the fact the backing array is modeling `T[*]`, it has been never spelled out nor recognized as an actual type.
