---
layout: post
mathjax: true
comments: true
title:  "Maybe you do not need a lock-free queue"
author: John Z. Li
date:   2023-04-19 21:00:18 +0800
categories: Concurrency
tags: lock-free, multi-threading, low-latency
---
It is said that if someone has a hammer, he will view every problem he encounters
as a nail. For low-latency multithreading programming, lock-free queues are sometimes
are such a hammer to programmers. The problem is about passing messages between
different OS threads with minimal latency impact. Lock-free queues seem a natural
choice because that the producer thread and the consumer thread (Let us focus on
the simplest case that there is only one producer and only one consumer) do not
have to wait on each other: the producer thread can just keep pushing messages to
a lock-free queue and the consumer thread just keeps fetching messages (removing)
messages from it. While both threads run in their corresponding big loop, no
synchronization neeeds ever to happen during the process. The mental model is
somehow like below:
```txt
|Producer Thread|-->>Lock-free Queue-->|Consumer Thread|
```

The problem here: Where or how does the producer thread gets information that is
needed to generate messages to be consumed by the consumer thread. Since the
producer thread needs to get its information from somewhere, often times, from
some external source via network, the data rate of the information source actually
determins how fast the whole system can go. If there is no data coming in, both
threads will wait for new data to come in. Let us further assume that information
that comes into the producer thread arrives in the form of messages naturally delimited
between each other. When a new message hits the producer thread, the producer thread
will start to process it while the consumer thread will wait. When the producer
thread finishes processing the incoming message, it will generate a new message
and pass it to the consumer thread. On receiving the message, the consumer thread
will start to process it. At the same time, the producer thread will start to
process new messages if there is any. Let us say that the time needed for the
producer thread to retrieve and process a message is T1, and the time needed for
the consumer thread to consume a message is T2. There are 2 cases in general:
1. T1 is bigger than T2, in this case, using a lock-free queue is generally beneficial. Because
the more we keep the consumer thread busy, the overall latency of the procedure
will be lower. This means, we need to make the consumer thread wait less. This,
in turn, means that when there is incoming data available for the producer thread,
it is better to proces all of it in a batch and put all generated messages into
a queue that wait the consumer thread to fetch.
2. T1 is smaller than T2, in this case, no matter how fast the producer therad is
generating messages, the consumer thread must consume them one by one. If we can
make sure that when the consumer thread is consuming one message, there is always
another message waits, the overall performance in tersm of latency will be the same
as using a lock-free queue (assuming all other factors are equal).

This inspires the author to design a simple flip-flop buffer to communicate between
two threads. This idea is in no way new. It is often known at the double-buffer
approach. This post is meant to treat it as a simple inter-thread communication
facility that in some cases can be used instead of a lock-free queue with strict
semantics reagarding thread synchronization. The design is described as below:

The flip-flop buffer contains the following:
1. Two slots that is good to contain 2 instances of a pointer-like type
2. One atomic bool called flip with its state managed by the producer thread.
3. Another atomic bool called flop with its state managed by the consumer thread.
A pointer-like type is a light-weight type that is
1. easy to copy, which means small in size and copying an instance of it is no more
expensive than do a `memcpy` of its facial value.
2. a nullable type.
3. owns some underying resource, The lifetime of the underlying resource should end
when the instance managing it expires.
It is called a pointer-like type because we often use pointers (in languages like
C/C++) to express the ownership relation between a managing pointer and resource
managed by it.

The algorithm goes like below.
Step one: the intial state, the buffer is empty.
```txt
|Producer Thread|--flip(false)--[Null]
                                [Null]--flop(true)--|Consumer Thread|
```
Step two: Producer thread put a message in the buffer, and alter `flip` to `true`.
```txt
|Producer Thread|--flip(true)  [MsgA]
                            \
                             \ [Null]--flop(true)--[Consumer Thread|
```
Step three: the cosumer thread starts to consume `MsgA` after alter `flop` to false.
this also means the Producer thread can start to put new `MsgB` in another slot.
```txt
|Producer Thread|--flip(true)    [MsgA]
                           \           \
                            \           \
							[MsgB]       \--flop(false)--|Consumer Thread|
```
Step four: After the consumer thread finishes consuming MsgA, it alters `flop`
to `true` again. And after putting `MsgB` into the second slot, the producer
thread alters `flip` to `false`. Now the lifetime of `MsgA` has ended,
```txt
|Producer Thread|--flip(false)--[MsgA(expired)]
                                [MsgB]--flop(true)--|Consumer Thread|
```
In each case, the thread that alters atomic states uses memory order `release`,
and the thread that read the atomics uses memory order `acquire`. And `flip` and
`flop` having the same value means there is new message to be consumed by the
consumer thread. And `flip` not equal to `flop` means either the consumer thread
can access the "current" slot (indicated by the value of `flop`) or there is no
more new data coming.

This design has two extra advantage comparing to lock-free queues:
1. Object lifetime managment is naturally incoporated into the overall flow. This
means resource allocation and deallocation alwyas happen in the same thread. For
example, some memroy allocators have trouble if you keep allocating memory in one
thread and and deallocating it in another thread. And the resource underneath
is objects like reading and write buffers, it is as cheap as to add some counters
to reuse the buffer.
2. This design is more cache-friendly than lock-free queue basically because
there is only 2 small memory chucks involved. This can be imporant for low-latency
application.

