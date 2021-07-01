---
layout: post
mathjax: true
comments: true
title:  "Use signed integer for size"
author: John Z. Li
date:   2020-10-30 19:00:18 +0800
categories: c++ programming
tags: sizeof, ssize
---
C++20 has introduced `std::ssize`,
which  returns a signed integer representing the size of an object
that has implemented a public method `size()`.
While this is one step towards the right direction,
certain things are missing.
First, it does not introduce a new type `ssize_t`,
and lacks a constant zero value of that type.
Second, there is no corresponding `ssizeof`. It can be easily remedied like below.
```cpp
    //in some header file
    #include <type_traits>
    using ssize_t = std::common_type_t<std::ptrdiff_t,
        std::make_signed_t<size_t>>;
    template <class C>
    constexpr auto ssize(const C& c) -> ssize_t
    {
      return static_cast<ssize_t>(c.size());
    }
    template <class T, std::ptrdiff_t N>
    constexpr ssize_t ssize(const T(&array)[N]) noexcept
    {
      return static_cast<ssize_t>(N);
    }
    const ssize_t szero = static_cast<ssize_t>(0);
    #define ssizeof(x) static_cast<ssize_t>(sizeof(x))
```
And it can be used like this:
```cpp
    int main()
    {
      std::vector<int> v = { 3, 1, 4 };
      std::cout << ssize(v) << '\n';
      int a[] = { -5, 10, 15 };
      for (auto i=szero ; i < ssize(a); ++i) {
      std::cout << a[i] << '\n';
      }
      typedef int i4[4];
      std::cout << ssizeof(i4) << '\n';
      return 0;
    }
```
