---
layout: post
mathjax: true
comments: true
title:  "If you want to pass a string, make it explicit"
author: John Z. Li
date:   2021-02-07 19:00:18 +0800
categories: c++ programming
tags: string, overloading-resolution
---
If you have a member function which takes strings as its parameters in C++,
it might be a good idea to always call the function by explicitly
initializing strings to be passed in.
Below is a minimal example to show why it is necessary.

Say, you have the following definition of a simple class
```cpp
    class S {
    public:
      std::string fun(std::string product_name, std::string product_category);
    }
```
With an object named `s` of the type `S`, you can call the member function like
```cpp
      s.fun("this product", "this category");
```
All will work well, until someone else adds a member function with the following signature:
```cpp
     std::string S::fun(std::string product_name, bool on_shelf );
```
After recompilation, you will notice that your code stops working
(or worse, it keeps working, just not in the way you expect).
The reason is that your code calls the newly introduced `fun()` now.
Without going to language details of overloading resolution,
one could avoid this kind of bugs by making the following part of his coding style:

>    If you want to pass strings into a function, always pass them  explicitly.

With this rule, the above sample code should be changed into
```cpp
    s.fun(std::string{"this product"}, std::string{"this category"});
```
It is a little bit verbose, but it is better safe than lazy.
It is a pity one can not decorate a member function with the keyword `explicit`
to disable implicit conversion of parameters, though.

