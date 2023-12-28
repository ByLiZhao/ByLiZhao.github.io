---
layout: post
mathjax: true
comments: true
title:  "The difference between isolcpus and cgroups"
author: John Z. Li
date:   2023-12-24 15:00:18 +0800
categories: Python
tags: low-latency, isolcpus, cgroups
---
CPU isolation is critial for low-latency applications, such as high-frequency trading
systems. To squeeze the most latency reduction from hardware, it is important to
use dedicated CPU cores that are free from interruptions by other processes or
the OS kernel. For Linux servers, both `isolcpus` and `cgroups` are popular choices
for isolating system resources. This post intends to provide a brief summary of
their difference.

First, `cgroups` works at the process level. One process can contain multiple
threads. This means that if a set of CPU cores are assigned to a process, threads
of that process might still be moved onto and from CPU cores. If the application
is a multithreading one, even occasional short-lived threads might cause some jitter
on the main thread that is supposed to be of low-latency. Secondly, `cgroups` is
orthogonal to the OS scheduler, assigning a set of CPU cores to a process does not
mean other processes can not use CPU cores in that set. If there are some idle
cores at a given moment, say, because some threads are waiting I/O operations to
finish, the OS scheduler might dispatch some other threads that do not belong to
the isolated process to those CPU cores to prevent waste of system resource.

The utility `isolcpus`, on the other hand, provides more protection at the cost
of being not able to change configurations dynamically while the OS is running,
meaning `isolcpus` configuration must be boot-time configurations and rebooting
is required if a new set of configuration is to be applied. When a set of CPU cores
are isolated using `isolcpus`, it means the OS scheduler will totally ignore the
existence of those cores as if those cores are non-existent. The isolation is
done in such a way that the only way to actually run some threads on those isolated
coares are to use the `taskset` command, or the `sched_setaffinity` system call.
(Actually, the former calls the latter underneath.)

`isolcpus` allows fine-grained control over CPU affinity. For example, if we have
a low-latency application that will run 3 long-running threads on starting-up,
we can use `sched_setaffinity` to assign one core for each thread. Combining thread
affinity control with CPU isolation provided by `isolcpus`, maximal speed of execution
of threads can be achieved. It is guaranteed that no other thread can use those
designated CPU cores, also that unless a threads is blocking on some system calls,
the OS kernel will never migrate that thread to another CPU core.
