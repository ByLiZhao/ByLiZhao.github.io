---
layout: post
mathjax: true
comments: true
title:  "The way compilers implement virtual functions makes them less usable"
author: John Z. Li
date:   2020-08-09 19:04:18 +0800
categories: c++ programming
tags: virtual-function, function-pointer
---

In C++, virtual member functions could have been more useful.
Unfortunately, the way they are implemented by mainstream compilers makes
them less so.

Using abstract classes as library interfaces is intuitively attractive.
It clearly expresses the idea of separation of interface and implementation.
Abstract classes are what many programmers have in mind when they think of interfaces.
However, if a library exposes an abstract class as its interface,
it can easily breaks ABI compatibility in future when the library gets updated.

Mainstream compilers implement virtual functions using virtual tables, or just vtables.
A virtual member function in compiled code is encoded in the form of base address of the vtable
plus an offset of the function in the vtable.
Each instance of a virtual class gets added a private member of a pointer by the compiler,
which is invisible to programmers, pointing to the vtable of that class, which is just an array of pointers to virtual member functions of that class.
The order of these pointers are the same as the corresponding member functions are declared.
To not further complicate the situation, let us not bring multiple inheritance into the picture.
At the call site, these offsets are hardcoded into generated binary.

This means, If the offset of a virtual member function gets changed due to
new virtual functions are added in the middle of the class, the code at caller’s side can not adapt to that without a whole-program recompilation.
The only option left is recompile all  code that relies on the library everytime the library gets updated.
But this is simply impractical, if the library is used by many users, and some users might not even have the source code.
Thus, many C++ programmers just don’t use abstract classes as interfaces of libraries.
Those chose to do so tend to become pretty frustrated. For example,
many COM libraries on Windows platform ended including version numbers in their name,
as "lib_version1. dll", "lib_version2.dll", …, "lib_versionk.dll".
That is the price to pay for ugly tricks to bypass the ABI problem.
Another workaround is to follow the rule that new virtual functions are always appended at the end of class declaration.
But this is fragile in terms of human error.

If I am not wrong, the C++ language standard does not mandate how
virtual functions should be implemented.
It could have been done differently.
A way to do it is through a runtime virtual function dispatching mechanism based on the
RTTI (Run-Time Type information) of "this" pointer and function parameters.
The trick is to transfrom a virtual function call to the following:
1. First, find the intended function via a hash table looking-up, using
   - RTTI informtion of `this` pointer as key if only single dispatching is needed,
   - RTTI information of `this` pointer plus that of all the function parameters as key for multiple dispatching.
2. After the function is found, invoke the function with `this` (potentially other parameters for multiple dispatching) to the correct type.
For example, if `class A` and `class B` both implements a virtual member of `fun()` of interface `I`.
a hash table is constructed as below:
```txt
 RTTI(A) --> A::fun()
 RTTI(B) --> B::fun()
```
When below called is encoutered,
```cpp
class A a;
I * p = &s;
a.fun();
```
the compiler first finds the actual type of the object pointed by `p` using RTTI,
and then find the correct function to call by consulting the hashtable,
then proceed to called the function.
This appraoch is likely to incur some performance penalty, but a well-optimized implementation for the single disptaching case
should not be much slower than vtable, as in most cases, the hash table should be relatively small in size.

The benefit is, as long as the ABI of how RTTI is implemented stays stable, no code would be broken
because of an abstract class gets updated.
Furthermore, if the compiler expose the mechanism to programmers, it will leads to
very flexible code, such as dding a new virtual function to a base class without
touching the code of that class, that is the ability to extend a virtual class from the outside.
This can be done to instruct the compiler to add
an new key-value pair to the underlying Hashtable of virtual functions.
This might be even done at runtime. Manipulating virtual function dispatching at runtime, how cool it is.
Multiple dispatching also naturally follows.
With a powerful enough language-level mechanism,
programmers won't need to invent their own wheels when he needs multiple dispatching.

If such an approach is adopted, there is other (possibly unexpected) benefits.
For example, In C, if one converts a function pointer to a void function pointer,
and later converts it back, he gets exactly the original function pointer.
This does not apply to C++, because the size of pointer to member functions is
different from pointer to non-member functions.
The reason is that pointers to member functions have to take
into account vtables and even multiple vtables in case of multiple inheritance.
If virtual functions are implemented in the way we just described, any virtual function can be
encoded with the address the hash table plus a pointer to a RTTI object that encodes the actual type of `this`.
With some bit-tagging, pointers to free functions, function pointers to member function,
and function pointers to virtual member function, be be represented in a uniform binary format.
With this improvement, we can have a type called `std::function_ptr`, which to function pointers is
what `void` to data types. We can cast any free/member/virtual function to it, and when casting back,
we just get the original function.
This would unlock quite some interesting use cases.

