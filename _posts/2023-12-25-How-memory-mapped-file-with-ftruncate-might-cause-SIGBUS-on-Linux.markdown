---
layout: post
mathjax: true
comments: true
title:  "How memory mapped file with ftruancate might cause SIGBUS on Linux"
author: John Z. Li
date:   2023-12-25 15:00:18 +0800
categories: linux
tags: linux, sigbus, memory-mapped-file
---
When memory-mapped files are used on Linux systems, once typical usage pattern
is to start with an newly created empty file or a small sized one, and then to
enlarge the file size using `ftruncate` to allocate some file space, so that the
application, after the file being mapped memory, can append data to the file.

One caveat of this approach is that Linux does not actually allocate the required
disk space when the file is enlarged using `ftruncate`. Just like over commit when
allocating memory, the Linux filesystem does something similar when `ftruncate`
is called. When a file is enlarged using `ftruncate`, the filesystem may not
actually allocate actual space on disk, but instead can choose to just mark the
enlarged file of being the required size. Actual disk space is only allocated
when the space is needed, e.g., some data is dumped at the newly added capacity of
the file. In the case of memory mapped file, this happens when there is a page fault.
When data is appended to a memory-mapped file, if there is not a corresponding
disk chunk for the page that is being written onto, a page fault will occur. In
handling the page fault, the filesystem will allocate a disk chuck for the file
and mapped that piece of file to the main memory.

Bad things can happen if there is not enough disk space left when the page fault
happens. In this case, the filesystem will not be able to create a new disk chunk
to be mapped to the main memory. On Linux systems, a SIGBUS will be raised and
the running process will be terminated. If you are responsible to find out the root
cause of such an SIGBUS error condition, you might have a hard time. Say, if someone
happened to delete a large file after the error condition, so that when you look
at the system, you did not even notice that the system had a not-enough-disk-space
issue when SIGBUS occured.

The remedy for this to to use `fallocate` instead. After a successful `fallocate`
call, subsequent writes into the range specified by the function call is guaranteed
to be successful.
