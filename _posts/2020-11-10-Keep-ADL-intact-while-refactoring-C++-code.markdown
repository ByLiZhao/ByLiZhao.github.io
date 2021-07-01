---
layout: post
mathjax: true
comments: true
title:  "Keep ADL intact while refactoring C++ code"
author: John Z. Li
date:   2020-11-10 19:00:18 +0800
categories: c++ programming
tags: ADL
---
Say that you have some code like below:
```cpp
    namespace a_ns {
      struct A {
      int i = 10;
      };
      void fun(A a) {
      std::cout << a.i << std::endl;
      }
    }
```
While refactoring, you decide that the class `A` is better to be
put into another namespace.
To avoid breaking free functions that using type `A` in their parameter list,
you leave a `typedef` in the original namespace, resulting code like below:
```cpp
    namespace a_ns {
      typedef b_ns::A A;
      void fun(A a) {
      std::cout << a.i << std::endl;
      return;
      }
    }

    namespace b_ns {
      struct A {
      int i = 10;
      };
    }
```
All works well, except it does not.

The problem is that if the function `fun` is used via ADL.
The rule of ADL name looking-up rules neglects typedefs,
only looking matching names in the namespace
the type is actually defined.
So, the refactored code does not compile, or, worse,
it compiles, by accidentally calling another unintended function.
Free functions in the same namespace as a type definition should
be considered as part of environment that must go with the type definition.
Thus, either you move all free functions along with the type to a new namespace,
or, add a using directive to introduce the function name in the new namespace.

Another thing to remember is that you do not want your function
to be ADL-used,
make it a static function of a static class.
By not making it a free function in the first place, refactoring becomes easier.

