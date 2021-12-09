---
layout: post
mathjax: true
comments: true
title:  "std::launder, the most dreaded feature in C++?"
author: John Z. Li
date:   2021-11-21 21:00:18 +0800
categories: c++
tags: launder
---
C++17 introduced a small feature called "std::launder" in header `<new>`.
What is this little weird-named function as below supposed to achieve?
```cpp
template<typename T>
constexpr T* launder(T* p) noexcept;
```
Given that this function just takes a pointer to type `T` and returns that pointer,
its usefulness is not clear at first glance. (It looks suspiciously like a `nop`, but it is really not.)
We will examine its semantics through examples below.
## The example given by the authors of the proposal that added `std::launder` to C++17.
You can find the proposal [here](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2016/p0137r1.html)
if you are interested.
```cpp
struct X { const int n; };
union U { X x; float f; };
void tong() {
  U u = {{ 1 }};
  u.f = 5.f;                          // OK, creates new subobject of 'u' (9.5)
  X *p = new (&u.x) X {2};            // OK, creates new subobject of 'u'
  assert(p->n == 2);                  // OK
  assert(*std::launder(&u.x.n) == 2); // OK
  assert(u.x.n == 2);                 // undefined behavior, 'u.x' does not name new subobject
}
```
Ignoring the very-hard-to-parse wording in the proposal, let us see what happens in the example:
1. First, `struct X` is defined that it contains a single `const int` member called `n`. The fact
that `n` is `const` means that `n` is initialized with its value at object construction, and its
value is guaranteed not to change during the lifetime of the object.
2. Union `U` has two members, one is an object of type `X`, and the other is just a float. Because
how `X` is defined, when an object of type `U` is active with a member of type `X`, the chunk of memory
that contains `n` should be altered during the lifetime of the union member of type `X`.
3. In function `tong`, `u` of type `U` is constructed with the union member being an object of type `X`,
which contains a `const` number `const n = 1`.
4. By executing the line `u.f = 5.f`, the lifetime of the object of `x` ends, and `f` becomes the active
member of the union. **Note: there is no constness violation here. The value of `const n` is not changed
during the lifetime of `n`. This means the value of `x` is not changed while `x` is the active member of the union `u`.**
5. In the next line, by calling `X* p = new(&u.x)X{2}`, `x` again becomes the active member of the union `u`. With
an placement `new`, a new object of type `X` is created and it starts its lifetime after operator `new` returns.
**Note: no constness violation happens because `x` now denotes a different object of type `X`. The lifetimes of
the two objects of type `X` don't overlap.**
6. Obviously, the assertion `p->n == 2` should hold, for data member pointer `p` points to an object of type `X`, which
is explicitly initialized with its member `const n`  has value of `2`.
7. But if we access `n` through the union, we have **Undefined Behavior** here, like in the assertion `u.x.n == 2`.
Wait, what has just happened?

What happens is that when the compiler sees `U u = {{1}}`, after compilation, the expression is reduced to something
as if it is `const u = 1`, which is a so-called Integer Constant Expression (ICE). Basically, this means that the compiler
is allowed to assume the value of `u.x.n` will never change afterwards. So, the compiler might perform optimization by
substituting every occurrence of `u.x.n` with `1`. This is called const-propagation. If this optimization takes place,
it is legitimate for the comparison `u.x.n == 2` is valuated as `false`. But the compiler can also compile `u.x.n` as
a memory loading of a lvalue. In this case, the comparison `u.x.n == 1` will be evaluated as false. Thus UB is guaranteed.

What `std::launder` does is basically telling the compiler that
> Do not assume anything about the value pointed by the pointer, which is the function parameter of std::launder.
> The value pointed by the pointer might have been updated. So, do not perform optimization based on the assumption it has not.

We can think of `std::launder` as a function that is used to suppress compiler optimization regarding `const` members of classes/structs.

## Another example given by an author of Clang
```cpp
struct A {
  virtual int f();
};

struct B : A {
  virtual int f() { new (this) A; return 1; }
};

int A::f() { new (this) B; return 2; }

int h() {
  A a;
  int n = a.f();
  int m = std::launder(&a)->f();
  return n + m;
  // A compiler correctly implements std::launder should return 3.
}
```
Remember that each object of a virtual class (a class with one or more virtual member functions)
contains a `const` point often called the `vptr` which points to the `vtable` of that class.
Though this `vptr` is not directly accessible to programmers, it exists in a way as if it is
just a `const` member of the class which is a pointer to an array of function pointers.

Since the compiler knows that `a` is really an instance of `A`, and `&a->f` is just a function call
from a pointer of type `A*` to an object of type `A`, the compiler might eliminate the virtual function
call by replacing it with a direct call of the member function `f` of `A`. This is also known as
"devirtualization". But inside the function `f`, the `vptr` is changed using placement `new`. This is
somehow like the previous example. Something that is supposed to be `const` is changed unexpectedly.
If one does not explicitly disable optimization based on false premise with `std::launder`, the compiler
may or may not perform the said optimization and leads to UB.
To avoid UB, the second call of function `f` has to be done via `std::launder`.

To sum up, placement `new` does not play well with `const` members of classes and unions.
