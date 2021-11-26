---
layout: post
mathjax: true
comments: true
title:  "Recurrent minor issues in C++ code: Part 2"
author: John Z. Li
date:   2021-04-17 21:00:18 +0800
categories: c++
tags: move-semantics
---
While reading real-world C++ code in some open-source projects, I noticed the following
minor issues appearing again and again: (read the first part of this post [here](https://bylizhao.github.io/c++/2021/03/17/Recurrent-minor-issues-in-C++-code-part1.html).

## Move an object returning it from a function
Consider the following code:
```cpp
std::vector<int> my_fun(void)
{
    std::vector<int> vec {1,2,3,4,5};
    return std::move(vec);
}
```
This kind of code appears frequently in projects I have encountered. The intention of programmers who
do this that they wants to accelerate `return` by avoiding unnecessary copies. But this actually make
the code slower.

Starting from C++17, Return Value Optimization (RVO) is now mandatory. This means, if the code is written as
`
return std::vector{1, 2, 3, 4, 5};
`
no copy or move is performed. The compiler guarantees that the vector is constructed at the caller's site.
If the returned variable is given a name first as in the first code sample, Named Return Value Optimization (NRVO)
might kick in. This also will leads to no copy or move being needed. Though NRVO, unlike RVO, is not mandatory,
most compilers have implemented it as an optimization. Performance-wise, RVO and NRVO are even better than move-semantics.
Using `std::move` to a variable before it is returned prohibits RVO or NRVO, thus, making the program slower.

Even if RVO and NRVO are not applicable in certain cases. The compiler is smart enough to invoke the move constructor of
the variable to construct the `return` statement. A copy constructor will only be used a move constructor is not available.

## Use `std::move` to `const` objects
Consider the following code:
```cpp
  const std::vector<cv::Point2i> box_corner_points =
      GetAffinedBoxImgIdx(0.0, 0.0, M_PI_2,
                          {
                              std::make_pair(param.front_edge_to_center(),
                                             param.left_edge_to_center()),
                              std::make_pair(param.front_edge_to_center(),
                                             -param.right_edge_to_center()),
                              std::make_pair(-param.back_edge_to_center(),
                                             -param.right_edge_to_center()),
                              std::make_pair(-param.back_edge_to_center(),
                                             param.left_edge_to_center()),
                          },
                          0.0, 0.0, 0.0);
  cv::fillPoly(
      *img_feature,
      std::vector<std::vector<cv::Point>>({std::move(box_corner_points)}),
      color);
```
In the above code, `box_corner_points` is first created as a `const` variable, then
it is moved to construct a temporary. The intention of the programmer is to use the
move constructor to initialize the temporary. But there are two problems regarding this kind of usage of `std::move`.
- First, use `std::move` to `const` variable is semantically problematic. A `const` variable should not
change its value during its lifetime. Though there are too many ways to circumvent constness constraints in C++,
we generally should avoid break const-correctness unless there are good reason to do so. In this case,
if we know that an object will be a moved-from one in subsequent code, we should not define it as being `const`.
- More importantly, the code does not do what the author intended to. `std::vector` has a copy constructor which takes
another vector by `const &`. The move constructor of `std::vector` takes another vector by **non-const** ravlue reference.
So, the above code actually calls the copy constructor. **More words on this: though it is technically possible to write
a copy constructor that take non-const lvalue reference, or write a move constructor that takes const rvalue reference.
The established convention  is to use a pair of `T(const T&` and `T(T&&) for type `T`**. Breaking this convention
leads to confusing code that most C++ programmers would not expect.

## Bad floating point comparison
Consider the following code:
```cpp
template <class T>
typename std::enable_if<!std::numeric_limits<T>::is_integer, bool>::type
almost_equal(T x, T y, int ulp) {
  // the machine epsilon has to be scaled to the magnitude of the values used
  // and multiplied by the desired precision in ULPs (units in the last place)
  // unless the result is subnormal
  return std::fabs(x - y) <=
             std::numeric_limits<T>::epsilon() * std::fabs(x + y) * ulp ||
         std::fabs(x - y) < std::numeric_limits<T>::min();
}
```
This function does one simple thing: to compare two floating point numbers to see whether they are close enough up to an given scaling factor.
How hard could be that, you may ask. Let us find out:
- First notice that the scaling factor could be zero or negative, which obviously is a mistake.
- Second, `std::fabs(x+y)` could cause floating point. When overflow happens, the expression will be valued as `+Inf` as the IEEE 753 standard mandates.
- The condition `std::fabs(x-y) < std::numeric_limits<T>::min()` is actually equivalent with `x == y`. Since the right side is  the smallest floating number representable
by the iEEE 753 standard. Because of [denormalized number](https://en.wikipedia.org/wiki/Subnormal_number), the only way to make a number smaller than the least representable
number is that the number is 0. This check is thus redundant.
- Because multiplying a number by a very small number such as `epsilon` reduces its precision. When comparing two numbers, it is better to use division than subtraction.
That is, for two floating numbers `a` and `b`, `fabs(a/b -1) < epsilon` is better than `fabs(a - b) < epsilon`. The latter, when comparing two large numbers, will result
in confusing results.
- Because epsilon is very small, for two numbers that both are near-zero very small numbers, the above function (unless `ulp` is very large)
will erroneously say they are not almost equal. Remember that `epsilon ≅ 2.2204e-16`. This can get pretty bad especially when x ≅ -y.
- The function fails to check if one or both of the parameter are ±Inf or NAN.
- At last, there is no need to write a function template here. C++ can implicitly convert a `float` to `double`.

The correct way to implement the function is like below:
```cpp
double almost_equal(double x, double y, unsigned int ulp =1){
	assert(std::isfinite(x) && std::isfinite(y));
	return fabs(x - y) <= std::numeric_limits<double>::epsilon * std::max(ulp,  1)
			|| fabs(x/y -1) <= std::numeric_limits<double>::epsilon * std::max(ulp,  1);
}
```

When `x` and `y` are near zero, the comparison is handled by  `fabs(x - y)`, otherwise, it is handled by `fabs(x/y -1`.
The latter comparison will lead to the correct result even if `y==0`.
We also have to make sure that `ulp >=1`.

## Unnecessary `memset`
Consider the following code:
```cpp
  float flipped_anchors_dx[kNumAnchor];
  float flipped_anchors_dy[kNumAnchor];
  memset(flipped_anchors_dx, 0, kNumAnchor * sizeof(float));
  memset(flipped_anchors_dy, 0, kNumAnchor * sizeof(float));
```
Arrays are aggregate types in C++. `float my_array[MY_SIZE]{}` or `float my_array[MY_SIZE] = {}`
leads to all elements of the array are set to zero.

Two things to notice here:
- Putting `memset` in code does not mean `memset` will be called in some cases, because the compiler might optimize it away.
If this is not what you want, see [this post](https://bylizhao.github.io/c/programming/2020/08/15/Forcing-memset-to-be-executed-against-compiler-optimization.html).
- For aggregate types other than arrays, it is behavior is more complicated because padding bytes also need to be considered,
see [this post](https://bylizhao.github.io/c++/programming/2020/08/27/What-happens-to-padding-bytes-during-initialization.html) for details.

**Note for C programming, one should use `float my_array[MY_SIZE] = {0} to define an array with all elements being zero.**
