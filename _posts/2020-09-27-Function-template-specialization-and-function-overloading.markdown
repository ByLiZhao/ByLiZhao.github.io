---
layout: post
mathjax: true
comments: true
title:  "Function template specialization and function overloading"
author: John Z. Li
date:   2020-09-27 19:00:18 +0800
categories: c++ programming
tags: template-specialization, function-overloading
---
In this post, I will use the term “template function”
to mean a base function template.
The intention is to emphasize the fact that function templates
act more like functions (less like templates as in class templates).

Since functions in C++ can be overloaded.
The interplay between function overloading and template function specialization
is somehow complicated,
mainly because that template functions are also functions and can be overloaded,
either by plain and old functions or by other template functions.
Rules can be summarized as below:

1. All functions are equal, but plain and old (POD) (concrete) functions are more equal.
Whenever there is a concrete function that matches types of a calling function’s argument list,
it is selected regardless of possible existence of candidates of template functions.
2. Template functions are only considered when so matching POD functions exist.
3. Template functions can’t be partially specialized (as class templates can be).
Trying to do so leads to an overloading template function. For example, the following code
```cpp
    template<class T>
    void f( T );

    template<class T>
    void f( T* );
```
creates two overloaded template functions.
Each of them is a template function in itself, and
the second is not a partial specialization of the first one.

Template functions can only be fully specialized (also known as explicit specialization)
Explicitly specialized functions are third class citizens.
(POD functions are first class citizens, and template functions are second class citizens).
During overloading resolution, when there is no first class citizens,
second class citizens are considered.
Each overloading template functions are compared against each other on its own merits.
During this phase, whether a template function is explicitly specialized does not matter.
Only when a certain template function is selected,
the compiler will check whether there is a better matching exists
as a fully specialized template function.
If Yes, that explicitly specialized one is selected.
If No, instantiate the template function with given types.

Two takeaways are:
1. If you want to make sure a specific implementation for a concrete type (or concrete types) is definitely selected, make it a POD function.
2. If you miss the full-fledged power with partial template specialization, wrap the function in a class template as a static member function.

How compilers choose between overloaded
template functions is not worth knowing,
if you are not a compiler author or a language lawyer.
**Just don’t overload template function with another template function.**
Things can be messy if you do, as the following well-known example shows:
```cpp
    template<class T> //  1
    void f( T );

    template<class T> // 2
    void f( T* );

    template<>
    void f<>(int*); // 3, explicit specialization of 2.

    int *p;
    f(p);  // the compiler first choose 2 during overloading resolution of template functions
    	// then choose 3 because it is better match.

    //in another file

    template<class T> // 4
    void f( T );

    template<>        // 5, explicit specialization of 4
    void f<>(int*);

    template<class T> // 6
    void f( T* );

    int *p;
    f( p );   // the compiler finds 6 is a better match, and instantiate it with int.
```
