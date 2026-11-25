---
layout: post
mathjax: true
comments: true
title:  "How C++ string class can be made better"
author: John Z. Li
date:   2020-12-24 19:00:18 +0800
categories: c++ programming
tags: string, small-string-optimization
---
By the title, I do not intent to nitpick on the design of C++’s string class,
or more precisely, the class template family: `std::string`, `std::wstring`, `std::u8string`, etc,
and their `std::pmr` counterparts. The term "string class" in its singular form
signifies the intention to view C++'s standard string facility as whole, taking a bird's
view of its semantics and behavior.
Discussion in this post should be regarded as thought experiments.
It tries to explore possible design freedom regarding C++'s strings.
Note they are not language feature proposals that I think C++ could pursuit,
but rather answers to the hypothetical question: What could have been done differently
if we had the chance to start with a clean slate.
Ideas presented in this post are language design ideas, and not necessarily related with C++.
C++ is used as a framework of reference here.
As C++ is a performance oriented language, and is already rich in performance-aware
language features, if an idea can potentially optimize C++, it might bears some weight.
Here we go.

# Extend the concept of string literals
First, let us start with string literals.
A string literal is a string of face value which is fully determined at compile-time.
C already had this in the beginning.
When C++ was invented, it could have chosen to generalize the concept.
Something that is easy to achieve is,
that strings initialized from string literals should not
incur runtime cost.

With C, the following code runs cheaply as it can be:
```c
    const char str[] = "some string literal";
```
But C has its drawbacks regarding string literals, that is,
after the initial assignment, form a perspective of type system,
there is no way to know whether a C string is a string literal or a runtime string.
Not perfect, but C's designers took a minimalist's approach with their language.
They never expect you to do advanced string manipulation in compile-time.

However, with C++, the following code makes an extra copy of the string literal,
often leads to an extra heap allocation.
```cpp
    const std::string s = "some string literal";
```
For now, let us not quickly jump to the "This is sub-optimal for performance" wagon.
Though it surely is, the more important issue is loss of  type information here.
After creation of `std::string`, you no longer know that the string is supposed to be a compile-time constant.
Note that I am not saying that C++ is inconsistent here. As `s` is a newly created object,
Although it copies its content from a string literal, it is not a string literal itself.
The point I try to highlight is that this is a wasted opportunity.

Considering that the compiler already knows string literals internally,
C++ could have a way to let the compiler to pass the identity to a string.
Hypothetically, it could have introduced a compile-time string class, say, it is called `std::static_string`.
With this explicitly named new class, you have have the following code:
```cpp
// Note this not valid C++ code
// This is code with a hypothetical new language feature
auto str1 = "Name: "; // this is a string literal
auto str2 = "John";   // this is also a string literal
auto str = str1 + str2;  // "Name: John", as if you typed it out literally
```

Two things one immediately notice is that:
1. String literals now have their own type. The type system guarantees that a string of this type is a compile-time constant.
2. You can do compile-time computation on string literals, including creating new compile-time strings.
   This would be `consteval` before it became a thing but more powerful, only for strings.

Once you realize that there could be a whole new universe of string algebra available at compile-time,
you open gates to many interesting use cases, for example:
1. `std::format` could specify that its first parameter should be of the type `std::static_string`.
   As a result, it does not have to template on `const char[N]` in `consteval` context.
2. `Boost.Hana` can just use `std::static_string` as non-type template parameters.
   No trick like `hana::string<hana::char_c<'a'>, hana::char_c<'b'>> ` is needed.
   Instead, one can just specialize a class template on `"ab"`.
3. Use `std::static_string` as keys or values in hash tables. This is safer at type system level than, say, using `constexpr std::string_view`.
4. Parse configuration stored as string literals in compile-time (with `constexpr` or `consteval` functions).

C++20 has made `std::string` usable in constant evaluation.
This can cover some use cases as listed above, but not all.
- `std::string` can not be used as valid NTTP (non-type template parameter).
  A workaround is to use wrapper classes templated on `const char[N]`.
- `std::string` must not escape constant evaluation. For example, you can't return another compile-time string from a `consteval` function
  and makes it usable from other parts of your code.

As modern C++ relaxes further on constraints of `consteval` and `constexpr` functions.
More things become possible, or easier without work-around.
But the core limitations remains, and are not likely to change.
As this is the result of a decision made long ago when the inventor of C++ decides the type of string literals
are `const char[N]` as in C.
Maybe, C++ being compatible with C should not have meant inheriting all C's quirks unconditionally.

# Strings should be able to made immutable
C++ has `const`, but it is not the same as immutability.
A `const std::string` object might still get altered as `const` in C++ only means shallow constness,
meaning the underlying buffer can still be modified (you know how).

Note that I am not making a generic criticism against C++'s `const` semantics,
but only advocating for an extension to the language that makes real immutability feasible.

Assuming we have this special C++ rule (note this is not standard C++):
```cpp
// For a string s, "const s" means constness propagates to its data
// as if the class is defined as below:
class string{
	[[propagate_const]]
	char * data;
	std::size_t size;
};
```
The hypothetical `propagate_const` attribute means top level constness will be
applied to the member `data` too.

With this, we can strictly define the `value type` of `std::map<std::string, std::string>`
as `std::pair<const std::string, std::string>`.
Note it is already the case signature-wise, but with different guarantees.
`const std::string` as the key type in pair means a key would never be modified.
Right now, C++ is doing it from convention. `const` is only a strong suggestion to
other people that you really should not try to alter an object.
Let the type system to prevent objects being modified when they should not be is a nice thing to have.

Compilers can do advanced optimization based on the guarantee that a `const std::string` will never
change. If the compiler knows that the value of an object won't change during its lifetime, it can
reschedule memory `load` or reduce some redundant `load`s, and as a result, some unnecessary conditional checks.
Since strings are such commonly used types, making strings fast is important for performance and correctness.

# Promote small strings to stack when possible
It is a long known fact if you can allocate small objects on the stack, instead of on the heap,
it is good for performance. C++ is introducing more of types that can use stack allocation, like `std::inplace_vector` (C++26).

Using a stack array as the backing memory of a `std::pmr::monotonic_buffer_resource`,
`std::pmr::string` can use stack allocation. The problem is `std::pmr::string` is a different type than `std::string`.
Unless you control all the source code of a project, you generally can't expect that
functions/methods written by other developers will accept `std::pmr::string` as parameters.
If they don't, whatever performance gain you have from using `std::pmr::string` will be lost on type conversions and
data copying.

Some C++ programmers tend to solve C++'s problem by introducing yet another type, or even another class template.
The problem with this is that type proliferation creates the need for type conversion.
A classical example is `std::char_traits` that let you define case insensitive strings.
But as long as you do that, you will find that you need to compare instances of your case insensitive strings with `std::string`.
The convenience or elegance of API will be lost on endless type conversions.
That is why people tend not to do that.

So, instead of switching to a new string type, stack allocation should be done in a type-transparent fashion, like below:
```cpp
// Note this not valid C++
std::string [[stack_alloc]] s = "This is a string that is too big for Small String Optimization, but not too big for stack allocation";
func(s); // func takes const std::string&
```
The type of `s` is still `std::string`. Anyone except the one who wrote this piece of code, does not need to know that this string is stack allocated.
The signature of subsequent functions don't have to change. Actually, they don't bother to know where the string is allocated.
The implementation should set an upper limit on how much you can allocate on stack for an object.
For strings, we can start small, say, the default maximal size for stack allocated strings is set to 128 bytes.

To achieve this, the requirements for allocators need to be changed.
A standard compliant allocator must expose a hook to the compiler, through which, the compiler will set a pointer as "non-owning" for the allocator.
This can be done by tagged pointers. When the allocator later sees an "non-owning" pointer, its deallocation operations reduces to a "No-op".

This is something that can be hacked by creating another type or even another library.
The mechanism must be built into the compiler and exposed to common users as no more than in-line small annotations.
When it becomes friction-less to do something, people tend to do more of it.

As long as we have this infrastructure ready, a more radical approach can be explored.
For example, let the compiler do Escape Analysis, and automatically apply the stack allocation optimization when it sees feasible.
My sense is that there is quite some potential performance gain hidden in there.
Modern OSes have already provided API to set stack size for a thread.
If the compiler has an estimation of how much stack space will be needed after applying the optimization,
it can insert calls to increase stack size into program in initialization phase.
