---
layout: post
mathjax: true
comments: true
title:  "Create an alias for a function in C++"
author: John Z. Li
date:   2021-12-06 21:00:18 +0800
categories: c++
tags: function, constexpr, mem_fn
---
An alias for something is another name for that something. In C++, for a variable `A a`,
we can create an alias for `a` simply by `A& b = a;`. We say that `b` is an alias for `a` in the sense
that whenever `a` is used, `b` can be used instead. But how should we do this for functions?

For simple case, this is actually very easy:
```cpp
int fun (int);
constexpr auto f = fun;
```
`f` hare is actually a function pointer with its type being deduced as `int (*const)(int)`.
Because its constness, the compiler can also optimize function calling through `f` just as efficient as `fun` itself.
So, no performance loss, and `f` is truly an alias of `fun`.

The same trick works if `fun` is a static member function or even a function template:
```cpp
struct S {
  static int fun(int);

  template <typename... T> static auto add(T... t) { return (t + ...); }
};

constexpr auto f = S::fun;
constexpr auto g = S::add<int, int, int>;
```

Unfortunately, this does not work for overloaded functions. To disambiguate between
overloaded functions, one has to use `static_cast`:
```cpp
struct S {
  static int fun(int);
  static double fun(double);
};

constexpr auto f = static_cast<int(*)(int)>(S::fun);
constexpr auto g = static_cast<double(*)(double)>(S::fun);
```

Up till now, we are basically dealing with function pointers. Function pointers of free functions
and static member functions are just like function pointers in C, well defined and well behaved.
Though, we can generalize this approach to non-static member functions, but the syntax for doing so is ugly.
If we do the following:
```cpp
struct S {
  int fun(int);
};

constexpr auto f = &S::fun;
```
We have to call the function like below:
```cpp
S s;
auto i = (s.*f)(2);
```
This is kind of beating the purpose of creating alias for functions. Instead of making code clearer, it fills code with syntax noises.
For non-static member functions, we can use `std::mem_fun` instead. An object of `std::mem_fn` is just a wrapper over a member function pointer,
but with better syntax. For example:
```cpp
struct S {
  int fun(int);
};

auto f = std::mem_fn(&S::fun);
// After C++20, the above can be constexpr
constexpr auto f = std::mem_fn(&S::fun);
```
After creating `f`, it can be used as if it is a free function:
```cpp
  int i = 0;
  S s;
  S &sr = s;
  S *sp = new S;
  auto ssp = std::make_shared<S>();
  auto sup = std::make_unique<S>();

  f(s, i);    // call with the first parameter of type  S
  f(sr, i);   // call with the first parameter of type S&
  f(sp, i);   // call with the first parameter of type S*
  f(ssp, i);  // call with the first parameter of type shared_pointer<S>
  f(sup, i);  // call with the first parameter of type unique_ptr<S>
```
Actually, any pointer-like type that defines the de-reference operator that returns `S` is valid.

Of course, we can always wrap a function or a member function in a lambda or use `std::function`.
Though for the task of  creating an alias for a function, they both seems like over-killing.
