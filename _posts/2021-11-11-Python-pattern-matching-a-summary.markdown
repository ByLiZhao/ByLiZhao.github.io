---
layout: post
mathjax: true
comments: true
title:  "Python pattern matching: a summary"
author: John Z. Li
date:   2021-11-11 19:00:18 +0800
categories: python
tags: python, dictionary
---
Python of version 3.10 onward has introduced a feature called pattern matching.
Basically, it works like pattern matching in other languages.
Though, there are  a few things that needs to be paid attention to.
## Match by values
We can pattern matching by explicitly specify the values we want to match against.
For example, with the following code:
```python
from fstring import fstring
from enum import Enum

class week_day(Enum):
    Sunday = 0
    Monday = 1
    Tueday = 2
    Wednesday = 3
    Thursday = 4
    Friday = 5
    Santurday =6

def match_value(e):
    match e:
        case 1:
            print("I found integer 1.")
        case True:
            # does not work here
            print("I found Boolean true.")
        case '1':
            print("I found string '1'.")
        case (1, ):
            print("I found tuple (1,)")
        case [1]:
            # the same with (1,)
            print("I found list [1]")
        case week_day.Monday:
            print("I found Monday")
        case [1, *tail]:
            print("I found element 1 as the head the list")
            print(f"the tail of the list is {tail}")
        case [*head, 1]:
            print("I found element 1 at the end of the list")
            print(f"the head of the list is {head}");
        case [_, 1, _]:
            print("I found element 1 in the middle of the list")
        case _:
            print("I don't understand.")
```
If we call the `match_value` function as the below, we get:
```python
match_value(1)  # matches to integer 1.
match_value(True) # matches to integer 1 or Boolean True
match_value("1")  # matches to string '1'
match_value((1,)) # matches to (1,) or [1]
match_value([1])  # matches to (1,) or [1]
match_value(week_day.Monday) # matches to weekday.Monday
match_value([1, 2, 3])  # matches to [1, *tail]
match_value([2, 3, 1])  # matches to [*head, 1]
match_value([2, 1, 3])  # matches to [_, 1, _]
match_value([2, 4, 1, 3, 5]) # matches to _
```
To performing pattern matching against values, literals and constants actually,
Python compare the value expression `e` with the value after the `case` keyword using the `==` operator.
Note that the underline `_` means any value.

There are few things to notice though:
- Because `True == 1` is evaluated to `True` in Python, in the above example, `match_value(True)` will falls to
the `case 1` branch.  To make the matching type-aware, see the next section of this post.
- Although `week_day.Monday` is defined as having value `1`, it does not matches to integer `1`, for user-defined
types are always matched with type-awareness.
- `case (1,)` and `case [1]` actually refers to the same pattern. According to
[the language specification](https://docs.python.org/3/whatsnew/3.10.html#other-key-features),
>> Like unpacking assignments, tuple and list patterns have exactly the same meaning and actually match arbitrary sequences.
>> Technically, the subject must be a sequence. Therefore, an important exception is that patterns don’t match iterators.
>> Also, to prevent a common mistake, sequence patterns don’t match strings.

This means `case (1,)`, `case [1]` and `case 1,` (**notice the comma after 1**) are the same.
To be consistent, we can stick to the square bracket syntax when matching against sequences.

# Match by types
We can specify the type of the object following keyword `case`. In this case, Python
ensures that types are also compatible while performing pattern matching. We can also
give default values to the fields of the type. For example, `case int()` matches to
any integer, and `case str()` matches to any string. To fix the problem in the previous
section, instead of matching with `case True`, we can instead matching with `case bool(True)`.
The latter means that we want to match to an instance of `bool` with value being `True`.
If we want to match a tuple that contains `1`, we can use `case tuple(1,)`, and
`list([1])` if list.

**Note that when matching against a type, instance initialization is not invoked.**
Consider the following example:
```python
class my_class:
    def __init__(self, name, age):
        self.name = name
        self.age = age
        print("do something crazily complicated during object construction")


def match_name_age(e):
    match(e):
        case my_class(name = "John", age = 20):
            print("John is 20 years old")
        case my_class(name = "John", age = 30):
            print("John is 30 years old")

match_name_age(my_class("John", 30))
```
The string "do something crazily complicated during object construction" is only
printed once. And that is when the parameter of the function is evaluated.
The `case` clause does not call the `__init__` function of the class. Pattern matching
is done as if the class is just an aggregation of a string  named `name` and an integer named `age`.
**Side effects are silently dropped.**

Among all types, dictionaries are special, Python has special support to pattern matching against
dictionaries. For example, the following code matches a key-value pair with the key being `name`.
```python
match {"name": "John", "age": 30}:
    case {"name": name}:
        print("the name is {}".format(name))
```
In the above code, we not only have pattern matching, but also name binding.
The second `name` in `case {"name": name}` refers to the corresponding value of key "name".
**Notice that if there is another variable named "name" exists in the outer scope of the match block,
its definition is shadowed by the "name" in the case block.**

When there is a binding name in a `case` clause, the pattern can be further constrained with a tailing predicated marked by `if`.
For example,
```python
match {"name": "John", "age": 30}:
    case {"age": age} if age >= 18:
        print("the person is at least 18 years old)
```
The above code matches a key-value pair with the key being "age" and the value being equal or larger than 18.

If no binding name is introduced, but we want to bind the matched result to a named variable, we can do so use `as`.
For example
```python
match {"name": "John", "age": 30}:
    case {"name": "John"} as record:
        print(record)
```
will print `{"name": "John", "age": 30}`.

## Combination of patterns with `|`
Several patterns can be combined together use the `|` operator, that is logical `or`,
though `or` itself is not allowed to be used in pattern matching context.
This is trivial with normal patterns, For example `case str() | int()` matches a string or a integer.
`None | False` matches `None` or `False`. `case "Hello" | "Hi"` matches `"Hello"` or `"Hi"`.
What makes it extremely useful is when matching against dictionaries. For example
```python
match {"name": "John", "age": 30}:
    case {"name" : ("John" | "john")} :
        print("uppercase or lowercase, I don't care")
```
matches to `{"name": "John"}` or `{"name": "john"}`. The parenthesis is added for readability. It is not required.
