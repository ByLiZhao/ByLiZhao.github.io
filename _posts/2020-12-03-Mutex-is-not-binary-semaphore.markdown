---
layout: post
mathjax: true
comments: true
title:  "Mutex is not binary semaphore"
author: John Z. Li
date:   2020-12-03 19:00:18 +0800
categories: c programming
tags: mutex, semaphore
---
Traditionally, the two terms are used by different people in many slightly different ways.
Thus, there are lots of confusions around the topic.
For some, mutex is a high level (abstract) concept,
and binary semaphore is one way to implement it.
This notion is popularized by many literature that teaches their audience on
things like “how to implement mutex using semaphore”,
or “you can use semaphore or monitor to implement mutex”, etc..
For some, mutex is just a term that is interchangeable with
binary semaphore,
what differentiates them is how they are used:
**mutex for exclusive ownership and binary semaphore for notification**.
While this is not wrong, there is another way to compare them against each other.
from the perspective of an operating system.

Semaphore has well-defined semantics.
The definition of a semaphore is independent from the existence of and OS.
The semantics of a binary semaphore does not change with or without an underlying OS.
On the other hand, mutex is almost always used in the cotext of an OS.
For any operating system to be usable,
it often must provide the following extra guarantees when implement mutex:
1.  The implementation of mutex must provide a way to avoid the effect of possible
priority inversion (for example, by setting priority inheritance in Linux).
2. The implementation of mutex must correctly handle the
case when the thread that owns the mutex is cancelled (for example, robust mutex in Linux).
3. The implementation of mutex must provide a way to successfully
Lock a mutex again from the same thread that owns it already (for example,  recursive mutex in Linux).

One can argue that the last two features are not essential,
but the first one (dealing with priority inversion) is essential to guarantee correctness
of any priority-based preemptive operating system.
For realtime systems, priority inversion can lead to catastrophic consequences
if a high-priority task misses its deadline because it is starved from priority inversion.

This means, the actual semantics of mutex must be considered in the context of the underlying  operating system
and based on what additional features are required for a particular use case. For example, if a function that tries to  lock
as mutex in its function body is recursively called, the mutex used must also be recursive-safe.


