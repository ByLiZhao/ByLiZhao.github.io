---
layout: post
mathjax: true
comments: true
title:  "Empty classes and zero length arrays in C++: Part 3"
author: John Z. Li
date:   2020-09-16 19:00:18 +0800
categories: c++ programming
tags: empty-classes, pass-key-idiom
---
This post is about the Pass Key idiom using empty classes.

Say there is a class called `tcp_connection` by you.
It is intended that instances of this class are managed by shared pointers,
so that when there is no outstanding asynchronous operations attached to it,
it gets destroyed automatically.
But you have made the constructor of that class private
and provided a static function `create` which returns a shared pointer to a newly created instance of
`tcp_connection`, for the reason   that a class user can not accidentally create
an instance of the class by directly calling its constructor.
You implemented the function `create` as below.
The code does not work because the constructor of the class is private, while `make_shared`
needs it to be public.
```cpp
    #include "pch.h"
    #include <memory>
    #include <boost/asio.hpp>
    using boost::asio::ip::tcp;
    class tcp_connection : public std::enable_shared_from_this<tcp_connection>
    {
    public:
      typedef std::shared_ptr<tcp_connection> pointer;
      static pointer create(boost::asio::io_service& io_service)
      {
      //does not what because the corresponding constructor is private
      return std::make_shared<tcp_connection>(io_service);
      }
    private:
      tcp_connection(boost::asio::io_service& io_service) //private constructor
      : socket_(io_service)
      {
      }
      tcp::socket socket_;
    };
```

One way to work around this limitation is to add a field of an empty class to it.
A public constructor can now be defined by wrapping the private one, like below,
```cpp
    class tcp_connection: public std::enable_shared_from_this<tcp_connection>
    {
      struct key{ explicit key(){}};
    public:
      typedef std::shared_ptr<tcp_connection> pointer;
      static pointer create (boost::asio::io_service& io_service)
      {
      return std::make_shared<tcp_connection>(io_service, key{});
      }
      tcp_connection(boost::asio::io_service& io_service, key)
      {//call the private constructor here
      }
    };
```
Since the class `key` is a privately defined  in the `tcp_connection` class,
there is no way, for an outside class user, to construct an instance of it.
Furthermore, since the default constructor of class key is marked `explicit`,
the restriction can’t be bypassed by
```cpp
    tcp_connection(my_io_service, {});
```
because there is no implicit conversion from an empty initializer list
to an instance of the empty class.

Similarly, empty classes can be used to achieve “selective friendship” is C++.
One problem with friendship in C++ is that once a class is declared a friend of another class,
The former will have access to all the members of the latter,
that is sometimes not what one wants.
For example, one wants to use a factory class with its only purpose
being to generate new instances of a class.
Except its constructors, we don’t really want the factory class to touch
anything else in that class.
In other words, we want that:
1. Only through the class `factory`, an instance of `A` can be constructed.
2. The factory class can not access private members of `A`
at the same time.
```cpp
    #include <string>
    class A {
      struct key {
      friend class factory;
	  private:
	  key(){}
	  };

    public:
      explicit A(std::string str_, key) :str(str_) {}
    private:
      std::string str;
    };

	class factory {
    public:
      A create(std::string str) {
      return A(str, {});
      }
    };

    int main() {
      factory fac;
      A a = fac.create("new A"); // OK
      A b{ "another A", {} }; //doesn't work because constructor of key is private
    }
```
Since class `factory` is a friend of inner class `key`, it can access private members
of `key`, but not the private members of the enclosing class, aka `A`.
So, the implementation details of class `A` can not be leaked via class `factory`.
Notice that the constructor of class `key` in this example is defined as being private but not explicit,
because the factory class relies on the fact that an empty initializer list can be implicitly
converted to an instance of it via its private constructor.
The constructor of class `A` is defined as public but explicit,
so that it can be called from the `factory` class,
but it can not be called directly bypassing the factory class to construct a new instance of `A`
because it requires that an instance of class key being initialized,
which an outside class user can do do he does not have access to class `key` or its constructor.
