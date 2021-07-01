---
layout: post
mathjax: true
comments: true
title:  "Some compiler options that disable UB-based behavior"
author: John Z. Li
date:   2020-11-20 19:00:18 +0800
categories: c++ programming
tags: undefined-behavior, optimization
---
C and C++ have many things left as undefined behavior in their language standard.
For example, the following simple function that tries to negate a signed integer
actually involves undefined behaviour:
```cpp
int negate(int a){
    return -a;
}
```
When the input to the functions happens to be `INT_MIN`, overflow occurs, that is
undefined behaviour in C/C++. Likewise, the following seemingly innocent function
also introduces undefined behaviour:
```cpp
int div(int dividend, int divisor){
    return dividend/divisor;
}
```
When `dividend` being `INT_MIN`, and `divisor` being `-1`, signed integer overflow happens.

This is kind of annoying, but it is not immediately problematic.
Careful C/C++ programmers either make sure that overflow doesn't happen,
or explicitly handles it when it might happen.
The real challenge is, modern compilers perform optimizations based on
the assumption that there is no undefined behavior in source code.
For example, `i +1 < i` where `i` is an `int` type will be optimized to `false`
in compile-time. Likewise, `i+1 < 1` will be optimized to `i < 0`,
and `2*x/2` will be optimized to `x`, and the following loop
```cpp
    for (i = 0; i <= N; ++i) { ... }// i is a signed integer
```
will be optimized based on the assumption that the loop will be executed exactly N+1 times.
This kind of optimization is generally not what programmers want. If a programmer
wants to check whether `i+1 < i` for integer `i`,
it means that he believes it's possible
that the condition might be false.
Maybe he is using this to detect whether integer overflow has happened,
and if it does, he will handle it explicitly.
A useful compiler option of GCC is `-fno-strict-overflow`.
This option disables optimization based on
undefined behavior of integer overflow.
Furthermore, if one wants integer overflow to have
deterministic behavior in case of overflow, he can use `-fwrapv` option.
The latter entails the former, and it tells the compiler to generate code
so that if integer overflow happens,
the result should be wrapping 2's complement to the bit width of the type.
For example, `INT_MAX +1` will evaluated to `INT_MIN`,
just like the unsigned case.

Another optimization based on undefined behavior that C/C++ compilers
might perform is optimization based on there is no null pointer dereferencing,
because dereferencing a null pointer is undefined behavior.
Consider the following example, taken from [What Every C Programmer Should Know About Undefined Behavior](http://blog.llvm.org/2011/05/what-every-c-programmer-should-know.html)
with a bit variation,
```c
void fun(int* P){
    int dead = *P;

	if (P == 0)
        return;

    *P = 4;
}
```
If the compiler reasons that since pointer `p` is dereferenced before the `if` clause,
`p` can not be a null pointer, and it transforms the code snippet into the following
"equivalent" form:
```c
void fun(int* p){
    int dead = *p;

    if (false)

        return;

    *p = 4;
}
```
And then the compiler figures out that except the last line in the function body, all
lines of the code in that scope is dead code. So it performs dead code elimination
and transforms the code snippet into the following "equivalent" code:
```c
void fun(int* p){
    *p = 4;
}
```
The programmer may falsely believes that since he performed null checking in the
function body explicitly, so the function must handle the case correctly when a
null pointer is passed to the function. When things doesn't work as expected, he
will have a difficult time finding out the root cause of malfunction of his program.

IMHO, this kind of optimization should not happen in the first place.
If the programmer explicitly checked whether a pointer is null,
compiler should trust him on this.
Compilers trying to outsmart programmers is not a good idea.
If, otherwise, in some cases where a pointer is clearly not null,
the programmer should be clever enough to figure that out
and doesn't write the null checking code in the first place.

Fortunately, this optimization can be disabled with `-fno-delete-null-pointer-checks` with GCC.
