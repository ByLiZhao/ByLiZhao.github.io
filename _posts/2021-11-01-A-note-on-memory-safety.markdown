---
layout: post
mathjax: true
comments: true
title:  "A note on memory safety"
author: John Z. Li
date:   2021-11-01 19:00:18 +0800
categories: c++
tags: memory-safety
---
Consider the following sample C++ code:
```cpp
void ReplaceInt(int * & i){
    delete i;
    i = new int{100};
    return;
}
int main(void){
    int * my_int = new int{};
    int * my_int_alias = my_int;
    ReplaceInt(my_int);
    printf("my_int: %d, my_int_alias: %d", *my_int, *my_int_alias);
}
```
In the above code sample, function `ReplaceInt` takes a pointer to an int by reference.
Inside the function, the pointed-to int is freed, and the passed pointer points to
another memory location after the function has returned.  In the `main` function,
the pointer `my_int` has an alias `my_int_alis` which points to the same piece of memory
as `my_int` does. After the call to function `ReplaceInt` has returned, `my_int_alias` now
points to freed memory. Thus, accessing the memory pointed by `my_int_alias` in the following
`printf` function leads to undefined behavior.

This is an example of the typical "use after free" type of memory bugs. This kind of memory security bugs
have been well understood by C and C++ programmers for a long time.
**We are tempted to think that if we just use a language with garbage collection, this kind of bugs will
not occur.** Let us translate the code into equivalent C# code:
```csharp
class Program {
  static void ReplaceInt(out object i) {
    int j = 100;
    i = j;
    return;
  }

  static void Main(string[] args) {
    int i = 0;
    object my_int = i;
    object my_int_alias = my_int;
    ReplaceInt(out my_int);
    Console.WriteLine("my_int: {0}, my_int_alias: {1}", (int)my_int,
                      (int)my_int_alias);
  }
}
```
Thanks to the garbage collector, `my_int_alias` now points to valid memory address
even after `my_int` has pointed to a different place. The garbage collector is smart
enough to know that the piece of memory that contains `0` is still alive because
it is reachable from the living root `my_int_alias`. So, problem solved?

However, if we look at the problem more closely, we find out that with the C# version,
`my_int_alis` points to valid memory but the wrong object or the value of that object.
**Indeed, valid memory only means that the program does not access memory it should not touch,
it says nothing about whether the program has right logic. Memory security could be just
a symptom of logic error.**

This kind of bugs actually more often than one might think. For example, on calling a library
API, a pointer (objects in languages that does not explicitly use pointers are just pointers in disguise)
is passed to the library. And inside that library, the address pointed to by the pointer is cached.
But the upper layer code later lets the pointer point to somewhere else, making the value cached
by inside the library stale. With C++, this leads to memory bugs. If the upper layer code does not
free the memory pointed to by the pointer, that piece of memory will never get freed, leading to memory leak.
If the pointed memory is freed by the upper layer code, the cached pointer of the library now points to freed memory.
That is an "use-after-free" bug as before.

With a language with garbage collection, **the code is still incorrect, but the problem is concealed by
the garbage collector.** It is very possible that you are accessing the wrong object without noticing it.
The garbage collector keeps the object alive, but it is only that. It takes more effort to right
correct code than memory-security code.
