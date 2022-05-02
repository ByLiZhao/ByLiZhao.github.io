---
layout: post
mathjax: true
comments: true
title:  "Lifetime implication of make_shared"
author: John Z. Li
date:   2022-01-10 21:00:18 +0800
categories: C++
tags: shared_pointer object_lifetime, make_shared
---
Function template `make_shared` (and its cousin `make_unique` if you care) was
introduced in C++14 to mainly address a problem related with exception safety.
Here is the story, consider the following code:
```c++
fun(std::shared_ptr(new std::string("foo")),
    std::shared_ptr<Rhs>(new std::string("bar")));
```
Before C++17, the order in which function parameters are evaluated is not
specified. So, a compiler can transform the above function call to the below
equivalent code:
```c++
auto p1 = new std::string("some long string literal long enough to defy SSO");
auto p2 = new std::string("some other string literal long enough to defy SSO ");
auto sp1 = std::shared_pointer(p1);
auto sp2 = std::shared_pointer(p2);
fun(sp1, sp2);
```
The problem with this is that if an exception is thrown during construction of
the string pointed by `p2`, string pointed by `p1` will become leaked memory.
Function template `make_shared` was introduced in C++14 to fix this problem.
The idea is that when allocation happens inside function  `make_shared`,
any exception thrown inside the function will cause stack unwinding of the
function thus destroy any temporary objects
created inside the function. In a word, using `maked_shared` leads
to exception safe code.

However, this has stopped being true since C++17. Starting from C++17,
the language standard requires that (emphasis mine),
> In a function call, value computations and side effects of the initialization
> of every parameter are **indeterminately sequenced** with respect to value
> computations and side effects of any other parameter.

Here **indeterminately sequenced** means that the function parameters can be
evaluated in any order but their execution **must not** overlap. With the new rule,
the exception safety merit of `make_shared` is no longer true.
It is another thing about it we want to talk about in this post.

While the standard does not require it, most implementations of `make_shared`
employs an optimization trick to reduce the number of allocations needed.
Recall that the following code involves two memory allocation,
one allocation for the string, one allocation for the control block of
the shared pointer:
```cpp
auto sp = std::shared_pointer(
    new std::string("a long enough string that defeats Small String Optimization");
```

Typical implementations of `shared_pointer` employs an optimization trick that merges
the two allocations involved into one, meaning that the control block of a shared
pointer and the memory block that contains the managed object reside in the same
contiguous memory chuck, which is the result of a **single memory allocation**.
This, though,  has implications for lifetime of the managed object.

The formal semantics of shared pointers requires that the lifetime of an object
managed by a shred pointer ends when all the copies of the shared pointer expire.
This means, weak pointers that refer to the same managed object don't affect the
lifetime of the managed object. Even if the "weak count" of the object is non-zero,
the object gets destroyed the moment when the "shared count" of the object hits zero.

However, with the optimization `make_shared` implementations typically employ, the lifetime guarantee
of objects managed by shared pointers is lost. If a shared pointer is created by
`make_shared`, a weak pointer pointing to the object will keep the object alive regardless whether the shared count of the object has hit zero. Hence, the following rules about `make_shared`:

- If the use pattern of a shared pointer is that no weak pointer is going to be used, it is OK to use `make_shared`.
- If weak pointers are needed, but the following is satisfied, it is OK to used `make_shared`:
    - all weak pointers eventually expire,
    - and delayed call of the managed object's destructor is acceptable in terms of
        both memory consumption and side effects.

If on the other hand, it is important to make sure that the destructor of the managed object is called exactly after the last shared pointer pointing to it expires, it is
better to use constructors of `shared_pointer` instead. It is little bit slower but has
the correct object lifetime guarantee. This can happen if the correctness of program
relies on some side-effects of the managed object's destructor
taking place  in exact timings  and in well-defined order.
