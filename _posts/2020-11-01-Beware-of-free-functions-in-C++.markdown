---
layout: post
mathjax: true
comments: true
title:  "Beware of free functions in C++"
author: John Z. Li
date:   2020-11-01 19:00:18 +0800
categories: c++ programming
tags: function-overloading, ADL
---
For any class member function in C++,
if it is a static one, outside that class,
it must be called via the type name of that class;
if it is an instance method, it must be called through an instance of that class.
Either way, the function call resolves to that specific function
as intended up to possible overloading resolution.

For free functions, rules get a little more complicated.
For one thing, there is so-called Argument-Dependent Lookup (ADL),
so that a function defined in the same namespace
where one of its argument-related type is defined must
get picked up in function name look-up.
 (ADL is not what we are going to talk about in this post.)
What we are going to talk about is that
the function name look-up rules for free functions
might leads to very subtle bugs if one is collaborating with other programmers in a large project.

Say that a project has its own namespace called `project_namespace`,
with a sub-namespace in that namespace called `subproject_namespace`,
and one is working in a innermost sub-namespace residing in `subproject_namespace` called `module_namespace`.

In the outmost namespace, there are an utility functions defined as  free functions,
and its called inside the innermost namespace, like below.
```cpp
    namespace project_namespace {
      void fun(A::S s) {
      std::cout << s << "\n";
      }
      //…
    }
    namespace project_namespace::subproject_namespace {
      //…
    }
    namespace project_namespace::subproject_namespace::module_namespace {
      class my_class {
      public:
      void g() {
      A::S s = "this is a string";
      fun(s);
      //...
      }
      };
    }
```
Everything just works. However, if another programmer who works in the `subproject_namespace`,
adds another free function in it, which is also called `fun`, like below
```cpp
    namespace project_namespace::subproject_namespace {
      //newly added function
      void fun(int i) {
      std::cout << i << "\n";
      }
    }
```
Now your code suddenly stops compiling.
The reason for that is in the name look-up phase,
the compiler only looks for functions with a name “fun”, regardless their signatures.
Once it has found one or more functions with that name in an enclosing scope,
it stops looking, and all name matched functions are subject to overloading resolution.
In our example, the compiler complains about not finding a matching function.
What makes things worse, is when the program compiles accidentally,
that is, the other programmer accidentally added a free function with a matching signature.
This can lead to hard-to-trace bugs.

One can avoid the unintended name-lookup issue by explicitly
specify which namespace the free function is located by calling
```cpp
    project_namespace::fun(s);
```
However, this will disable ADL too, which is sometimes unintended.
If one also wants to retain ADL, one can instead do
```cpp
    using project_namespace::fun;
    fun(s);
```
so that if a function is named “fun” is defined in namespace `A`,
it is also considered in overloading resolution.

One take-away is that: whenever you spot a free function call in C++,
consider explicitly specifying its namespace if no ADL intended,
or bring the function name into the current scope explicitly with a `using` directive
if ADL is desired.

