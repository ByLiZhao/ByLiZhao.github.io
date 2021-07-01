---
layout: post
mathjax: true
comments: true
title:  "overcommit-and-its-implication"
author: John Z. Li
date:   2020-10-20 19:00:18 +0800
categories:  programming
tags: overcommit, OOM
---
In a multi-process OS, there are more than one processes which are concurrently running.
Each process has its own virtual memory space, and each process acts in a way
as if it owns all memory resource that the machine has.
Virtual memory space, as an abstract layer provided by the OS kernel,
does not know from within itself that how much physical memory it can actually use.
Physical memory, as a global resource, is taken over by the OS kernel and granted
to processes as needed.
Naturally, when there is an out-of-memory situation,
a single process has not enough knowledge regarding what should be done about it:
A program can abort, or it can freeze and wait for the OS kernel to kill some other processes
(hopefully less important ones) so it can resume running later.
On the other hand, the OS kernel already has a mechanism to swap virtual memory
pages back and forth between RAM and hard disks (the swap partition or swap file as in Linux),
during handling page faults.
If swapping happens a lot, program performance will deteriorate to a critical point
that should be noticed by a system admin,
before an out-of-memory condition materializes.
So, unless swap is disabled, or the swap partition or swap files are two small,
OOM is rarely an exception that requires programmers to handle explicitly.

The C programming language exposes heap memory allocation via
`malloc` and `calloc`
( C++ through operator `new` and `new[]` ),
and programmers are supposed to check return values of
them to make sure that memory allocation has succeeded
( or catch out-of-memory exceptions in C++’s case ).
With memory over commitment of modern OSes, which is enabled by default on Linux,
it seems of little point to check return values of `malloc` or `calloc`.
Take Linux as an example, under default settings, unless the user is allocating
a ridiculously large chuck of memory (in this case, the kernel can know for sure the memory allocation would fail, so
it signify the failure immediately), heap memory allocations behaves in a way as if they are always successful.
Actual physical memory will be provided when the allocated memory is actually accessed.

Since over commitment makes heap memory allocation almost never fail.
It is only when the allocated memory is accessed, the OS kernel might
realize that an out-of-memory condition has happened,
and the OOM-killer starts killing processes. This makes one wonder:
> Is there any point in checking the return value of "malloc" or "calloc".

The problem with this perception is that over commitment might be disabled
by the system admin, or the application code might be running in a cgroup,
and the system admin has put a memory limit on it. If a program is running in a cgroup,
and a maximum limit of memory it can use is assigned to that cgroup,
the program might get killed even if there are physical memory not being used
(This is what happens if you set a hard memory limit on a docker container).
As a programmer, one should always check whether heap memory allocation has succeeded,
in order to make portable programs with respect to different hosting environment,
only keeping in mind that `malloc` returns success does not mean there won’t be OOM.
So, don’t make false assumptions, that is, don’t assume the following:

1.  Don’t assume that an OOM will not happen because all your malloc calls return success.
2.  Don’t assume your error handling code for failed malloc calls will be executed when there is memory exhaustion.
3.  If the OOM-killer starts running, don’t assume your program will (or will not) got killed.

With all this said, we can see that the problem of what to do in face of
an out-of-memory situation, is hard to answer in general.
The main issue is that, if a program has already run out of memory,
it is almost impossible to correctly handle the situation,
because your error-handling code probably needs to allocate some memory itself
(if it ever has a chance to run).
To make things worse, if you are a library writer,
there is no way you could know how to handle an OOM event because you have no
idea how a library user will use your library.
If potential allocation errors of the library is exposed to its users,
it will pollute the library interface,
and a library user will be expected to check whether memory allocation
errors might have occurred during a library call,
which they themselves might have no idea what to do other than abort the program.
If it is so, why not just let the library author call abort himself on OOM?
So, despite sounding brutal-force, aborting at OOM-events is not a horrible idea as it looks like.
And a lot of libraries did handle OOM conditions in this fashion

If, on the other hand, one wants to write high-reliable code,
he must know the host environment upon which his program will be running.
Only with the knowledge of things such as whether overcommitment is enabled,
whether memory swap is enabled, whether the program will run in a container with certain confguratons
etc., one can get deterministic behavior caused by out-of-memory conditions,
and how to correctly handle them.

Note: memory over-commitment is a natural consequence of the system call `clone`.
If the latter is introduced into an OS kernel, the former must follow.
It is a perfect example of intrinsic trade-offs of design decisions.

