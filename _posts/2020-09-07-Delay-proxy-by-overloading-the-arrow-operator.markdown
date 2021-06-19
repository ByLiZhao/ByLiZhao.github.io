---
layout: post
mathjax: true
comments: true
title:  "Delay proxy by overloading the arrow operator"
author: John Z. Li
date:   2020-09-07 19:00:18 +0800
categories: c++ programming
tags: operator-overloading, class-proxy
---

There is this class
`A` defined as below:
```cpp
class A {
public:
  A(std::string m_, int c_) : msg(m_), count(c_) {}
  void side_effect() {
    for (auto i = 0; i < count; ++i)
      std::cout << i << ": " << msg << std::endl;
  }

private:
  std::string msg;
  int count;
};
```
`A` has a public member function `side_effect()`,
which is used to trigger some side effects.
For this toy example, it does nothing but print a message for a given number
of times to the terminal.

The problem at hand is that when `A` is tested,
we want the `side_effect()` function performs its task in a controlled manner.
Again, for this toy example, we would like to illustrate the point by asking
the thread on which `side_effect()` is called to sleep for a pre-specified
time interval before the function body is executed.

Of course, we don't want the solution to be intrusive. That is, we don't want to
modify the class `A` itself. We also want the solution to be transparent, meaning
that calling the function in our test code is done in the same fashion that `A` is
used.

One way to achieve this to  create a delay proxy class of  class `A`.
In the proxy class the arrow operator is overloaded to insert
what we want to perform before the side effects are triggered.
To make the delay proxy class relatively generate, we make it a class template, as below:
```cpp
template <class T, int Pause, class... Args>
class delay_proxy {
public:
  delay_proxy(Args&&... args)
      :  up(std::make_unique<T>(std::forward<Args>(args)...))
        {}

  T* operator->() {
    std::this_thread::sleep_for(std::chrono::milliseconds(Pause));
    return up.get();
  }

private:
  std::unique_ptr<T> up;
};
```

What happens here is that the `delay_proxy` class owns a `unique_ptr` to an
instance of the test class; when an instance of the `delay_proxy` class is constructed,
parameters are forwarded to the constructor of the tested class, and configurable
parameters are also set. Then, one can just test class `A` like
```cpp
delay_proxy<A, 100, std::string, int> dp("this is a test", 10);
dp->side_effect();
```

We can see that we are using `dp` as if it is an instance of class `A`, because
the arrow operator "falls through" along the calling chain, eventually resolving
to a normal member function call after satisfying "pre-conditions" defined in the
function body of the arrow operator in class `delay_proxy`.
