---
layout: post
mathjax: true
comments: true
title:  "Reducing usage of macros in C++ programming"
author: John Z. Li
date:   2021-11-30 21:00:18 +0800
categories: c++
tags: macro
---
The problems with macros are well understood by C and C++ programmers. Basically,
the preprocessor operates on the source code independently and before the compiler
has a chance to check the code. Macros are basically textual replacement instructions
that does not understand the syntax of the host language (C or C++). Macro expansion
can also be affected by defining or not defining certain environmental variables or
the configuration of the building phase. This poses challenges for certain things like static analysis and debugging to
be done in C and C++. But with C, macros are the only thing that can make certain things easier.
With C++, we generally should use less macros if there are other ways to achieve what macros can do.

## Macros that can be replaced with modern C++ features
### Macros that can be replaced with `constexpr` expressions
Consider the following macros:
```cpp
#define PI 3.14
#define TWO_THOUSAND  1000*2
#define FILENAME "my_file_with_a_very_long_name.hpp"
```
This kind of macros can be replaced by `constexpr` expressions:
```
constexpr auto Pi = 3.14;
constexpr auto Two_thousnd = 1000 * 2;
constexpr auto Filename = "my_file_with_a_very_long_name";
```
If these definitions are to be used in multiple translation units, one can also apply the `inline` keyword to them (After C++17).
Sometimes, macros are used to introduce an alias of a function, for example
```cpp
#define PRINT printf
```
This kind of aliases can also be replaced with  simple `constexpr` expressions. The above code can be replaced by
```cpp
constexpr auto Print = printf;
```
### Macros that can be replaced with lambda functions
Consider the following example:
```cpp
#define CIRCLE_AREA = PI * R * R
//
int main()
{
	double R = 10;
	double area = CIRCLE_AREA;
}
```
Here, the macro `CIRCLE_AREA` refers to a variable `R` that is to be defined before the invoking site of the macro.
This kind of macros act like lambda functions in that they "capture" the context where they are invoked.
Thus, this kind of macros can be easily replaced by
```cpp
// assuming Pi has already been defined
int main()
{
	double r = 10;
	auto circle_area = [&r](){return Pi * r * r};
	double area = circle_area;
}
```
### Macros that can be replaced with `using` declarations or `typedef`s.
Consider the following example:
```cpp
#define RECORD vector<int>
#define SIZE_T unsigned int
#define RECORD_PRT RECORD*
```
This kind of macros can be replaced by so-called type alias:
```cpp
using Record = vector<int>;
using Size_type = unsigned int;
using Record_ptr = Record *;
```
In C, one often needs to use macros to achieve generic data structures. The following code
is taken from "openbsd/src/sys/sys/queue.h":
```c
/*
 * Singly-linked List definitions.
 */
#define SLIST_HEAD(name, type)						\
struct name {								\
	struct type *slh_first;	/* first element */			\
}

#define	SLIST_HEAD_INITIALIZER(head)					\
	{ NULL }

#define SLIST_ENTRY(type)						\
struct {								\
	struct type *sle_next;	/* next element */			\
}
```
In C++, generic programming is achieved by templates. To give a new name to a template,
we can use the so-called *alias template* like below:
```cpp
template<typename T>
using List = std::list<T>;
```
`typedef` can also achieve the same purpose, but it has to be wrapped in a struct template, thus more verbose:
```cpp
template<typename T>
struct List{
	typedef std::list<T> type;
};
```
### Function-like Macros that can be replaced by function templates
Consider the following typical usage of macros:
```cpp
#define MIN(A, B) ((A) < (B) ? (A) : (B))
```
Since we don't know anything about `A` and `B`, like whether they are values or references,
whether they are `const` qualified, whether they are lvalues or rvalues, we have to use
perfect forwarding along with `decltype` on the return value in function templates to mimic its behavior:
```cpp
template <typename T1, typename T2>
inline auto MIN(T1&& A, T2&& B)
-> decltype(((A) < (B) ? (A) : (B)))
{
return ((A) < (B) ? (A) : (B));
}
```
The `inline` keyword here can be omitted if we are only going to implicitly instantiate the function template.
In this case, it is implicitly declared as `inline`.

Similarly, variadic macros can be replaced by variadic templates.
Consider the following macro definition:
```cpp
// use compiler extension, so that when __VAR_ARGS is empty, there is no trailing comma
#define debug(format, ...)  printf(format, ## __VA_ARGS__)
// or more portalbe way
#define debug(format, ...) printf(format, __VAR_OPT(,) __VAR_ARGS__)
```
The above code can be replaced by the following variadic function template:
```cpp
template <typename... Args>
inline void debug_str(const char * fmt_str, Args&&... args) {
    printf(rt_fmt_str, std::forward(args...));
	return;
}
```
**Note: don't use this code in real projects. Use `std::format` introduced in C++20, or `fmt` from which `std::format` is modeled.**

If a function-like macro calls an function object or a lambda function which is
only accessible from the context where the macro is invoked, it should be replaced
by a lambda function as discussed in a previous section.
The lambda function itself might needs to be generic or even be templated,
Lambda templates are a feature introduced in C++20 while generic lambda is a C++14 feature.
In case the macro is variadic, its replacements need also to be variadic.

### Get information of source code information
The preprocessor recognizes the following macros (**note `__func__` is not a macro**):
- `__FILE__`, which expands to the file name of the current source file.
- `__LINE__`, which expands to the current line number of the current source file.
- `__FUNCTION__`, (this is a compiler extension) which expands to the name of the current function.
- `__PRETTY_FUNCTION__`, (this is a compiler extension) which expands to the function name with parameter list of the enclosing function. This is handy to check the type information of a specific function template instantiation.
- '__func__', (after C++11) expands to the function name of the enclosing function.
C++20 introduced `std::source_location`. Now the above information can be retrieved by these new facility.
`std::source_location` contains the following methods:
- `file_name()`, the return value of which can replace `__FILE__`.
- `line()`, the return value of which can replace `__LINE__`.
- `column()`, the return value of which is the column at which the calling function invokes `std::source_location::current()`.
- `function_name()`, the return value of which can replace `__PRETTY_FUNCTION__`.
The new facility is actually more powerful than the old macro-based solutions, because now if we have the following `log` function:
```cpp
void log(const std::string_view message, const std::source_location location = std::source_location::current());
```
Every time we call the `log` function, the information related with `source_location` is that of the calling site.
That is, if `log` is called inside a function named `foo`, `function_name()` will return the name of `foo`.

## Macros that can be partially replaced by C++ features
### Conditional compilation
C++17 introduced `if constexpr`, which can replace some use cases of conditional compilation using `#if` and friends.
An example of this is like below:
```cpp
if constexpr (DEBUG_LEVEL_INFO)
    {
        log(\* do some logging if only debug level is set to Info*\);
    }
```
This kind of stuff is traditionally done by macros, for example from [folly/logging](https://github.com/facebook/folly/tree/main/folly/logging)
>>disabled log statements should boil down to a single conditional check.
Arguments for the log message should not be evaluated if the log message is not enabled.
Unfortunately, this generally means that logging must be done using preprocessor macros.

But `if constexpr` can not replace or use cases of conditional compilation, because even if a branch in
a `if constexpr` statement is going to be 'optimized' away in compile-time because the condition is an `constexpr` equals `false`,
the code inside that branch still needs to be valid code. This means, `if constexpr` can not be used to deal with platform specific things,
like calling a platform specific function only when built against that platform.

### Increasing code readability and easing maintainability using macros.
Generally, macros tend to make code less readable, and can cause maintainability headache if over used.
But sometimes, using macros can actually make code more readable and easier to maintain.

1. An example of this is [X macros](https://en.wikipedia.org/wiki/X_Macro).
The idea is to use macros to generate boilerplate code, which, if written by hand,
can be error-prone or boring.

2. Another example is like above
```cpp
    #define AUTO_COUT(x) {\
     boost::io::ios_all_saver ias( cout );\
     x;\
     }while(0)
```
This code is used to restore `ostream` format settings back to the original state,
so that things like `std::hex` stops in effect after it is being called.

3. Yet another example is when you want to specialize `std::greater<>` and friends
on types that don't have comparison operators defined like below:
```cpp
#define MAKE_COMP(class_name, comparison, field, symbol)                       \
  template <> struct std::comparison<class_name> {                             \
    bool operator()(const class_name &lhs, const class_name &rhs) {            \
      return lhs.field symbol rhs.field;                                       \
    }                                                                          \
  };

struct person {
  unsigned int age;
  std::string name;
};

MAKE_COMP(person, greater, age, >)
MAKE_COMP(person, less, age, <)
MAKE_COMP(person, equal_to, age, ==)
MAKE_COMP(person, not_equal_to, age, !=)
```
There will be quite some code to write if we are going to write these code manually.
Maybe after static reflection is introduced in C++, such repetitive code can be generated without macros.
4. The `FWD` macro and `LIFT` macro
I won't repeat what is in this good [blog post](https://blog.tartanllama.xyz/passing-overload-sets/).
The point here is that using macros can sometimes make your code more readable.
This kind of things can be done without macros at the cost of writing more boilerplate code.
Maybe new features added into the language in the future will reduce macro usage.

## Things that can only be done via macros (up till now)
### Language feature test.
There are a whole lot of [feature test macros now in C++](https://en.cppreference.com/w/User:D41D8CD98F/feature_testing_macros).
Their sole purpose is to determine whether a certain language feature is supported by the compiler.
Currently, this can only be achieved by macros.
### Stringification
Stringification is the ability to treat any sequence of characters as a compile-time `const char` array, that is C-style string.
For example:
```cpp
#define STR(x) #x

auto enum_name = STR(Color::Green);
auto class_name = STR(my_namespace::my_class);
auto type_name = STR(vector<bool>);
```
It is equivalent to:
```cpp
const char* enum_name = "Color::Green";
const char* class_name = "my_namespace::my_class";
const char* typename = STR(vector<bool>);
```
This itself does not seem very impressive.
But it can be a powerful tool when it is combined with other macro techniques like X-macros.
For example, you can define the following `enum_to_string` function like below so that it can
return a string that contains the `enum` value's corresponding name and its value:
```cpp
#include <iostream>
#include <string>

#define COLOR \
  X(Green, 1) \
  X(Blue, 2)  \
  X(Red, 4)

#define X(x, ...) x __VA_OPT__(= __VA_ARGS__),
enum class Color { COLOR };
#undef X

std::string enum_to_string(Color c) {
  std::string enum_name, enum_value, enum_str;
  switch (c) {
  #define X(x, ...) \
    case Color::x: \
      enum_name = "Color::" #x;   \
      enum_value = std::to_string(static_cast<int>(c)); \
      break;
    COLOR
  #undef X
  }
  enum_str = enum_name + "=" + enum_value;
  return enum_str;
}
```
The beauty of this is that whenever the definition of the macro is changed, like
adding a new element to the `enum`, one only needs to change the definition of `COLOR`.
Currently, this trick can not work without macros. This will change after static reflection
is added to the language.
### Batch generating identifier names
In C and C++, macros can be concatenated using the `##` operator.
This is extremely useful when we need to generate many different identifier names.
An example is `PYBIND11_EMBEDDED_MODULE` and friends in [pybind11](https://github.com/pybind/pybind11).
Basically, to call C++ library from Python, we need to create functions like `extern "C" PyObject *pybind11_init_impl_##name();`
with different names for different libraries. It will be very inconvenient if the users are
asked to write these function names by hand. C++ test frameworks also heavily rely on this trick
to automatically generate many identifier names that one would not bother to give names manually.
