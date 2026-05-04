---
layout: post
mathjax: true
comments: true
title:  "Forcing memset to be executed against compiler optimization"
author: John Z. Li
date:   2020-08-15 19:00:00 +0800
categories: c programming
tags:  memset, compiler-optimization
---

With following code

```c
    char a[10];
    scanf("%9s", a);
    printf("%s\n", a); // compiled to calling "puts" by the compilers
    memset(a, 0, sizeof a); // REMOVED by the compilers
```
if turning on compiler optimization with `-O2`,
It is very likely that the `memset` function call will be removed.

This is due to an compiler optimization called dead code removal.
If a compiler sees that the char array is not going to be used in following code
(like being written in, or being printed to the screen, or being sent via socket, etc.),
it deduces that the `memset` function call is unnecessary, thus removes it from
generated object file.

If the char array is reused in following code for other purposes, but the compiler again can prove
that its content will soon be overwritten, the compiler might also conclude that the `memset`
is unnecessary, for if the content of the array will be overwritten anyway, whey bother
firstly `memset`ing its elements to zeros, only to be overwritten next.

Problem is, this optimization is sometimes not what a programmer would expect.
For example, if the `memset`
function is called in a security sensitive context to zeroize
sensitive information like user passwords,
it is undesirable for it to be optimized away.

To force `memset` actually to be executed,
we can define the following `memset` replacement:

```c
    void memset_s(volatile char * p, int value, size_t num){
    	int i;
    	for(i=0; i<num; ++i)
    		{*(p+i)=value;
    	}
    }
```

Because the function `memset_s` takes a pointer to `volatile char`
as its first parameter, writing to the pointed char array cannot be optimized away.

Actually, C11 introduced a function `memset_s` with the signature
```c
     errno_t memset_s( void *dest, rsize_t destsz, int ch, rsize_t count );
```
The C11 standard mandates that this function won’t be optimized away by the compiler.
