---
layout: post
mathjax: true
comments: true
title:  "The ARRAY_SIZE macro defined in the Linux kernel"
author: John Z. Li
date:   2021-01-18 19:00:18 +0800
categories: c programming
tags: type_check
---
There are two macros for type checking in the Linux kernel.
The header "typecheck.h" (in "include/linux") is quite short,
only containing two macros, which are straightforward:
{% raw %}
```c
    /*
     * Check at compile time that something is of a particular type.
     * Always evaluates to 1 so you may use it easily in comparisons.
     */
    #define typecheck(type,x) \
    ({	type __dummy; \
    	typeof(x) __dummy2; \
    	(void)(&__dummy == &__dummy2); \
    	1; \
    })

    /*
     * Check at compile time that 'function' is a certain type, or is a pointer
     * to that type (needs to use typedef for the function type.)
     */
    #define typecheck_fn(type,function) \
    ({	typeof(type) __tmp = function; \
    	(void)__tmp; \
    })
```
{% endraw %}
As the comments say, the two macros are intended to check if
a variable or function is of the desired types,
both being dependent on GCC's `typeof` extension.
Usually, if a variable is defined but not used,
the compiler will complain about it.
By explicitly converting it to the type of `void`, according to the standard,
the value of such a variable will just be silently discarded without issuing a warning.

In the first example, comparing two pointers of incompatible
types leads to a compiler warning.
In the second example, trying to assign a function (pointer)
to another function pointer of an incompatible type will make the compiler complain.
Maybe a better signature for the macro could be
```c
    typecheck_fn(fun_ptr_type, fun)
```
One interesting is, when I searched the source tree with
```bash
 ag -w "typecheck_fn" path_to_source_tree_root
```
I found nothing except the macro definition. It seems that the second macro has not been put into use.

