---
layout: post
mathjax: true
comments: true
title:  "std::deque, almost always the wrong choice"
author: John Z. Li
date:   2026-02-07 20:00:00 +0800
categories: programming
tags: c++, dequeue, queue
---
This post argues that `std::deque` and `std::queue` (an adapter class of the former)
are almost always the wrong choice in C++ programming.

## How does `std::deque` work?
From the perspective of data structure,
a `std::deque` is basically an array of pointers to a set of fixed-sized chunks of memory.
The chunk size is a hard-coded compile-time constant and is implementation defined.
Take GCC's standard library implementation `libstdc++` as an example, for type of small sizes,
a chunk size of `512` bytes is used.

Take `std::deque<double>` as an example. Starting from an empty queue:
- The size of `double` is `8` bytes, which means a chunk can hold `64` elements.
- When a `double` is pushed into the queue, a chunk of `512` bytes of memory is
  allocated.
- The `double` is put at the beginning of the chunk.
- A pointer pointing to the chunk is pushed into the pointer array
```txt
// dequeue size is 1
|pointer array|                   |begin  |end
|pointer[0]   |--------> Chunk_0 (double, [empty], ...)
```

Another `63` elements can be pushed into the queue without triggering memory allocation.
After `64` elements have been in the queue, pushing another `65`-th
element will trigger allocation of a new chunk:
```txt
// queue size is 65
|pointer array   |                   |beggin
|pointer[0]      |--------> Chunk_0 (double[64])
|pointer[1]      |--------> Chunk_1 (double, [empty], ...)
                                             |end
```

Popping one element from front deconstructs the first element in first chunk and
increases an internal `begin` offset:
```txt
// queue size is 64
|pointer array   |                            |beggin
|pointer[0]      |--------> Chunk_0 ([empty], double[63])
|pointer[1]      |--------> Chunk_1 (double, [empty], ...)
                                             |end
```

It is only when all the first `64` elements are popped, the first chunk is freed:
```txt
// queue size if 1
|pointer array   |
|pointer[0]      |--------> NULL (Chunk_0 is freed)
|pointer[1]      |--------> Chunk_1 (double, [empty], ...)
                                     |begin  |end
```

Pushing another `64` elements into the back of the queue will be either like this
```txt
// pointer array reallocated or rotated, iterators to existing elements invalidated
// pointer[0] is previously pointer[1]
// dequeue size if 65
|Array of pointer|                   |begin
|pointer[0]      |--------> Chunk_1 (double[64])
|pointer[1]      |--------> Chunk_2 (double, empty, ...)
                                             |end
```
or like this:

```txt
// pointer array not reallocated nor rotated, iterators to existing elements still valid
// queue size if 65
|Array of pointer|
|pointer[0]      |--------> NULL (Chunk_0 is freed)
                                     |begin
|pointer[1]      |--------> Chunk_1 (double[64])
|pointer[2]      |--------> Chunk_2 (double, empty, ...)
                                             |end
```
In practice, this means you need to assume that any `push_back` or `push_front`
will invalidate iterators to stay safe.

The iterator for `std::deque` typically contains `4` elements:
- `node`, a pointer pointing to an element in the pointer array.
- `first`, a pointer pointing to the begin of the chunk.
- `last`, a pointer pointing to one-past-the-end of the current memory chunk.
- `current`, a pointer to the current element.

For an iterator `i`, `++i` first checks whether `++current == last`. If no, then
`++current` already points to the next element. Otherwise, update `node` to `++node`,
and update `first/last` to the begin/end of the chunk pointed by updated `*node` respectively,
and `current` to the value of updated `first`.
`--i` is similar, but `current` is compared with `first` for boundary check.

Random access is done using modular arithmetic. For an iterator `i`, `i + n` is computed
by taking into account of the fact that a chunk only contains a fixed number of elements.
```txt
node  = node + (current - frst + n) / Number_of_elements_per_chunk;
current = *node + (current - first + n) % Number_of_elements_per_chunk;
frst = *node;
last = first + Number_of_elements_per_chunk;
```

We can see the fixed chunk size serves two purposes at the same time: to make random access possible
and reduce memory allocation cost by allocating a big chunk at a time.
A typical optimization employed by many implementations is to keep one chunk when the queue is popped empty,
so that a operation sequence like below would not trigger repetitive  allocation and deallocation.
```txt
push one; pop one; push one; pop one, ...
```

## Why you probably should not use `std::deque`
`std::deque` uses a clever trick to reduce memory allocation cost,
while keeping iterators random accessible and stable under popping from both ends.
All of these are nice at first glance, but not so much under closer inspection.

In C++, with modern malloc implementations such as `jemalloc` or `tcmalloc`,
each thread often has its own per-thread memory pool or arena. Allocation of a small
pieces of memory can often be satisfied very fast without contending on a global lock.
Large allocations, likely requires coordination with global malloc state or the OS.
The latter can be 10 times or 100 times slower than the former.
This means, with certain use pattern, `std::deque` can even be slower than `std::list`.
Yes, `std::list` will trigger more allocations and deallocations, but modern malloc implementations
are very good at handling frequent small memory operations. In a sense, they are already doing
small objects pooling for you but transparently. So, instead of "I am going to use `std::deque` because it is fast",
measure before making a decision. The numbers might surprise you.

Even if the standard allocator is not enough,
C++ already has a way to optimize memory allocation for C++ containers, that is, allocators.
Allocators are supposed to abstract away memory management for container types.
The problem with `std::deque` is that it is tied to a  specific memory management strategy in a hard-coded way,
making it virtually impossible to use custom allocators with it.
An allocator for `std::deque` must handle 2 types of memory allocation,
one for the allocation of the internal pointer array,
and one for the allocation of fix-sized chunks, the size of which is an implementation detail that is not exposed to users.
If you want to make an optimized allocator for `std::deque`, you need to know what is chunk size used  in GCC's `libstdc++`,
The size will be different for Clang's implementation of `std::deque`, which uses a chunk size of `4096` bytes.
Even if you are able to implement a custom allocator for `std::deque` with some clever tricks,
will it be faster than, say, `std::list` with custom allocators?

The argument here is not that you should use `std::list`, but that you could.
Actually, if you need a first-in-first-out queue, you probably should consider a ring buffer (also called circular buffers) first.
Modern implementations of ring buffers often support capacity grow. It is only that users need to explicitly trigger reallocation.
For example, you can use `set_capacity` to grow capacity of a `boost::circular_buffer`.
If you need to push one million double numbers into a `boost::circular_buffer`, you can set its capacity to be big enough to hold
all of them. This will trigger a one-time memory allocation. While with `std::deque`, it will trigger at least ` 1 million / 64 = 15625` memory allocations.
`std::deque` does not expose a `reserve` API to users. Even if you know in advance how many elements you are going to push into a queue,
there is no way you can let `std::deque` to preserve memory and stop eager deallocation.

For simple cases:
- If you only need a first-in-first-out queue, consider circular buffers. Elements are stored in contiguous memory, which makes it fast and random accessible.
  Better, push and pop don't invalidate iterators. Growing capacity will invalidate iterators, but that is under your control. Moreover, it does not frequently
  grow and shrink like `std::deque`. Usually, you have a good estimation of the size of data set of your application. Setting a big enough capacity at queue
  construction will suffice in most cases.
- if you really need to push and pop on both ends, consider `std::list` first. If performance is confirmed an issue after measurement, consider `std::pmr::list`.
  If even `std::pmr::list` not fast enough for you, consider using custom allocators with `std::list`. Performance-wise, the biggest concern for `std::list` is
  usually poor cache locality and repetitive memory allocation/deallocation. Custom allocators can greatly reduce the cost.

## Go one step further
If you application is really performance critical, and circular buffers are too restrictive and `std::list` is too slow even with custom allocators,
we can go one step further to pursue a radical design.

Let us start with queues. We can make a queue by wrapping a `std::vector<std::optional>` like below.
The code sample is only to illustrate the idea, and intentionally incomplete.
Too much code would hide the core idea in implementation details.
```cpp
template<typename T>
class queue {
public:
template<typename... Args>
void emplace(Args&&... args) {
	data_.emplace_back(std::in_place, std::forward<Args>(args)...)
}

void push(const T& value)
{
	data_.emplace_back(value);
}

void push_back(T&& value)
{
	data_.emplace_back(std::move(value));
}

void pop() {
	if(!empty()){
		data_[head_++].reset();
	}

	if (empty()) {
		compact();
	}
}

bool empty() const {
	return head_ == data_.size();
}

std::size_t size() const {
	return data_.size() - head_;
}

void compact() {
	data_.erase(data_.begin(), data_.begin() + head_);
	head_ = 0u;
}

private:
	std::vector<std::optional<T>> data_;
	std::size_t head_ = 0u;
};
```

With this design, the queue has the following properties trivially:
1. All elements reside in a contiguous range of memory. It has good cache locality and fast iteration.
2. It is random accessible just like `std::queue` but the iterator is lighter (because it is essentially just a pointer).
3. Pushing elements might trigger reallocation and invalidate iterators just as `std::queue`,
   but the cost of memory allocation of pushing n element is O(log(n)) contrasting to O(n) of `std::queue`.
   Because the capacity of `std::vector` grow geometrically while that of `std::queue` grows linearly.
   In other words, our queue is friendlier for bulk operation.
4. "bulk-push-bulk-pop" cycles does not cause continuous allocation and deallocation cycles.
5. Users can reserve capacity in advance to reduce reallocation.
6. It works well with custom allocators.

The only drawback is that compaction only happens either when the queue is drained or when method `compact` is explicitly called.
So, with some peculiar usage patterns, for example, a sequence like "push 100 elements, pop 99 elements, repeat ..." will cause trouble.
But honestly, in most of cases, there is no reason to have left-over elements in a queue.
The reason you push some objects into q queue is usually because you need to process them all later.
Even if you have good reasons to have left-over elements in the queue, you can always call `compact` when you see that the queue has become too sparse.

`std::optional` is optional for nullable types and trivially destructible types.
- For a nullable type `T`, `std::vector<T>` can be used. When `pop` is called, the element is set to the null value of `T` followed by `head_` increased.
- For a trivially destructible type `T`, `std::vector` can be used with `pop` being just increase `head_`.

With two vectors arranged head-to-head, `deque` can also be implemented with the same idea.
Code is omitted here. I don't want to make this post unnecessarily.

To sum up, When you can easily come up with better alternatives, it might be a strong indicator that `std::deque` and `std::queue` are not well designed.

