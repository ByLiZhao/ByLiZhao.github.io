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
input string is empty, it puts an empty string into the output vector.
With the following code, the result vector is not empty but contains an empty string.
```cpp
    std::vector<std::string> result;
    boost::split(result, "", boost::is_any_of(","), boost::algorithm::token_compress_on);
	assert(result.size() == 1u);
```
This is really bad because it violates the Null-to-Null correspondence, which is
a term coined by me to express the idea that for algorithms like `split`, the expected
behavior by most programmers, I believe, should always be mapping an empty string to an empty set.
Splitting nothing is intuitively a Non-op, resulting nothing, an empty vector in this case.
Ironically, the boost documentation falsely claims that "This function is equivalent to C strtok",
but `strtok` does not have this problem.

If one looks more closely, `boost::split` is actually broken in more than one way.
1. First, it potentially triggers a lot of memory allocations.
If a `vector<string>` is used as the output parameter as the example in the beginning,
Each string of the vector will likely trigger a memory allocation, besides the allocation of the
vector itself.
If the output type is  `vector<iterator_range>`, only the vector itself will be allocated.
Interestingly, the vector allocation can not be avoided by calling `reserve` before passing the vector to `boost::split`.
The reason is that, inside `boost::split`, a new vector is always allocated and then
swapped with the passed-in vector. This is the price `boost::split` pays for exception safety.
Even if the output vector already has large capacity, `boost::split` can not predict the eventual vector size.
It also can't assume that memory allocation will be always successful.
So it plays safe by allocating a new vector, and swap it with the passed-in vector before returning.
If an exception is thrown in the middle, when the caller captures the exception,
it will still have the original vector untouched.
But if exception safety is so important, why not just return a vector by value?
The pass-by-non-constant-reference output vector is misleading here.
The only purpose it serves is to specify the type of the output parameter.
2. Second, `token_compress_on` should be the default.
Empty tokens are seldom what one needs.
When designing an API, the most common case should be made default, while less used feature should be made configurable.
Take comma separated tokens as an example.
For a input string like `1, 2, 3, 4,`, we usually don't want the trailing comma to lead to a trailing empty token.
When you reads from a configuration file, or handle user input, the presence of trailing separators should be tolerated in most cases.
3. Third, `boost::split` splits aggressively,
meaning it splits the whole input string all at once.
For potentially long input strings,
this is likely not what one wants, as "stop at the first error" is
a commonly used strategy when doing tokenization.

On the other hand, if we check `strtok` in C's standard library, it has none of this
issues:
1. If the input string is an empty string (a null pointer in C), the returned string
is guaranteed to be empty. The Null-to-Null correspondence is preserved.
2. It ignores consecutive separators, in `boost` terms, `token_compress_on` is the
default behavior. Actually, it is the only behavior, which I think makes sense.
3. It does its job lazily, calling the function again, you gets the next token.
You can stop at any step as soon as you found an error in a token.
4. It does not allocate inside. If the caller wants to copy a token, the can
always do it explicitly.

There are, though,  drawbacks of `strtok`:
1. It is not thread safe because each call of the function updates a static variable.
2. It is destructive. It writes `\0` characters in the input string to separate tokens.

C's lack of native OOP support is affecting API design here.
- Instead of using a static variable in the function to save intermediate state,
  a small object that encapsulates the same information, is obviously a better design.
  Besides thread safety issues, the static variable makes the function not reentrant.
- Except returning a C string, a pointer-size pair could be returned instead.
  This way, the input string can be left unchanged.
  Moreover, the caller does not have to call `strlen` to get the length of the returned token.
  How many CPU cycles have been wasted on this kind of  senseless byte counting because a function
  does not want the caller to the size of returned strings.

A better design is something like below, still pure C:
```c
struct tok_state{
	const char * ptr;
	const char * delim;
};

struct token{
	const char * str;
	const size_t size;
}

// new strtok signature
struct token strtok(struct tok_state*);

// used like below
const char *p = "1,2,3,4";
struct tok_state ts{.ptr = p, .delim = ","};
struct token tk = strtok(&ts);
```

For C++, state can be wrapped in a `tokenizer` object, while `operator()` is overload to return the next token, like below:
```c++
class tokenizer {
    std::string_view input_;
    std::string_view delim_;

public:
    constexpr tokenizer(std::string_view input,
              std::string_view delim) noexcept
        : input_(input), delim_(delim) {}

	// return empty string means no more token
    constexpr std::string_view operator()() noexcept
    {
		constexpr auto npos = std::string_view::npos;

        // skip leading delimiters.
        auto first = input_.find_first_not_of(delim_);
		// update input to exclude leading delimiters
        input_.remove_prefix(first == npos? input_.size() : first);

        auto last = input_.find_first_of(delim_);
        auto token = input_.substr(0, last); // substr handles last == npos
        input_.remove_prefix(token.size());

        return token;
    }

	constexpr explicit operator bool() const noexcept{
		if(input_.empty()){
			return false;
		}
		else{
			auto first = input_.find_first_not_of(delim_);
			return first != npos;
		}
	}
};

// used like this
tokenizer tok("1, 2, 3, 4,", ", ");
while(tok){
	auto sv = tok();
}
```
Indeed, a little bit of OOP makes API better.

Note that C++20 introduces a `std::ranges::views::split` class template,
but it is totally a different beast.
The biggest semantic difference is that it treats the delimiter as a single pattern, as if doing sub-string match.
For example, the below will output: `"Hello" "^C++" "^20" "^!"`, instead of `"Hello" "C++" "20" "!"`.
```cpp
// this example is from website cppconference with some adaption
int main()
{
    using std::operator""sv;
    constexpr auto words{"Hello^_^C++^_^20^_^!"sv};
    constexpr auto delim{"^_"sv};

    for (const auto word : std::views::split(words, delim))
        // with string_view's C++23 range constructor:
        std::cout << std::quoted(std::string_view(word)) << ' ';
    std::cout << '\n';
}
```
Moreover, empty ranges are part of its semantic contract, which means:
- split an empty string yields one empty token.
- split `","` with delimiter `","`, produces 2 empty tokens.
- split `"a,,b"` with delimiter `","` produces `"a", "", "", "b"`.
The contract is that if you join tokens together with the same delimiter, you will get the original view.
Though returning one empty token on empty input is still a design quirk,
with `std::ranges`'s fluent API, you can easily filter empty results out if you want like
{% raw %}
```cpp
auto tokens =
    ",1,2,3,4,"sv
    | std::views::split(',')
    | std::views::filter([](auto&& r) {
          return !std::ranges::empty(r);
      });
for(auto sub_range : tokens){
	std::string_vew sv{sub_range}; // only for C++23
}
```
{% endraw %}
A good thing of `std::ranges::views::split` is that it works with any range.
It also avoids allocation and is lazily executed.
For simple tasks like string splitting, if its behavior is what you want, it is enjoyable to use.

There is also `boost::tokenizer`. It is highly customizable.
With some tweaks, you can inject complex splitting logic into it.
Though I would argue that its flexibility is also its biggest weakness.
For simple tasks, it is two heavy-weight.
For complex tasks, you would be better off with having a real parser.
If your splitting logic is no unique, it is clearer to write it in parsing rules.
Modern parser combinator can generate the actual code for you.
