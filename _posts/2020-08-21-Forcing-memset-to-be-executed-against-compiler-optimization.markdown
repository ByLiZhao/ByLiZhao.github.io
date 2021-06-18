---
layout: post
mathjax: true
comments: true
title:  "Forcing memset to be executed-against-compiler-optimization"
author: John Z. Li
date:   2020-08-15 19:00:00 +0800
categories: c programming
tags:  memset, compiler-optimization
---

With following code

```c
    char a[10];
    scanf("%9s", a);
    printf("%s\n", a); // changed to puts by the compilers
    memset(a, 0, sizeof a); // REMOVED by the compilers
```
if turning on compiler optimization with `-O2`,
It is very likely that the `memset` function call will be removed.

I believe this is due to an optimization technique called
dead code removal.
If a compiler sees that the char array is not going to be used in following code
(like to be written in, be printed to the screen, or be sent via socket),
it feels that the `memset` function call is unnecessary, thus making removes it from
the compiled object file.
If the vector array is reused for other purposes, and the compiler again can prove
that its content is overwritten, the compiler might also conclude that the `memset`
is unnecessary, for if the content of the array will be overwritten anyway, whey bother
firstly zeroing all its elements, thus again optimize away the `memset`.

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
as its first parameter, writing to the pointed char cannot be optimized away.

Actually, C11 introduced a function `memset_s` with the signature
```c
     errno_t memset_s( void *dest, rsize_t destsz, int ch, rsize_t count );
```
The C11 standard mandates that this function won’t be optimized away by the compiler.
