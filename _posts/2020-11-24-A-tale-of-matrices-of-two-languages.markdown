---
layout: post
mathjax: true
comments: true
title:  "A tale of matrices of two languages"
author: John Z. Li
date:   2020-11-24 19:00:18 +0800
categories: c++ programming
tags: array, matrix
---
If you are working with matrices in C,
the following code snippet must be the first thing that comes to mind:
```cpp
double (*matrix_ptr)[m][n] = malloc(sizeof(double) * m * n );
```
The code allocates space on the heap that can hold an m-times-n matrix,
and the piece of memory is referred by a pointer to the type of double[m][n].

Before C99 introduced variable length arrays into the C language,
to make the above code to work, both `m` and `n` must be compile-time constants.
This applies also to C++.
The reason is that the compiler must know the exact size of an array during compile
time to make sure that everything is consistent with the type system.
This changed with C introducing variable length arrays in C99.

From the perspective of type systems, variable length arrays are less
about allocating variable length arrays on the stack.
(Of course, one can do it, but it also adds the possibility of stack overflows If used without care)
It is more about relaxing the type system of the language, which now recognizes
some types which can only be fully specified during runtime.
The result is that `double[m][n]` is now a legitimate type with both `m` and `n`
being runtime variables.
This means, among other things, one can apply `sizeof` operator to it, leading to the following more concise code:
```cpp
    double (*matrix_ptr)[m][n] = malloc(sizeof(double[m][n]));
```
The new version is more compact, more natural to users.

The matter of matrices is another story with C++.
With C++, one can overload the subscript operator `[]`,
but it only takes one parameter.
To make it work with a syntax like `matrix[i][j]`,
one has to first let the subscript operator returns a helper object
and apply the operator again to the helper object to return a reference
to the corresponding matrix element.
One bad thing about this approach is that a simply accessing an element of a matrix
now involves two consecutive function calls.
A typical workaround is to  overload the `()` operator.
This approach works but is sub-optimal since C++ already has the operator `[]`.
The operator has been also generalized to vectors. So the square bracket operator has
a well-established meaning of accessing array elements. And a matrix is just a 2-dimensional array.
In C, one can use `a[i][j]` to access the i-j-th element of a matrix.
Two consecutive application of the square bracket operators should be interpreted
as only one operation, maybe with a simpler syntax like `a[i,j]`.

Partially because of the above mentioned reason,
The C++20 standard incorporated a proposal that
deprecates usage of comma expressions inside square brackets,
that is, `a[I, j]` with integer `i` and `j`
does not evaluate to `a[j]` anymore.
With this modification, the syntax can be reused to denote
subscript operation of multi-dimensional array-like object.
So, in  post-C++20 era,
C++ libraries will start to use the syntax of `matrix[i, j]`
to refer to an element of a matrix.
The bad thing of it is that it introduced one more context-sensitive parsing rule into C++,
while the good thing is that working with matrices in C++ will feel more natural.

