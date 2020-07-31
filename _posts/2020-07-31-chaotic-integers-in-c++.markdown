---
layout: post
mathjax: true
comments: true
title:  "Chaotic integers in C++"
author: John Z. Li
date:   2020-07-31 07:16:03 +2020
categories: c++ programming
tags: integer-types
---

Integer types in C++ is really problematic. Languages and compilers should be your friends, but they keep make you surprised.

* When a signed integer is compared to an unsigned one, the signed integer is implicitly converted to an unsigned one, see the below example:

```cpp
#include <iostream> 
     
int main() 
{ 
    unsigned int ui = 2; 
    int i = -1; 
    if (ui < i) //true 
    	std::cout << "This shouldn't happen, but happens silently." << std::endl; 
    return 0; 
} 
```
 This means, if you have a function which takes an object of container type as parameter, and someone passes in an empty container, funny things would happen:

```cpp
#include <iostream> 
#include <vector> 
#include <string> 
     
void fun(std::vector<std::string> vs) { 
    for (int i = 0; i <= vs.size() - 1; ++i) { 
    	std::cout << vs[i] << std::endl; 
    } 
} 
int main() 
{ 
    fun(std::vector<std::string>{"first", "second"}); //works 
    fun(std::vector<std::string>{}); //bang, blow up in your face 
    return 0; 
} 
```

* When doing integer arithmetic, silent promotion might happen, changing both type and signed-ness, as below:

```cpp
#include <iostream> 
#include <type_traits> 
int main() 
{ 
    unsigned short a = 0; 
    unsigned short b = 0; 
    decltype(a + b) c = 0; 
    std::cout << "The size of a is: " << sizeof(a) << std::endl //2 
    	<< "The size of c is: " << sizeof(c) << std::endl; //4 
    std::cout << "is c unsigned? " << std::boolalpha 
    	<< std::is_unsigned<decltype(c)>::value << std::endl; //false 
    return 0; 
} 
```

* In old days, people rely on the fact that `a + b < a`
is true for signed integer a and a positive b, to detect whether overflow happens.
But the language standard has defined overflow for signed integers being undefined behavior since then.
Today’s compilers are very *smart* by performing optimization based on the assumption that undefined behavior does not happen. 
The compiler might deduce that since b is nonnegative, `a + b < a`will never be true, 
and optimize all related code away. 
So it is hard to tell what the following piece of code will output: 
Will the while loop be *optimized* to a infinite loop? 
Will only the if clause be *optimized* away? Or 
the whole program will be *optimized* to an Non-op.

```cpp
#include <iostream> 
int main() 
{ 
    short us1 = 0; 
    const short us2 = 1; 
	while (us1 + us2 > us1) { 
	us1++; 
    if (us1 + us2 < us1) 
		std::cout << "will program hit here, if yes, how many times? << std::endl; 
    } 
    return 0; 
} 
```

* The root cause of the above mentioned problems concerning integer types in C++ is really that unsigned integers and signed integers actually having different semantics other than signed-ness. According the language standard, An unsigned integer type uintp_t obey the laws of arithmetic modulo 2^p, making it more like a representation of a finite integer ring, that is, for uintp_t a, b, there is
{% raw %}
 $$𝑎+𝑏:=𝑎+𝑏\,\mod{2^𝑝},\quad 𝑎∗𝑏:=𝑎∗𝑏\,\mod{2^𝑝},$$
{% endraw %}
 where $:=$ reads "defined as". On the other hand, signed integer types are supposed to model the  set of integer numbers. 
 The fact that they have limited size and range is of technical reasons. 
 That is why signed number overflow is regarded as undefined behavior. 
 Because it is programmers responsibility that he chooses int type of proper size so that all things work in such a way as if they are normal integer numbers. 
 Since unsigned integers and signed integers are actually of different types, implicit conversion between them should not have been allowed.


{% comment %}
math formula can be input like below:
{% raw %}
  $$a^2 + b^2 = c^2$$ --> note that all equations between these tags will not need escaping! 
{% endraw %}
{% endcomment %}
