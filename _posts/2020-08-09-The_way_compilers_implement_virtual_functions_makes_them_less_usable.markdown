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
it immediately breaks ABI compatibility when the library gets updated.
Adding new virtual member functions will lead to rearrange the order of the elements of the virtual table,
breaking compiled calling code that has hardcoded the offsets of virtual functions in the virtual table into its binary.
Appending newl virtual functions only at the end of the class definition works theoretically,
but relative code should stay together for readability reasons, for locality is important for code maintainability.

Mainstream compilers implement virtual functions using so-called vtables.
A virtual member function is indexed with the form of base address of the vtable of the class
plus an offset corresponding to that function.
Each instance of the class gets added a private member of pointer, which is invisible to programmers,
pointing to the vtable of that class, while the vtable is just an array of function pointers.
The order of these pointers are the same as the order they are declared, if multiple inheritance is
omitted for the sake of not further complicating the discussion.
If the offset of an pre-existing virtual member function gets changed due to
new ones are added in the middle of the class, the code at caller’s side can not adapt to that without recompilation.
Because all the calling code has hard coded the index information into its binary.
The only option left is recompile all  code that relies on the library.
If the library is used by many users, that is simply too much a price to pay.
Thus, many C++ programmers just don’t use abstract classes as interfaces of libraries.
Those chose to do so always ended pretty frustrated. For example,
many COM libraries on Windows platform ended including version numbers in their name,
as "lib_version1. dll", "lib_version2.dll", …, "lib_versionk.dll".
That is the price to pay for ugly tricks to bypass the ABI problem.

If I am not wrong, the C++ language standard actually does not mandate how
virtual functions should be implemented.
It could have been done differently.
Another way to do it is devise a runtime virtual function dispatching mechanism based on the
RTTI (Run-Time Type information) of the "this" pointer and function parameters.
This way, as long as the ABI of how RTTI is implemented stays stable, no code would be broken
because of an abstract class gets updated.
Furthermore, if the compiler expose the mechanism to programmers, it will leads to
very flexible code, such as dding a new virtual function to a base class without
touching the code of that class. This can be done to instruct the compiler to add
to new element to the underlying Hashtable of virtual functions which maps type information of
the "this" pointer and function parameters to the actual implementation.
This might be even done at runtime. Manipulating virtual function dispatching at runtime, how cool it is.
With a little bit of effort, multiple dispatching should also be possible.
With a powerful enough language-level mechanism, programmers won't need to invent their own wheels.

If such an approach is adopted, there is other (possibly unexpected) benefits.
For example, In C, if one converts a function pointer to a void function pointer,
and later converts it back, he gets exactly the original function pointer.
This does not apply to C++, because the size of pointer to member functions is
different from pointer to non-member functions.
The reason is mainly because that pointers to member functions have to take
into account vtables and even multiple vtables in case of multiple inheritance.
If, instead, a functions, regardless being a free function, or an ordinary member function,
or a virtual member function, is of the same size with another function pointer of another type,
we can cast it to a `std::function_ptr` which to function pointers is
what `void` to data types. When it cast back, we just get just the original pointer.
this would unlock quite some interesting use cases.

