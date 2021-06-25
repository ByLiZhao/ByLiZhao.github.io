---
layout: post
mathjax: true
comments: true
title:  "Flexible array member in C"
author: John Z. Li
date:   2020-09-17 19:00:18 +0800
categories: c programming
tags: flexible-array-member
---
This post is about flexible array member, which is a C99 feature ( it is  not part of C++).

A flexible array is a member of a `struct` of type array. An example is as below:
```c
    struct message{
    	uint8_t size;
    	float data[]; //leave square bracket empty means it is a flexible array
    };
```

With GCC, it is also possible to introduce a flexible array member with the following
syntax:
```c
    struct message{
    	uint8_t size;
    	float data[0]; //with a zero-length array
    };
```

A flexible array must appear as the last element of a `struct`,
and it can not be the only member of a `struct`, otherwise, it is  undefined behavior.

This seems a trivial feature, but it can be handy sometimes.

Say, a little program receives small-sized messages via a TCP connection.
Each message is sent in the order of one leading byte
(denoting the message size)
followed by a sequence of bytes of the specified size, as the message body.
If we can know in advance that received messages are of small size,
for example, usually only one (4 bytes) or two floats (8 bytes), and
never more than 100 floats (400 bytes).
We can  allocate  stack memory for an object of type `message`,
do something with it (for example, validate the message and
send those floats to a robot controller).
Afterwards, function stack unwinding will clear memory usage,
Since stack allocation and unwinding is really fast
(as cheap as increasing or decreasing the stack pointer), we can get really fast with
this. Modern systems have large stack size, for example, 64-bit Linux has a default
size of 8M stack for each thread. Moving small sized objects from heap to stack can
be a performance boost without worrying about stack overflow.
