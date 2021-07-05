---
layout: post
mathjax: true
comments: true
title:  "An assert replacement for embedded programming"
author: John Z. Li
date:   2020-12-12 19:00:18 +0800
categories: c programming
tags: inline
---
The worst thing about the `inline` keyword in the C programming language is its name,
because it has little to do with the compilation optimization technique called inclining,
that is in-place code substitution of function calls.
Modern compilers  inline function calls as they see proper for
performance reasons, with or without the `inline` keyword.
What the keyword really means is as below:

- It loosens the One Definition Rule (ODR) of the language.
ODR means, any entity, being it a variable or a function, can only be defined once.
If a function is specified with `inline`,
the restriction is relaxed for it.
Now it can be defined in multiple translation unit, with the requirement that
- all definitions of the same inline function are identical.
The language standard spent quite some words to specify what it means
that two functions are considered identical.
It is the compiler’s responsibility to ensure that this identical requirement is satisfied.
- Furthermore, if a function specified as `inline` is not inlined during compilation,
it is just a normal function. If, otherwise, it gets inlined,
for an external observer (user of compiled binary),
he does not have to care about whether it is inlined or not.
In other words, he can treat the function just as a normal function.
This means, a symbol of the function is exported,
the address of the function can be taken and passed around,
or anything one can do with normal functions
(remember that inline functions have external linkage).
- If an inline function is not inlined, there is no binary bloating.
It costs as much as a normal function in terms of function call overhead.

If inline functions are used correctly, the programmer have all the above guarantees.
However, if they are marked as `static inline` instead, we have problems:

- First, the `static` keyword nullifies the semantic meaning of the `inline` keyword.
A function is marked with `static inline` are just static functions,
which means, whether it gets inlined,
each translation unit including the definition of the function gets a binary bloat.
- More importantly, there are no guarantee that each copy of the function
in different translation units is identical with each other.
It is possible that some macro definitions or some `static const` variables
have made the same inline function in different translation units behave differently.
But the programmer may be left with the false assumption that they are identical.
Imagine that you have to debug a program that is broken because the
identical requirements are silently violated.
- Since the function is marked static,
no symbol will be exported in the binary.
External users of the binary are deprived of the option
to use the function as a normal function.

To sum up, do not mark your function as `static inline`.
If you mean static, use static. If you mean inline, use it properly.

