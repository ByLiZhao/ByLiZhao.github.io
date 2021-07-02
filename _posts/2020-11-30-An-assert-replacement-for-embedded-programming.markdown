---
layout: post
mathjax: true
comments: true
title:  "An assert replacement for embedded programming"
author: John Z. Li
date:   2020-11-30 19:00:18 +0800
categories: c programming
tags: assert
---
When doing C programming on MCUs,
it sometimes become impossible to use `assert` because the C compiler does not implement it.
To circumvent this, and partially regain some convenience provided by `assert`
, we can go with the following macro definition:
```c
    #if defined(NDEBUG)
    #define CHECK(fun_name, error_no, ...) ((void)0)
    #else
    #define CHECK(fun_name, error_no, ...) \
      do {  \
      if (fun_name(__VA_ARGS__))  \
      return error_no;  \
      } while (0);
    #endif
```
This piece of code introduces a variadic macro called CHECK,
which takes a function name as its first parameter,
an `error_no` to return as its second parameter,
and some unspecified number of parameters with which the function will be called.
The macro is switched on or off with the same macro definition `NDEBUG`
that also enable or disable asserts.
Usage of the macro is straightforward.
For example, while setting resolution of a LCD device, one defines the following checking function
```c
    static int check_resolution(int i, int j) {
      if (i <= 0 || j <= 0)
      return 1;
      return 0;
    }
```
which checks whether both integer `i` and `j` are positive,
returns 0 on success and 1 otherwise.
Then in the function `set_resolution`, one can invoke runtime check as
```c
    int set_resolution(int i, int j) {
      // check the precondition of the function
      CHECK(check_resolution, 1, i, j);
      // setting resolution
      return 0;
    }
```
So, if check fails inside the function,
the function will early return with value 1.
This kind of check can be applied during the debug phase
to help quickly spot mistakes made by the programmer and
removed in the release version, so it does not
introduce extra performance loss

