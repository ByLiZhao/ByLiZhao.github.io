---
layout: post
mathjax: true
comments: true
title:  "Providing a stable ABI via PIMPL"
author: John Z. Li
date:   2020-09-30 19:00:18 +0800
categories: c++ programming
tags: ABI, PIMPL
---
In a previous [blog post](https://bylizhao.github.io/c++/programming/2020/08/09/The_way_compilers_implement_virtual_functions_makes_them_less_usable.html),
I talked about why abstract classes are not suitable to serve as module interfaces,
for the reason that ABI compatibility can easily get broken with module updates.
When static linking is not a valid option, it is kind of important to keep
interfaces stable down to the ABI level.
When a new version of a dynamic library is released,
it should be possible to just replace the old version
of it with the new one, and programs relying on it should just work fine.

There is a saying that you can solve any problem in CS by introducing
an extra layer of indirection. PIPML, short for pointer to implementation,
does just that.
The idea is that a class exposed as part of interface only
contains public member function
(possibly public member variables too, but maybe not a good idea,
implement getters and setters instead)
that forward calls to corresponding member functions of
a implementation class, to which the interface class holds
a pointer as a private field. A demo code is as below
```cpp
    // in a header file
    class interface {
     private:
     class impl; // a forward declaration, its definition is in a cpp file
     impl * ptr; // a pointer pointing to an instance of class impl.
     public:
     // constructors, assignment operators omitted
     // destructor omitted
     void do_something();
    };

    // in a cpp file that include the header:
    class interface::impl {
       // implement the impl class here
       public:
    	void do_something(){
    	//real work is done here
    	}
    };
    //implementation of constructors and destructors of interface class omitted
    interface::do_something(){
    	ptr→ do_something();
    }
```

Since class `interface` is only a concrete class that does not rely on
internals of the `impl` class, the `impl` class can be forward declared.
That means a caller does not have to know anything about the `impl`
class other than the fact it is a class.
No matter how the `impl` class may change in future,
the memory layout of class `interface` does not change.
Thus it will not cause a problem to add new public member functions to interface,
as long as old ones don’t get removed,
it will not break binaries of caller’s side.
ABI stability is thus achieved.

Several features added into modern C++  make the `pimpl` approach even better.

The first one is unique pointers added in C++11.
Instead of using a plain pointer, one can use a `unique_pointer<impl>`.
By doing this, the lifetime of the underlying `impl` object
is managed by the corresponding interface object.

Second, move constructor, move assignment operator and destructor of class interface can be defaulted. So no manually implementing them anymore.

Third, copy elision will kick in when arguments and return values are passed between methods of the interface class and their counterparts in the `impl` class.

However, there is still one thing that is broken with this pimpl implementation.
Suppose that there are overloadied methods of class `impl` that differ only by `const` specifiers, as below
```cpp
    class interface::impl {
       // implement the impl class here
       public:
    	void do_something(){} // act on non-const instances
    	void do_something() const {} //act on const instances
    };
```
After method forwarding, the information of constness is lost.
There is an experimental feature called `propagate_const` can be used to solve this problem. `propagate_const`
is a class template that can be instantiated with pointer types,
including smart pointers. The sole purpose of it, as its name suggests,
is to propagate the constness of the object,
to where the pointer  is dereferenced.

