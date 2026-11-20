---
layout: post
mathjax: true
comments: true
title:  "A fast hashtable for low latency trading systems"
author: John Z. Li
date:   2026-03-07 20:00:00 +0800
categories: programming
tags: c++, low-latency
---
First of all, this post is not about designing a general purpose hashtable.
The hashtable design we will present in the following part of this post is purely business driven.
It is suitable only for the niche area of low-latency trading.
More explicitly, the hashtable is one used for Order Book in low latency trading systems.
From an algorithmic perspective, the idea is trivial, though I think a better word is "simple".
A simple design, that exactly solves the problem at hand, no more no less, is a good design.

Let us start with some background information. An indispensable part of an electronic trading
system is its Order Book, which is essentially one or several hashtables that map order ids to orders.
Order ids are generally just small-sized alphanumeric strings, while orders are objects that track realtime state of trading orders.
Since an order object is usually quite large, possibly several hundred bytes in size, the mapped-to type is often a pointer type or a smart pointer type.
The concrete signature does not matter. Essentially, we want a hashtable in the form of `hashtable<ShortString, Order*>` in C++ syntax.

One unique of the hashtable is that it does not need to support erase or delete operations,
as it is beneficial to keep the full day's trading data in memory.
The system is started before trading hours in the morning, and is shutdown after hours in the afternoon.
The order book can not grow unlimited, for the users of the system are apparently more constrained by other factors than the server's main memory.

Another characteristic of the system is that when a new order is inserted into the hashtable,
it will typically filled by Exchange in minutes or even in seconds. This becomes more apparent as users send IOC or FOK orders.
An order sits idly on Exchange for a long period of time is usually a sign that your pricing algorithm is somehow wrong.
This temporal locality of orders is the key factor that shapes the design of order book, the hashtable.
A key-value pair that has been just inserted into the hashtable has a very high chance of being visited again shortly after.
In other words, the hashtable should be optimized for young elements.

Starting with a open-addressing scheme, the eviction policy for our hashtable is naturally a LCFS (last-come-first-served) one.
When a new key-value pair is inserted, if the slot indicated by key's hash value is already occupied, just evict the old occupant,
so that, when it is searched again later, its performance is of that of perfect hashing.
The evict pair, starting from the slot indicated by its key's hash value (not the slot it is evicted from),
does linear probing to find a new slot to reside. The maximal steps this linear probing can take is capped by a pre-specified constant.
A good choice of it is `16`. Looping over `16` slots in contiguous memory is fast due do cache locality.

Now it comes to the question: what to do if an evicted pair can't find a free slot within the `16` slots after tis perfect-hashing slot?
A great amount of wisdom has poured into this topic for general purpose hashtables.
The difficulty for general purpose hashtables is that they have to perform reasonably well in the face of severe hash collision.
An popular choice in the area of low-latency programming is Robin Hood hashing.
But for our specific use case, there is actually a simpler way: just throw it into a backup hashtable.
The performance of the backup hashtable is not very important, as it only serves as a backup store for evicted elements.
You can use `std::unordered_map` for that purpose.

To control the probability that a pair is evicted to the backup table, the load factor of the main table should be controlled.
A simple choice is `2/3 ≈ 0.66`. Whenever the load factor of the main table exceeds `0.66`, the underlying array grows by a factor of `1.5`.
Assuming the hash function used is good enough to roughly distribute elements evenly onto the underlying array.
When load factor is `0.66`, The probability that an evicted pair can't find a lost in `16` slots is about $ 0.66^{16} \approx 0.0015 $.
The probability that this order is also an open order is even lower, as open orders at any moment in the system except at market open is only a small fraction
of all the orders. Say, less than 10 percent of all the orders on Order Book are open orders, the probability that an open order
is evicted to the backup table is $0.0015 \times 0.1 < 0.0002$, that is 2 in every 10 thousand orders.

With this simple design, for almost all newly inserted orders, searching is just calculating hash value and jumping to its perfect-hashing slot.
The performance is near that of a globally perfect-hashing hashtable. For a small fraction of queries, an extra linear probing of fix-sized range
is needed. Only when you are really unlucky, 2 hashtable looking up will be needed. The pattern is well-suited for branch predictors of modern CPU's.
After several times of warm-up looking-up, it will predict with high certainty about which branch to take.

Sometimes, an ad-hoc design is a good design, if you know your business requirements.
