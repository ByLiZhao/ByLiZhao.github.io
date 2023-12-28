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
with a paramter of positive normal floating pointer number, the socket is set to
blocking mode with the specified timeout in seconds.

If you are a programmer who are farmilar with Linux's TCP stack, it is tempting
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
the number of bytes transferred will be returned; If no data is transfered before
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
of Python are sometimes caught off guard if they hold the wrong assumption.
For example, if the `timeout` of a blocking socket is set to `5` seconds, is it
guaranteed that the `send` function call will retuan within `5` seconds? The answer
is NO. The reason is that in each `send` or `recv` call on sockets, a `select` is
first called with the specified timeout. If the corresponding `send` or `recv` can
be performed after the `select` call, it move forward to actually send or receive
data from the socket. But for blocking sockets, once the sending or receiving starts,
unless there is an error, the system call will only return after all data is sent
for `send`, or either all available data is received given a big enough receiving
buffer.

We can see that the design of Python's `socket` module emphasize on ease of use
at the cost of some performance loss. This means even you use `epoll` to wiat on
I/O events on sockets, at the time of actually calling `send` or `recv`, there is
always the overhead of calling `select`. This is obviously suboptimal in terms of
performance. The bright side of this design is that it makes writing simple message
exchange applications easy. This means, for blocking sockets, the `send` function
is a **All or nothing** interface, as well as the `recv` function given a big
enough receiving buffer. With this design, the users usually do not have to worry
about message fragmentation at the socket level.

This design align well with the overall design goals of Python, that is ease of use.
Some other Python network related libraties inherit the same design principles of
`socket`, for example, with the below code, what is the expected behavior here?
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
Say, if a remote server sends 1 bytes per 4 seconds, `session.get` is totally
fine with it and will never trigger a timeout exception.
In a sense, this is also a **All or nothing** interface.

