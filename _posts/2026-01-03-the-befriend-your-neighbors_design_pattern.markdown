---
layout: post
mathjax: true
comments: true
title:  "The Befriend-Your-Neighbors design pattern, a recipe for huge classes"
author: John Z. Li
date:   2026-01-03 20:00:00 +0800
categories: programming
tags: c++, design-pattern
---
Though generally big classes should be avoided in programming, they are necessary sometimes.
C# has this nice feature called "partial classes", which allow you to split the definition
of a large class into separate small pieces, possibly located in different files.
When the compiler sees the class definition, it will pull in all pieces together and combine them
into a single class definition. That is a neat feature of practical usage. You need it when you need it.

Rust has a different approach to handle a similar problem. In Rust, the encapsulation boundary is enforced
at the module level, instead of file or type level. Code that belongs to the same module is allowed to
access other code in the same module. Since that a piece of code in a module often needs to work closely with
other code in the same module internally, it does not make much sense to hide implementation details
from each other behind class abstraction.

We can see that, C# and Rust solve the same practical software engineering problem through different technical approaches.
The lesson here is clear: *classes are often the wrong granularity of encapsulation*.
To solve real-world programming problems,
either you need to break class boundary sometimes, or you need to push the encapsulation to a higher abstraction layer, both
of which need support from the programming language.

When it comes to C++, the situation is nuanced. On one hand, C++ does allow you to break class encapsulation using the `friend`
keyword. On the other hand, friendship in C++ is not symmetric nor transitive.
If you want to make all classes inside a module befriend each other,
it becomes tedious. Say, you have 10 classes inside a module, mutual friendship will need you to declare
friendship `10 times 9 = 90` times. This is not ideal.

If all the classes that you want to make it friends of a class `A` are specialization of a class template `B`,
you can declare class template `B` a friend of class `A`.
```cpp

// Forward declaration of the class template
// Its definition resides in another file
template <typename T>
class B;

class A {
private:
    // Declare the template as a friend
    template <typename T>
    friend class B;
};
```
When the compiler compiles the definition of class `A`, it leaves a record in
the generated AST, which essentially says that for "Any class that is a specialization
of class `B`, do not restrict it from accessing the internal state of class `A`.

This method has an important variation, in which that the befriending class and
the befriended class is actually the same class template, like below,
```cpp
// Class template A
template <typename T>
class A {
private:
    // Declare all specializations of A as friends
	// class A here refers to the same class under definition
	// Important: you need to use another type variable so that is free from top-level one
    template <typename U>
    friend class A;
};

```
If all the classes inside a module happen to be conveniently abstracted as specialization of
the same class template, this basically achieves what Rust has as default.

If it is not the above mentioned case, and you do need big classes. For example, you have
a big class that models a trading session. This class needs to
- manage TCP state of one or more connections.
- manage session level state and actions (sequence numbers, resending, etc.) and keep the session alive by sending heartbeats when necessary.
- serialize and deserialize messages upon sending and receiving on TCP according to a given trading protocol.
- expose a control and observability interface to users, so that users can monitor its realtime status and exert controls from business requirements.
- knows how to persist trading messages and recover from possible errors.

Since the functionalities are highly coupled with each other, a big class, though not elegant,
is not that bad from a practical perspective.
But we can make things better by breaking functionalities into different classes,
each containing a logically separate subset of functionalities.
All these classes form a module, and code from a class in the module has to access
the private members of other classes of the module to do its job.
Friendship between these neighboring classes is essential to get the benefit of big classes
while having some level of separation of concerns.
This, is called the "befriend-your-neighbors" design pattern.

When the number of neighboring classes becomes large, as we mentioned, it becomes tedious to
let each class befriend every other classes in the module. We can avoid this tediousness
by creating a manager class, and let it
- manage every other classes in the module via a unique pointer
- be a friend of classes it manages.
- befriend each managed class in the module
- be accessible by each managed class from a pointer.

Besides the above, the manager class also has a list of `std::mem_fn` as static private
members, which wraps methods of the managed classes that need to be called by other classes.
So that if a managed class `A` wants to call a method from another managed class `B`,
it can do so by invoking the `std::mem_fn` wrapper of that method through the manager class.
Below is a code sample with two managed classes.
```cpp
class Manager;
class A{
	friend Manager;
	Manager* manager;
	void method(){
		// call B::method
		Manager::b_method(manager->b);
	}
};
class B{
	friend Manager;
	Manager* manager;
	void method(){
		// call A::method
		Manager::a_method(manager->a);
	}
};

class Manager{
	friend A;
	friend B;
	std::unique_ptr<A> a;
	std::unique_ptr<B> b;
	static inline constexpr auto a_method = std::mem_fn(&A::method);
	static inline constexpr auto b_method = std::mem_fn(&B::method);
};
```
When the number of managed classes becomes large, this pattern leads to clearer code.
Modern compilers can optimize away latency of indirect method call in this case.
