---
layout: post
mathjax: true
comments: true
title:  "A note with compound string literals in C"
author: John Z. Li
date:   2021-01-08 19:00:18 +0800
categories: c programming
tags: string-literal
---
The following seemingly innocent code actually contains undefined behavior with C99.
```c
      struct person {
        int age;
        char *name;
      };

      struct person *sp;
      if (condition)
        sp = &(struct person){6, "John"};
      else
        sp = &(struct person){7, "Joy"};
```
Starting from the C99 standard, an if statement without braces around its body
is just a shorthand for that with braces around its body.
In other words, The if statement in the above code is equivalent with the following:
```c
    struct person *sp;
      if (condition){
        sp = &(struct person){6, "John"};}
      else{
        sp = &(struct person){7, "Joy"};}
```
A pair of braces introduces a block scope.
As a consequence, all objects with automatic storage duration cease to
exist after the program goes beyond the scope.
And compound literals, unless it is defined outside any function,
have automatic storage duration.
This means, after the if statement ends,
those compound string literals are no longer there, leading to undefined behavior.

