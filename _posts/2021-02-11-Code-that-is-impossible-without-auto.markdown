---
layout: post
mathjax: true
comments: true
title:  "Code that is impossible without auto"
author: John Z. Li
date:   2021-02-11 19:00:18 +0800
categories: c++ programming
tags: type-deduction, auto
---
Consider the following code snippet:
{% raw %}
```cpp
    auto get_colored_text() {
      struct Red_text {
        std::string Text;
        Red_text(std::string s) : Text(s) {}
      };
      Red_text red_text = std::string{"\033[1;31mHello World\033[0m\n"};
      return red_text;
    }
```
{% endraw %}
The function `get_colored_text()` returns an object of the type that
is defined inside the function.
Without `auto`, this function can not be defined.
The reason one might want to do this is that he might want to change
the implementation of the function, even its definition,
without also changing all code that uses this function.
He can do that because the definition of the return type is only visible from
the inside of  the function.

This means the caller of the function also has to use `auto`,
because the return type of the function is unknown to him.
(`decltype` will also work, but leads to more verbose code.)
Code like below will work as expected:
```cpp
      auto what_is_this = get_colored_text();
      // the following code will make the compiler complain about "unknown type
      //  Red_text red_text = get_colored_text();
	  // the following will not.
      std::cout << what_is_this.Text << std::endl;
```
Notice that after the type of variable `what_is_this` is deduced,
its public members also become accessible to the caller.
If one wants to enforce a certain interface of the return type of the function,
this approach can be a lightweight replacement than having a base class and subtyping it.
This example is given with a  type defined in function scope,
the same principle applies to nested class definition
in class scope or in anonymous namespaces.
