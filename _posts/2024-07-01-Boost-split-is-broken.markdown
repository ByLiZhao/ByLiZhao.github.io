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
input string is empty, it puts an empty object into the output vector. This means
```cpp
    std::vector<std::string> result;
    boost::split(result, "", boost::is_any_of(","), boost::algorithm::token_compress_on);
	assert(result.size() == 1u);
```
This is really bad because it violates the Null-to-Null correspondence, which is
a term coined by me to express the idea that for algorithms like `split`, the expected
behavior by most programmers, I believe, should always be mapping any empty set to an empty
set. Ironically, the boost documentation falsely claims that "This function is equivalent to C strtok".

If one looks more closely, `boost::split` is actually broken in more than one way.
First, it potentially triggers a lot of memory allocations, one for each string in
the output vector if one uses `vector<string>` types. This can be avoided by
using `vector<iterator_range>`, though it still might allocate for the vector. The
vector can reserve additional capacity before it is passed to `boost::split`, but
it is unclear how much memory should be preserved beforehand. Second, `token_compress_on`
should be the default, for empty tokens are seldom what one needs, as many realworld
input data might contain redundant sperators. Third, `boost::split` splits aggressively,
meaning it splits the whole input string until the end. For potentially very long
input strings, this is usually not what one wants, as "stop at the first error" is
a commonly used strategy when doing tokenization.

On the other hand, if we check `strtok` in C's standard library, it has none of this
issues:
1. If the input string is an empty string (a null pointer in C), the returned string
is guaranteed to be empty. The Null-to-Null correspondence is preserved.
2. It ignores consecutive seperators, in `boost` terms, `token_compress_on` is the
default behavior. Actually, it is the only behavior, which I think makes sense.
3. It does its job lazily, calling the function again, you gets the next token.
You can stop at any step as soon as you found an error in a token.
4. It does not allocate inside.

There are, though,  drawbacks of `strtok`:
1. It is not thread safe because each call of the function updates a static variable.
(This, obviously can be mitigated by using a thread local variable instead.)
2. It is destructive, writing `\0` characters in the input string. The reason for
this is that C strings must be null terminated.

Lesson learnt: a function with the signature
`std::string_view strtok(std::string_view str, std::string_view)` with the semantic
of C's `strtok` with a function scope thread local variable to store `str`, is
lightweight and efficient.

