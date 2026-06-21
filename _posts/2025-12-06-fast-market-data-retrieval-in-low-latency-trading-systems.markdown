---
layout: post
mathjax: true
comments: true
title:  "Fast market data retrieval in low-latency trading systems"
author: John Z. Li
date:   2025-12-06 20:00:00 +0800
categories: programming
tags: c++, low-latency, lock-free
---
A market data module is necessary for any trading system.
For low-latency trading systems, this is often done with multi-threading on a concurrent hash table,
with a dedicated market data thread responsible for receiving,
parsing and updating market data in the hash table, and another trading thread acting as
a reader who retrieves market data from the hash table.

While this approach definitely works, and there are many ways to implement thread-safe
hash tables. The tradeoff regarding latency and implementation complexity is not always obvious.
For example:
- Should new symbols be allowed to be inserted into the hash table while trading is ongoing.
  If yes, will this trigger reallocation/rehashing?
- Thread safe reallocation/rehashing is hard to be done correctly. We also do not want long pauses
  during trading because reallocation/rehashing. So we might want to give the hash table
  a big enough initial size. But how big is big enough?
- Do we want the hash table to have stable iterators? If so, latency might be negatively affected by
  design constraints imposed by iterator stability.
- If we choose an open-addressing hash table implementation for cache locality, but iterators will not be stable.
- If iterators are not stable, we lose an important optimization opportunity, that is to cache the address of the market data
  of a given symbol, which allows us to only do hash table looking-up for a symbol once and access the market data of the symbol
  via the cached pointer afterwards.
- If we think the above caching optimization is important because we don't want to pay the price of hash table looking-up
  each time the symbol is traded, we might choose to change the hash table type from `hashtable<Symbol, MarketData>` to
  `hashtable<Symbol, MarketData*>`, but adding a layer of indirection hurts memory locality.

In this post, a fast market data mechanism  will be presented, which I believe solves some of these concerns.
Instead of multi-threading, we use a separate process for market data. This market data process shares market data
with the trading process with shared memory. The shared memory is mapped into the virtual space of the market data
and that of the trading process uisng `mmap`. The market data process writes to the shared memory, while the trading
process reads from it.

The shared memory is exposed to both processes as an array of `MarketData`, which looks roughly like below
```cpp

struct MarketData{
	char symbol[MaxSymbolLength]
	double bid;
	double ask;
	//... etc
	std::size_t counter;
} market_data_array [MaxSymbolCount];
```
- The `MarketData` at offset `0` is re-purposed as a counter for the total number of symbols in the array.
  That is, the value `counter` stands for the effective length of the array minus 1.
- The allocated size of the array`MaxSymbolCount` is a very big number that is enough for our use.
  Though the array is large, it won't consume large amount of memory, as the OS initializes `mmap`ed memory lazily.
  Only when a page is actually needed, accessing the page triggers a page fault, and the OS will allocate a new physical page
  by handling the page fault.

With this arrangement, adding a new symbol is as simple as appending a new element at the end of the array, which is a simple
`memcpy`, updating the total symbol count at offset `0`. Only actually used memory (up to multiples of page size) is allocated by the OS.
Essentially, we have a growable array without ever triggering reallocation. After the market data process appends new element at the
end of the array, it notifies the trading process using any IPC mechanism (for example, by writing to an `eventfd` monitored by
the event loop of the trading process). On receiving the notification, the trading process reads the total symbol count at offset `0`,
compares it to the number of symbols it already has, and updates its own hash table by reading from the newly added elements at the array end.

The market data process and the trading process, maintains their own symbol-to-market-data hash tables separately.
So each of them can choose a hash table implementation that best suits its needs. For example, the trading process
might already has a hash table that stores symbol related information to validate trading orders, it can add market
data to that symbol by adding a `MarketData *` member to the symbol definition class. An trading order is Order Book
might also has a member of `MarketData *` member for quick order modification validation. As the market data for a symbol
is just a stable pointer which never expires, we can cache the pointer wherever needed.

There is still one piece missed, that is synchronization safety.
It is not safe to writing to and reading from the same piece of memory by two threads at the same time.
This can be solved by a simple trick:
- Treat the `counter` in each `MarketData` as an atomic variable;
- The initial value of `counter` is always `0`;
- Whenever the market data threads needs to update the market data of a symbol;
  it sets the `counter` to an odd number (by increasing it by `1`), then updates
  market data. When udpate finishes, it sets the `counter` to an even number (by increasing by `1` again).
This way, the `counter` being an odd number means the containing `MarketData` is in transition state.
When the trading process reads `MarketData`, it
- First check atomically whether `counter` is of odd value, if yes, retry, until getting an even value;
- Go ahead and reads from `MarketData`;
- After reading from a `MarketData`, check atomically whether the value of `counter` has changed.
  If yes, starts over.

As in low latency trading systems, we have pinned each thread into a dedicated CPU core.
A thread won't be preempted by the scheduler because another thread needs to use the same physical core.
The probability that the writer thread is interrupted while it is updating a `MarketData` is extremely low.
Even a signal handler interrupts the thread in-between a `MarketData` update, it will be resumed in a very short period of time.
So, retry on the reader thread side is not expensive. For CPUs with Cache Coherence Protocol,
it is just spinning until a cache line is invalidated by another core.

Extra optimization can be applied trivially. For example, the market data process can
arrange symbols in the array from ones with high trading volumes to those with low trading volumns,
so that the "hot" symbols always appear at the beginning portion of the array.
As hot symbols are traded more frequently, it is more likely their market data is already in
CPU cache, thus further reducing latency.
