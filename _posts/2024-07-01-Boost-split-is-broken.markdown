---
layout: post
mathjax: true
comments: true
title:  "Boost::split is broken"
author: John Z. Li
date:   2024-07-02 15:00:18 +0800
categories: c++
tags: c++, boost, split, tokenize
---
Several days ago, I was bitten by this weird behavior of `boost::split`: if the
input string is empty, it puts an empty string into the output vector with the
following code.
```cpp
    std::vector<std::string> result;
    boost::split(result, "", boost::is_any_of(","), boost::algorithm::token_compress_on);
	assert(result.size() == 1u);
```
This is really bad because it violates the Null-to-Null correspondence, which is
a term coined by me to express the idea that for algorithms like `split`, the expected
behavior by most programmers, I believe, should always be mapping an set to an empty
set. Ironically, the boost documentation falsely claims that "This function is equivalent to C strtok",
but `strtok` does not have this problem.

If one looks more closely, `boost::split` is actually broken in more than one way.
1. First, it potentially triggers a lot of memory allocations.
If a `vector<string>` is used as the output parameter as the example in the begining,
Each string of the vector will likely trigger a memory allocation, as well as the
vector itself.
If a `vector<iterator_range>`, only the vector itself will be allocated.
Interestingly, the vector allocation can not even be save by calling `reserve` on
the vector, because inside `boost::split`, a new vector is allocated and then
swapped with the passed-in vector.
2. Second, `token_compress_on` should be the default.
Empty tokens are seldom what one needs.
Many realworld input data might contain redundant sperators.
3. Third, `boost::split` splits aggressively,
meaning it splits the whole input string all at once.
For potentially very long input strings,
this is usually not what one wants, as "stop at the first error" is
a commonly used strategy when doing tokenization.

On the other hand, if we check `strtok` in C's standard library, it has none of this
issues:
1. If the input string is an empty string (a null pointer in C), the returned string
is guaranteed to be empty. The Null-to-Null correspondence is preserved.
2. It ignores consecutive seperators, in `boost` terms, `token_compress_on` is the
default behavior. Actually, it is the only behavior, which I think makes sense.
3. It does its job lazily, calling the function again, you gets the next token.
You can stop at any step as soon as you found an error in a token.
4. It does not allocate inside. If the caller wants to copy a token, the can
always do it explicitly.

There are, though,  drawbacks of `strtok`:
1. It is not thread safe because each call of the function updates a static variable.
(This, obviously can be fixed by using a thread local variable instead. Not sure
why this has not been done.)
2. It is destructive. It writes `\0` characters in the input string.
This is unfortunate because a pointer-size pair could be returned instead.
The design constraint that leads to this is that C strings must be null terminated.

Lesson learnt: a function with the signature
`std::string_view strtok(std::string_view str, std::string_view)` with the semantic
of C's `strtok` utilizing a function scope thread local variable to store `str`, is
both clearer and more efficient than `boost::split`.

