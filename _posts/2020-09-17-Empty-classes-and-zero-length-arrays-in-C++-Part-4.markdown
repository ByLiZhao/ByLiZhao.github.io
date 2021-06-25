---
layout: post
mathjax: true
comments: true
title:  "Empty classes and zero length arrays in C++: Part 4"
author: John Z. Li
date:   2020-09-17 19:00:18 +0800
categories: c++ programming
tags: empty-classes, tag-dispatching
---
This is post is about using empty classes to do tag dispatching in generic programming.

Tag dispatching is a technique to do compile-time polymorphism,
that is, dispatch function call to different versions through a unified interface.
Empty classes are used to disambiguate overloaded function calls in compile-time.
This is can be done because empty classes defined separately with different names
are different types, and they take part in function overloading resolution.
For example, in the following code snippet, function `fun(A)` and `fun(B)`
are two different overloaded functions, that is, `fun(A{})` will call the first version
while `fun(B{})` will call the second version.
```cpp
    struct A{};
    struct B{};
    void fun(A);
    void fun(B);
```
This alone is interesting, but not particularly useful, since disambiguating
overloaded functions by introducing an extra parameter is trivial work.
But it becomes very useful when combined with type traits.
By extracting the type information of tag objects via type traits,
the dispatching process can be delegated to compilers to do in compile-time.
Users only need to call the function as if it is a normal function without the extra empty-class-argument,
and the correct version of that function will be automatically called by compilers.
Thus, implementation details are hidden behind a unified interface, that is the whole point of polymorphism.
Since compile-time polymorphism has the advantage being more efficient than run-time polymorphism using virtual functions,
when the former is enough for our use,
we should always prefer the former.
An well-known example of tag dispatch is the `std::advance` function.
The signature of the advance function is like below:
```cpp
    template <class InputIterator, class Distance>
      void advance(InputIterator& i, Distance n)
```
where `i` is an object of type `InputIterator`, and `n` is the number of steps to advance.

Different input iterator classes have different characteristics.
Some input iterator classes have the characteristic that their underlying index of the object can be
randomly accessed, while other input iterator classes only allow sequential access.
Among sequential-access input iterators, there are bi-directional types and unidirectional types.
For the former, it makes sense that distance `n` is a negative number,
meaning step backward, while for the latter, distance `n`
being negative would be a fatal error.

By calling `std:: iterator_traits<InputIterator>::iterator_category`,
an empty class denoting the corresponding iterator characteristics
is resolved in compile-time, that is, one of the following will be returned.
```cpp
      struct input_iterator_tag { };
      struct bidirectional_iterator_tag { };
      struct random_access_iterator_tag { };
```
Now the problem is reduced to providing implementation separately
for each of the below.
(they are all templated functions, defined in a sub-namespace called `detail`.
We made some simplification here for clarity.)
```cpp
 void advance_dispatch(InputIterator& i, Distance n, input_iterator_tag) {

 void advance_dispatch(BidirectionalIterator& i, Distance n, bidirectional_iterator_tag)

 void advance_dispatch(RandomAccessIterator& i, Distance, random_access_iterator_tag)
```
And the actual `std::advance` function can be simply implemented by delegating
function calls according to the category of input iterators to its correct implementation like below,
```cpp
    template <class InputIterator, class Distance>
      void advance(InputIterator& i, Distance n) {
      typename iterator_traits<InputIterator>::iterator_category category;
      detail::advance_dispatch(i, n, category);
      }
    }
````
