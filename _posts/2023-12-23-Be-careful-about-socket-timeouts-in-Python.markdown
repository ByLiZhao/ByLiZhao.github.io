---
layout: post
mathjax: true
comments: true
title:  "Be careful about socket timeouts in Python"
author: John Z. Li
date:   2023-12-23 15:00:18 +0800
categories: Python
tags: python, socket, TCP
---
The `socket` package in Python's standard library has an API to set timeout,
that is, the `settimeout` member function. `my_socket.settimeout(None)` sets
`my_socket` to blocking mode, and `my_socket.settimeout(0.0)` sets `my_socket`
to non-blocking mode. (They are equivalent to calling `my_socket.setblocking(True)`
and `my_socket.setblocking(False)`, respectively.) When `settimeout` is called
with a paramter of positive normal floating pointer number, the underlying socket fd
is set to non-blocking mode, yet the Python socket object is set to
blocking mode. Actually, the `getblocking()` member function is equivalent with
checking whether the socket object has `timeout != 0`.
A positive timeout does equal to `0`, such `getblocking()` will return `False`.
In other words, the non-blockness of the underlying fd and that of the Python socket
object are not the same.

If you are a programmer who are familiar with Linux's TCP stack, it is tempting
to think that the timeout mechanism of Python's socket library is implemented via
socket options `SO_RCVTIMEO` and `SO_SNDTIMEO`. These two options specify that
when receiving or sending data via a blocking TCP, the corresponding operations
are blocked for this period of time at most (specified using `setsockopt` with
`struct timeval`). The behavior of these two options are as below:
- The default values of them are `0`, meaning no timeout will never occur for
blocking mode TCP's, and having no effect on non-blocking mode TCP's.
- Only affect socket system calls that perform I/O, that is `accept`, `connect`,
`send`, `recv` and friends, but not `select`, `poll` or `opoll_wait`.
- For I/O related system calls, if some data has been transferred before timeout,
the number of bytes transferred will be returned; If no data is transferred before
timeout, `-1` is returned with errno set to `EAGAIN` or `EWOULDBLOCK`, or
`EINPROGRESS`. (The last one is only for `connect`.)

We can see that the Linux TCP stack is designed with flexibility in mind. It allows
the user to twist how I/O operations should be done to best suit his needs. For
example, if `SO_SNDTIMEO` is set to `1` second, and `send` is called on a
blocking-mode TCP with a very large chuck of data, say `1GiB`,
`send` is guaranteed to return within `1` second. This means, a very large size
of data being written or read can not freeze the whole application, which sometimes
is important.

Back to Python's `socket` in its standard library, it is tempting to assume that
`settimeout()` works in a similar fansion. But that is **not** what happens. Users
of Python are sometimes caught off guard if they hold wrong assumptions.
For example, if the `timeout` of a Python socket is set to `5` seconds, is it
guaranteed that the `send` function call will return within `5` seconds? The answer
is NO. The reason is that in each `send` or `recv` call on sockets, a `select` (implemented in terms of `poll` on Linux)
is first called with the specified timeout (this timeout is recalculated in each iteration of the loop).
If the corresponding `send` or `recv` can be performed after the `select` call, it moves forward to actually send or receive
data from the socket. But once the sending or receiving starts,
the timeout setting is not effective. The system call might take an unexpected long time.
As we said before, a positive timeout of `socket` object means the underlying `fd`
is in non-blocking mode. Yet, even for a non-blocking `fd`, the only guarantee we have
is that the OS does what it can do in one go and returns immediately. Returning immediately
does not seem bad, actually sufficient in many cases, but one should keep in mind
that the OS-level `send` and `recv` do not know about the timeout you set inside a Python `socket` object.

We can see that the design of Python's `socket` module emphasizes on ease of use
at the cost of some performance loss. Once a positive timeout is set, you always pay the price of the overhead
of calling `select` to check whether the socket is writable or readable before actually writing or reading.
This is obviously suboptimal in terms of performance.
Writing and reading is non-symmetric.
If there is no new incoming data available on a socket `fd`, it is no point of receiving.
But sending is different. Since the underlying `fd` is in non-blocking mode, optimistically trying sending without `select` first
does no harm. The worst can happen is that `send` returns immediately with no data being sent.
A good implementation should take this into account:
- for `recv`, `select` first, send when socket becomes readable, repeat until finish or timeout.
- for `send`, send first, if succeed, return immediately. Only enter the `select, send, check timeout` loop when needed.

The bright side of Python's `socket` design is that it makes writing simple message
exchange applications easy. This means, for blocking mode Python sockets with a positive timeout,
the `send` function can be basically regarded as a **All Or Nothing** interface, as well as the `recv` function given a big
enough receiving buffer, assuming that the other side is not acting too weird.
With this design, the users usually do not have to worry about many corner cases of TCP.
Given a reasonable timeout, and a reasonably large receive buffer,
if after sending a message, a reply is not received within the tiemout,  we can just close the socket.

It is worth to note that some other Python network related libraries inherit the same design of
`socket` regarding timeout. For example, with the below code, what is the expected behavior here?
```python
from requests import Session
session = Session()
session.get("http://example.com/example.zip", timeout = 5)
# equivalent to
# session.get("http://example.com/example.zip", timeout = (5, 5))
```
It is easy to think that the `session.get()` function call is guaranteed to
return in `10` seconds (`5` seconds for connection timeout and `5` seconds for receiving
data). The actual behavior is that if the 'zip' file is very big, if the server
starts sending data within the timeout, the function will only return after the whole
file has been received. The documentation of `requests` is explicit about this, quote
> timeout is not a time limit on the entire response download;
> rather, an exception is raised if the server has not issued a response
> for timeout seconds (more precisely, if no bytes have been received on
> the underlying socket for timeout seconds).

In extreme cases, for example, if a remote server sends 1 bytes per second, `session.get` is totally
fine with it and will never trigger a timeout exception.
In a sense, this is also a **All or nothing** interface.

