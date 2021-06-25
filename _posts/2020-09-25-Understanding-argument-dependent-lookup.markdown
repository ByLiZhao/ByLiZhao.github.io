---
layout: post
mathjax: true
comments: true
title:  "Understanding argument dependent lookup in C++"
author: John Z. Li
date:   2020-09-25 19:00:18 +0800
categories: c++ programming
tags: ADL, argument-dependent-lookup
---
One can draw an analogy between programming languages and natural languages,
where classes/objects are nouns, and functions/member functions are verbs.
OOP languages have a strong tendency to arrange verbs around nouns.
In a pure OOP language, the only way to call a function is through an
class/object that “owns” the function.
In a sense, the world of OOP is a noun-centric one,
in whch (assuming it is class-based OOP) nouns are masters,
functions are slaves, and each slave logically belongs to one master.
And there are namespaces, which are collections of classes.
So when one is talking about a function, he is talking about a function in a class in a namespace.

C++ is not a pure OOP language for good or for bad.
It provides means to break the wall when it makes sense.
It has free templated functions.
When a function logically span over many namespaces,
it just makes sense not to include it as a member function in each related classes,
or in each applicable namespaces.
Yet, we want to retain some control so that customized optimization points can
opt-in when necessary.
An example is how `std::swap` can be extended by providing user
defined implementation for some classes.
For most classes, (which are move constructible and move assignable,) `std::swap`
just works.
For classes that have a customized version of `swap`,
there needs a way to tell the compiler to use the customized version instead.

Function `std::swap` is a templated function.
Technically, one can at least try to specialize it on a given type.
(Functions in the std namespace is not allowed to be overloaded,
but it is OK to add template specializations.)
But, here, we do it by overloading function `swap` in a namespace other than
the `std` namespace.
(we are no going to enter a lengthy discussion of template function
specialization versus function overloading).
Say we provide a overloaded `swap` as below
```cpp
    namespace my_namespace{
    	class some_class{
    		friend void swap(some_class& obj1, some_class& obj2){
    		//implementation
    		}
    	};
    }
```
Argument Dependent Lookup (ADL) means, when the compiler sees the
following code (anywhere, not necessarily in my_namespace)
```cpp
    some_class obj1, obj2;
    swap(obj1, obj2);
```
it will check the namespace where class `some_class` is defined to see if there
is a function `swap` that matches the signature.
If it exists, that function is selected.
This does not seem very interesting, because we can always type
```cpp
    	my_namespace::swap(obj1, obj2)
```
The problem is,
this code does not work when there is no function `swap`
defined in `my_namespace`.
We don’t want to check if a user-defined version `swap` is provided whenever we want to swap two objects.
Plus, if we are writing generic code, checking is impossible.
Fortunately, ADL can come as the rescue, we can write code as below:
```cpp
    using std::swap; //std;;swap as a fall-back
    swap(obj1, obj2);
```
In this case, if an overloaded swap exists in `my_namespace`,
it will be used, if not, `std::swap` will be used as a fall-back.
Best part about this is that the mechanism works recursively,
thus the following code will use customized swaps of members of
some_class if they are defined in the same fashion.
```cpp
    namespace my_namespace{
    	class some_class{
    		friend void swap(some_class& obj1, some_class& obj2){
    		using std::swap;
    		swap(obj1.member1, obj2.member1);
    		swap(obj1.member2, obj2.member2);
    		//...
    		}
    	};
    }
```
Lastly, remember that one should only use ADL to provide commonly expected behavior for a certain class.
Overusing it can leads to confusing code. In C++ core guidelines, it suggests that
> T.47: Avoid highly visible unconstrained templates with common names
>
> Reason:
> An unconstrained template argument is a perfect match for anything
> so such a template can be preferred over more specific types that
> require minor conversions.
> This is particularly annoying/dangerous when ADL is used.
> Common names make this problem more likely.

With the following example:
```cpp
namespace Bad {
    struct S { int m; };
    template<typename T1, typename T2>
    bool operator==(T1, T2) { cout << "Bad\n"; return true; }
}

namespace T0 {
    bool operator==(int, Bad::S) { cout << "T0\n"; return true; }  // compare to int

    void test()
    {
        Bad::S bad{ 1 };
        vector<int> v(10);
        bool b = 1 == bad;
        bool b2 = v.size() == bad;//Caution here
    }
}
```
The function `test` prints "T0\nBad\n".
When performing comparison `v.size()==bad`, the `==` operator defined in
namespace `Bad` is preferred, because which is a perfect match, while the `==` operator
defined in namespace `T0` needs an type conversion to `int`. Thus, the former is
found by ADL and selected via function overloading rules.

