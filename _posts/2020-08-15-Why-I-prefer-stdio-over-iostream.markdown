---
layout: post
mathjax: true
comments: true
title:  "Why I prefer stdio over iostream"
author: John Z. Li
date:   2020-08-15 19:00:00 +0800
categories: c++ programming
tags: stdio, iostream, string-format
---

Say you want to implement a simple class called `csv_writer`.
As its name suggests, it writes formatted outputs to csv files.
Suppose that you want to the "csv_writer" class not as a one time shot task,
but as a building block of your private toolbox, that is, you want it to be general purposed.
Trying to implement the class using `stdio` and `iostream`, respectively,
you will soon find some very annoying things about `iostream`.

First thing you will notice is that using `iostream` is quite verbose,
for example, you write the following to print a 8-bit register value represented as `uint8_t`.

```cpp
     csv_file << std::setfill('0') << std::setw(2) << std::hex<<static_cast<unsigned>(value_uint8_t) << "\t";
```

Since you are a capable programmer and you like typing, you make it work.
Now you proceed with printing out a sensor reading which is an `int`.
So you go on with

```cpp
    cav_file << sensor_reading_int << “\t”
```

You notice that the output is not correct, because `std:hex` is still in effect,
and the `int` is output in hexadecimal format.
That is not what you want, So you change the above code into

```cpp
    cav_file << std::dec << sensor_reading_int << “\t”;
```

Despite the fact that you like typing, you soon find this is not fun when you
have to do this kind of thing for many times.
Your friend tells you that there is a  `boost` library to help you out, which is
called `iostate`.
Since you are a very smart programmer, you soon figured out how to wrap each
csv field output in a macro like this:

```cpp
    #define AUTO_COUT(x) {\
     boost::io::ios_all_saver ias( cout );\
     x;\
     }while(0)
```

And now the output code becomes:

```cpp
    AUTO_COUT(std::setfill('0') << std::setw(2) << std::hex<<static_cast<unsigned>(value_uint8_t) << "\t");
    AUTO_COUT(cav_file << std::dec << sensor_reading_int << “\t”);
```

Great, you get a working `csv_writer` class.

You start using the `csv_writer` class in several your projects.
But strange things began to happen in some multi-threading programs
where multiple threads write lines to the same `csv` file and you get
inter-mingled lines in your `csv` file.
After some googling, you find out that `iostream` is not thread safe,
because each call of operator `<<` is actually a separate function call.
Before a line is completed in outputting,
there are chances that the current thread is scheduled out and another thread
kicks in and calling the operator `<<` to output in the same line.
You fix the problem by adding a `lock_guard` into your `write_line()` function.
And your friend tell you that C++20 introduced `std::osyncstream`,
so that you won’t have to manually handle synchronization in the future.
That is great, but you still need your library work with earlier language standards.

Finally, you get your `csv_writer` class done.
It is a useful class, even your friend starts using it in his own projects.
Problem is, your friend somehow find your default output format is ugly.
He wants you modify your `csv_writer` class so that it becomes configurable.
It is at this point, you find yourself completely helpless with `iostream`.
You find you have to invent a mini language that can be used to specify
how a sequence of values should be formatted.
You also want the mini language to use only ASCII characters,
so that you can store formatting objects written in the mini language in a C-string,
which can be passed around, or even constructed on the fly according to user input
when the program is running.
Of course, you also want the mini language to be simple enough,
because you don’t want to include a full-fledged parser into your `csv_writer` class.
You are a very smart guy, you succeed in inventing the mini language.

You feel like it is really a great idea,
because this is how things should be done.
Your mini language is expressive and its usage is succinct.
Now outputting a line consists of only a single underlying system
call named `write`. In all modern OSes, it is thread safe per se.

At the same time, the other you, the one who chooses to implement the `csv_writer`
class with `stdio`, will find that you have just re-invented what is already in `stdio`.

