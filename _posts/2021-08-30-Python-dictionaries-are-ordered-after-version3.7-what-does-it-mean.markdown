---
layout: post
mathjax: true
comments: true
title:  "Python dictionaries are ordered after version 3.7, what does it mean"
author: John Z. Li
date:   2021-08-30 19:00:18 +0800
categories: c++
tags: python, dictionary
---
Someone asked on stackoverflow that [whether dictionaries are ordered in Python 3.6+](https://stackoverflow.com/questions/39980323/are-dictionaries-ordered-in-python-3-6).
The answer to that question, quoting the accept answer,  is that (emphasis mine):

    > As of Python 3.6, for the CPython implementation of Python,
	> dictionaries remember the order of items inserted.
	> This is considered an implementation detail in Python 3.6;
	> you need to use OrderedDict if you want insertion ordering that's guaranteed
	> across other implementations of Python (and other ordered behavior[1]).
    > As of Python 3.7, this is no longer an implementation detail and instead becomes a language feature.

What does it mean to "remember the order of items inserted"?
In short, it means that if you initialize a dictionary like:
```python
job_title = { 'John': 'Engineer', 'Keith' : 'Manger',}
job_title['Eric'] = 'Director'
job_title['Judith'] = 'CTO'

print(job_title)
```
For Python3.6+, the output will be
```
{'John': 'Engineer', 'Keith': 'Manager', 'Eric': 'Director', 'Judith': 'CTO'}
```
In other words, an dictionary object now keeps the order of its elements as the same as they are added.

Previously, that is before Python 3.6, a Python dictionary is just a set of key-value pairs which
are stored in a sparse array. Where exactly a given key-value pair will be located depends on the hash function used.
Typically, this means that dictionary elements are stored in a random fashion.
A possible memory layout is as below:

|Hash of key      |Key           |Value object  |
| --------------- | ------------ | ------------ |
| hash('Eric')    | "Eric"       | "Director"   |
| ...             | ...          | ...          |
| hash('Judith')  | "Judith"     | "CTO"        |
| ...             | ...          | ...          |
| hash('Keith')   | "Keith"      | "Manager"    |
| ...             | ...          | ...          |
| hash('John')    | "John"       | "Engineer"   |

The new implementation after Python 3.6 reserves the order of inserted elements of a dictionary
by adding another layer of indirection: The "values" in key-value pairs are now stored in a vector,
and the indices of those "values" are referenced in the hash table, like below


|Value index   |
| ------------ |
| 2            |
| ...          |
| 3            |
| ...          |
| 1            |
| ...          |
| 0            |
with a corresponding dense vector as
```python
[[hash('John'),   'John',   "Engineer", ],
 [hash('Keith'),  'Keith',  "Manager", ],
 [hash('Eric'),   'Eric',   "Director",]
 [hash('Judith'), 'Judith', "CTO"]]
```

The advantages of the new implementation are three-fold:

1. First, iterating over a dense vector is much faster than iterating a sparse array.
It only involves increment a pointer by a fixed step size.
2. Since the sparse array now only needs to store indices, which are integer numbers,
it will consume less memory.
3. Most of time, dictionaries are inquired for the existence of a key-value pair, or
new key-value pairs are added. Removing key-values pairs happens less frequently.
A typical implementation of dense vectors is capable of handling vector length growth
quite well. A good implementation should also make it possible that a vector grows in-place
using things like `realloc()`.
