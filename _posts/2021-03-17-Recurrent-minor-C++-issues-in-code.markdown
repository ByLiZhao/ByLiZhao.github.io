---
layout: post
mathjax: true
comments: true
title:  "Recurrent minor C++ issues in code"
author: John Z. Li
date:   2021-03-10 19:00:18 +0800
categories: robotics
tags: ROS, Navigation2
---
While reading real-world C++ code in some open-source projects, I noticed the following
minor issues appearing again and again:

## Use `auto i = 0` when one really wants `unit64_t` or `size_t`.
Consider the following code snippet from a project I encountered lately:
```cpp
auto assistant_wheel_index = 0;
      {
        std::lock_guard<std::mutex> lock(current_assistant_wheel_index_mutex_);
        assistant_wheel_index = GetAssistantWheelIndex(
            current_assistant_wheel_index_ + assistant_ticks);
        assistant_wheel_[assistant_wheel_index].AddTask(task);
      }
```
The variable `assistant_wheel_index` is induced as `int`, but the programmer actually wants
an `uint64_t`.  The compiler would complain that a value of `uint64_t` is assigned to
an `int`, thus might potentially leads to incorrect results. If we change the first line
of the code snippet to
```cpp
auto assistant_wheel_index = 0ul;
```
or
```cpp
auto assistant_wheel_index = 0uul;
```
The compiler would stop complaining. But, this quick and dirty fix has the following problems.
1. The first `ul` suffix in the first fix will make `assistant_wheel_index` an variable of type `usigned long`.
even on 64-bit systems, it is not guaranteed to be of 64-bit width. On Windows, an `unsigned long` is of 32-bit width,
while on Linux, an `unsigned long` is of 64-bit width.
2. The second fix is slightly better, except it still can be wrong strictly speaking. Because the language
standard does not guarantee that an `usigned long long` is of the same size as an `uint64_t`.
Although on most real-world platforms, they are of the same size.

If you want to be pedantically correct, making sure that your code is portable even to most arcane platforms,
the line of code can be fixed by
```cpp
auto assistant_wheel_index = uint64_t{0};
```
But, hey,  please be reminded that we don't have to always use `auto`,
a plain and old variable definition is and (better?) good:
```cpp
uint64_t assistant_wheel_index = 0u;
```

This is just a minor thing, but can be very annoying sometimes. Actually, there is
a [language proposal](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2019/p1280r2)
that tries to fix this by adding literal suffixes to width-fixed integer types.

The same can happen one wants to index an array, or an vector with a variable of type `size_t`.
C++23 adds a `z` or `Z` suffix to integer literals for `size_t`.  Or, maybe you want to go
with `std::ssize()` introduced in C++20, see [this stackoverflow post](https://stackoverflow.com/questions/56217283/why-is-stdssize-introduced-in-c20)
for a discussion.

## Using `size() == 0` instead of `empty()` to containers.
For example, in a cpp file of a project, there are the following:
```cpp
 if (frame_id.size() > 0){\\...}
 if (frame_id.size() == 0) {\\...}
```
Granted, this is only a minor issue, but but C++ standard library containers
do have a `empty()` member method. `sth.empty()` is more readable than `sth.size() == 0`,
and `!sth.empty()` is more readable than `sth.size() > 0`.

Note: Before C++11, GCC's implementation of `size()` for `std::list` is actually of O(n)
time complexity. See [this interesting post](howardhinnant.github.io/On_list_size.html)
on why some people think it is a good idea to make the `size()` method an O(n) operation
for lists. From C++11 onward, the language standard mandates that the `size()` method for
all standard containers must return in O(1) time.

## Unnecessary `static` specifier for `constexpr` global variables.
For example, in a header file that is included by multiple source file, there are
the following definitions:
```cpp
constexpr float minExpPower = -10.0f;
constexpr float maxExpPower = 5.0f;
constexpr int anchorSizeFactor = 2;
constexpr int numScales = 3;
```
These are all `constexpr` global variables. Later the author of the code change the piece of
code into the following:
```cpp
static constexpr float minExpPower = -10.0f;
static constexpr float maxExpPower = 5.0f;
static constexpr int anchorSizeFactor = 2;
static constexpr int numScales = 3;
```

What the author of the code intended to achieve is that he wanted to avoid violating  the One Definition Rule.
But `constexpr` implies `const`, and `const` global variables have internal linkage.
Thus each source file including the header has its own copy of those `constexpr` variables.

Note: if you are working with C++17, adding the `inline` specifier to them might be better, as below:
```cpp
inline constexpr float minExpPower = -10.0f;
inline constexpr float maxExpPower = 5.0f;
inline constexpr int anchorSizeFactor = 2;
inline constexpr int numScales = 3;
```
This should be the preferred way is for the following reasons:
1. The `inline` specifier relaxes the One Definition Rule. This means if the definition appears multiple times,
the resulted code is still valid, as long as these definitions are identical. This can sometimes improve code
readability, as we can put the definition of a `inline constexpr` variable near where it is used.
2. The language standard guarantees that the symbol of the `inline constexpr` variable is unique in the whole program.
So, if someone takes the address of a `inline constexpr` global variable, the address is unique no matter in which translation unit
the address is taken.

Note: functions and static member variables of a class that are specified as being `constexpr` is `inline` by default.
In those cases, one does not have to add the `inline` specifier.

## Use `min()` when one should use `lowest` when dealing with limits of floating point number.

The confusion comes from the fact that when one applies `std::numeric_limits<T>::min()` to integer types,
the function return the minimum value of that type. For example
```cpp
std::numeric_limits<short>::min();
```
returns `-32768`.

The gotcha appears when one tries to apply the function to floating point numbers.
For floating point number, the function actually returns the **minimal positive number** of that type.
For example,
```cpp
std::numeric_limits<float>::min()
```
yields `1.17549e-38`, that is the smallest number that is greater than zero representable by the type of `float`.

the smallest finite number that can be represented by a number of type `float` is given by
```cpp
std::numeric_limits<float>::lowest()
```
which will yields `-3.40282e+38`.

Even experienced C++ programmers can be bitten by this one. Be careful.

## use `front()` to empty containers.
Use `front` to empty containers is UB in C++.
But this happens frequently in real-world C++ code.

The reason why this happens is that it is OK to get the `begin` iterator of a container
even if its is empty. Actually, the language standard guarantees that for an empty container,
`a`, `a.begin() == a.end()`. Many programmer gets the wrong impression that it is OK
to get a reference using `front()` to a container even if it is empty as long as he
does not try to access the object referenced by the return value of `front()`.

One takeawy: Stick to iterators.

## Unnecessary usage of `to_string`.
The family of `to_string()` functions are supposed to provide replacements for `sprintf()`
with corresponding format specifiers.
If one only wants to output some numbers on screen, he can just go with `ostream`.
It is not only that `to_string` is redundant in this case (one does not have to
first get a string from `to_string()` to output a number), but the resulted output can be different.
Consider the following example:
```cpp
#include <iostream>
#include <sstream>
#include <string>

int main()
{
    double f = 23.43;
    double f2 = 1e-9;
    double f3 = 1e40;
    double f4 = 1e-40;
    double f5 = 123456789;

    std::string f_str = std::to_string(f);
    std::string f_str2 = std::to_string(f2); // Note: returns "0.000000"
    std::string f_str3 = std::to_string(f3); // Note: Does not return "1e+40".
    std::string f_str4 = std::to_string(f4); // Note: returns "0.000000"
    std::string f_str5 = std::to_string(f5);

    std::ostringstream oss;
    oss << f << '\n'
            << f2 << '\n'
            << f3 << '\n'
            << f4 << '\n'
            << f5 << '\n';
    std::cout << oss.str() << '\n';

    std::cout  << f_str  << "\n"
               << f_str2 << "\n"
               << f_str3 << "\n"
               << f_str4 << "\n"
               << f_str5 << '\n';
}
```
This program will print the following:
```
23.43
1e-09
1e+40
1e-40
1.23457e+08

23.430000
0.000000
10000000000000000303786028427003666890752.000000
0.000000
123456789.000000
```

We can see that the `ostream` functions have more sensible defaults, producing more human-friendly output.
Another thing to notice is that for small numbers, the `to_string()` function produces results that are less precise.
This is probably the price to pay to be exactly compatible with C's `sprintf()` function.






