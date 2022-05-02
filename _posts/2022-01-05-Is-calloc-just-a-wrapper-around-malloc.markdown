---
layout: post
mathjax: true
comments: true
title:  "Is calloc just a wrapper around malloc?"
author: John Z. Li
date:   2022-01-05 21:00:18 +0800
categories: c
tags: malloc, calloc, memory-allocation
---
There are two functions in C to allocate memory on the heap. One is `maloc`, which has the following signature:

```
void* malloc (size_t size);
```

It takes the size of the memory block that the user wants to allocate as a parameter, and return a `voud` pointer to the allocated memory block on success, otherwise a null pointer will be returned. The other heap allocation function in C is `calloc`, which has the following signature:

```
void* calloc (size_t num, size_t size);
```

`calloc` receives two parameters. On success, it will return a `void * ` pointer to a memory block which has the size of `num*size`. Furthermore, each bits of the memory block is set to zero (each byte is set to `\0`).

It is tempted to think that `calloc` is just a convenient wrapper of `malloc` by calling `malloc` with parameter `num*size` followed a call of `memset` setting each byte to `\0`. In other words, it might be assumed that `calloc` is implemented as below:
```c
void* calloc (size_t num, size_t size){
	size_t n = num*size;
	void* p = malloc(n);
	if(p)
	memset(p, '\0', n);
	return p;
}
```
Actually, many online tutorials teaches that this is how `calloc` works.

However, the answer to the above question is No. There are two reasons:
- First, any implementation worth its salt should check whether `num*sise` leads to integer overflow. Surprisingly, many implementations don't (or didn't) correctly handle this, for example [both glibc and jemalloc used to have security vulnerabilities resulted from not correctly handling the case of integer overflow inside calloc](https://kqueue.org/blog/2012/03/05/memory-allocator-security-revisited/). Those bugs have been identified and fixed. It is fair to say that one should generally prefer `calloc`over `malloc` even for this reason only for programmers don't always remember correctly handling integer overflow.

- Secondly, using `memset` along with `malloc` might be bad in terms of performance. The reason is that for a modern operating system, it has to maintain a list of zeroized pages usually for security reasons, so that a process can't read the content from the memory space of another process. Failing to do so  can lead to severe security problems because there might be sensitive data on some pages that should be kept away from other processes. For this reason, a clever implementation of `calloc` doesn't have to call `memset`. Instead, it can just ask the OS to allocate the required memory block on a already zeroized pages. This is essentially free lunch compared with `malloc` followed by a `memset`.
