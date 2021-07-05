---
layout: post
mathjax: true
comments: true
title:  "How to write a main function that does not return"
author: John Z. Li
date:   2021-01-22 19:00:18 +0800
categories: c programming
tags: main_function
---
This is previously an [answer](https://qr.ae/pG41kq) written by me to a question on Quora.
The question asked (my rephrased version) is that
> If a main function is supposed to never return, how should it be implemented?

The main function is just a function.
Like other functions, a return statement returns control
from within the function to the statement that the function is called from.
If a function doesn't return a value,
the return type is given as `void`.
In C, all functions except functions with
variable argument list have their corresponding prototypes.
A function prototype consists of a parameter type list and a return type.
That is why the type of `void` exists.
The main function is special in the sense that
its prototype(s) is enforced by the language standard.
The programmer is not supposed to alter its prototype.
The programmer only provide his own implementation of the main function.
The language standard guarantees that the following
two prototypes of the main function are always accessible,
```c
    int main(void);
    int main(int argc, char * argv[]);
```
On some embedded platforms, where the main function is
never expected to return,
the compiler often provides an extra prototype as
```c
    void main(void) ;
```
to indicate that the program is not supposed to return.
In other words, if the main function returns, it implies a fatal error.

I personally think this language extension is inconsistent.
After all, **a function returning `void` has nothing to do with
the programmer's intention that this function SHOULD NOT return in normal circumstances**.

If you really want to have a way to suggest that
a main function should not return,
my suggestion is omitting the return statement.
The language standard says that reaching the end of
the main function is equivalent to a return statement with `EXIT_SUCCESS`.
If you write your program correctly,
the main function will never return as intended.
If someone else reads your code,
he will notice that there is not a return statement in the main function,
and figure out that it will never return.
Perhaps a bit of commenting will make it even more obvious like below ,
```c
    int main(void){
        // ...
        /* return; this function never returns. */
    }
```
