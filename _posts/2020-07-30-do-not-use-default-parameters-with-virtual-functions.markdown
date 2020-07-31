---
layout: post
mathjax: true
comments: true
title:  "Don't use default parameters with virtual functions"
author: John Z. Li
date:   2020-07-30 16:44:18 +0800
categories: c++ programming
tags: virtual-function default-parameter
---

The below code will print the following:

> in derived class with parameter: base class.

That is not what one would expect. The reason behind this is that C++ compilers 
determines the value of defaulted parameters solely based on the type of the pointer,
from which a virtual function is called.
```cpp
#include <iostream>
#include <string>
class base {
public:
  virtual void do_something(std::string x = "base class.") {
    std::cout << "in base class with parameter: " << x << std::endl;
  }
};
class derived : public base {
public:
  virtual void do_something(std::string x = "derived class.") override {
    std::cout << "in derived class with parameter: " << x << std::endl;
  }
};
int main() {
  derived d;
  base *base_ptr = &d;
  base_ptr->do_something();
  return 0;
}
```
{% comment %}
math formula can be input like below:
{% raw %}
  $$a^2 + b^2 = c^2$$ --> note that all equations between these tags will not need escaping! 
{% endraw %}
{% endcomment %}
