---
layout: post
mathjax: true
comments: true
title:  "A minimalistic implementation of a circular buffer "
author: John Z. Li
date:   2020-10-26 19:00:18 +0800
categories:  c programming
tags: circular-buffer
---
A circular buffer is semantically equivalent with a fixed-length
producer-consumer queue.
Here we only consider the single-producer and single-consumer ones.
It is trivial to implement a producer-consumer queue if a lock
(with or without a condition variable) is used.
But locks are two heavy weight in some cases and there are
lock-free implementations of producer-consumer queues.
The main difference between a general producer-consumer queue and a fixed-length one
is that the former can grow to any size.
However, the sole purpose of a lock-free producer-consumer queue
is to make data transfer between threads quick.
If a queue grows infinitely,
it means that the speed of the consumer thread processing data is
slower than the speed of the producer thread feeding new data.
If it is so, the system is doomed anyway.
To make a producer-consumer model really work,
the designer has to guarantee that the speed of the producer is equal to or
less than the speed of the consumer averagely in the long run.
With this in mind, we can see that a fixed-length producer-consumer queue is,
in many cases, what one really wants, that is, a circular buffer.
With a large enough buffer,
the probability that the producer will be blocked is approaching to zero,
since, the producer only gets blocked when the buffer is full.

In the following, the author will implement a minimalistic circular buffer,
which is lock-free when the buffer is neither full or empty.

First, we define a global buffer with fixed length.
Making the buffer global saves us from dealing with lifetime management of the buffer,
while the cost is neglectable, that is,
the buffer can not grow on demand.
The buffer will be initialized when the program starts and destroys when the program finishes. It is simply by
```c
    #define BUFFER_SIZE 10
    char buffer[BUFFER_SIZE];
```
The buffer is a char array, so each object to be transferred is supposed to
convert to a char array by type-punning and converted back on the consumer’s side.
The reason for this is, beside keeping the interface clean,
that for small trivially copiable objects,  copying them around is much faster
than passing  pointers to the to-be-transferred objects.
On the other hand, if the to-be-transferred data is copy-expensive,
one can always resort to passing a `void*` pointers instead, by type-punning it to a char array, of course.

Now comes to the circular buffer, that is, we must iterate over indexes of the array. We introduce the following macros:
```c
    #define INCR(x) (((x)+1)%BUFFER_SIZE)
    #define DECR(x) ((((x)+BUFFER_SIZE)-1)%BUFFER_SIZE)
```
This is straightforward. The macro `INCR` increases the index of the array
and if it is not currently the array’s largest index and jumps to the beginning of the array.
Likewise, the macro `DECR` works similarly, only it iterates over the array backwardly.

Since we don’t want to use locks, some memory barriers must be used instead.
The Visual C++ compiler has this nice extension of the volatile qualifiers in C/C++
so that it acts like their counterparts in Java or C#, namely, it means

1. a write to a volatile object has Release semantics;

2. a read of a volatile object has Acquire semantics.

This is neat, so we don’t have to use language-specific memory barriers
with C and C++. (Remember to compile the program with the `/volatile:ms`
command-line option if you are not working with x86/64 CPUs,
with x86/64 CPUs, it is the default.) We introduce the following two volatile integers:
```c
    volatile int start_idx = 0;
    volatile int end_idx = 0;
```
The `start_idx` is the array index where stored data starts (inclusive),
and the end_idx is the array index where the stored data ends (exclusive).
If these two collides, it means the buffer is empty.
A consequence of this is that the buffer can hold at most `BUFFER_SIZE -1` bytes.

Now we define the user interface, simply by
```c
    void write_buffer(char* data, int len);
    void read_buffer(char* data, int len);
```
The signature of the interface speaks for itself. Recall that the user is supposed to cast between char arrays and objects using type-punning.

Notice that if we have a help function for function `write_buffer()`,
that returns the number of bytes it fills into the buffer at one call,
we can continually call this help function until all the data is written into the buffer.
Since the buffer is seldom full, this busy-waiting strategy is not costly as it looks.
Suppose that a helper function is declared as below:
```c
int write_buffer_help(char* data, int len);
```
Function `write_buffer()` can be simply implemented as
```c
    void write_buffer(char* data, int len) {
      int i = 0, sent = 0;
      do {
      i = write_buffer_help(&data[sent], len - sent);
      sent += i;
      } while (sent != len);
      return;
    }
```
Similarly, with a help function for `read_buffer()`, which is named `read_buffer_help()`,
the read_buffer function can be simply by
```c
    void read_buffer(char* data, int len)
    {
      int i = 0, received = 0;
      do
      {
      i = read_buffer_help(&data[received], len - received);
      received += i;
      } while (received != len);
      return;
    }
```
Now comes to the tricky part of implementing the two helper functions.
The key is to notice that function `read_buffer` only write to `start_idx`,
and function `write_buffer` only write to `end_idx`,
while they both read from `start_idx` and `end_idx`.
Carefully arranging memory loads and writes such that,
from the view point of another thread, the history of the buffer is consistent. Implementation is as below:
```c
     int write_buffer_help(char* data, int len)
    {
      int temp_start_idx, temp_end_idx, j;
      for (j = 0;
      temp_start_idx = start_idx, //acquire read
      temp_end_idx = end_idx, //acquire read
      temp_end_idx != DECR(temp_start_idx) && j < len;
      ++j)
      {
      buffer[temp_end_idx] = data[j];
      end_idx = INCR(temp_end_idx); //release write
      }
      return j;
    }
    int read_buffer_help(char* data, int len)
    {//read untill data is filled
      int temp_end_idx, temp_start_idx, j;
      for (j = 0;
      temp_end_idx = end_idx, //acquire read
      temp_start_idx = start_idx, //acquire read
      temp_start_idx != temp_end_idx && j < len;
      ++j
      )
      {
      data[j] = buffer[temp_start_idx];
      start_idx = INCR(temp_start_idx); //release write
      }
      return j;
    }
```
Note1: If the buffer size is 256, the `INCR` and `DECR` macro can be simply
implemented using the unitary `++` and `–-` applied to a `uint8_t`
type of variable, leading to slight efficiency improvement. Likewise, with buffer size of 2 to power 16, `uint16_t` can be used.

Note2: The busy waiting in read_buffer is wasteful in some cases.
This can be mitigated by yield the current thread after several times of trying by
calling `pthread_yield()` in C or `std::this_thread::yield()` in C++.
Compared with using condition variables, some CPUs cycles will be wasted, but with less delay.

