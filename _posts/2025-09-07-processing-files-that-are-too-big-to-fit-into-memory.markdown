---
layout: post
mathjax: true
comments: true
title:  "Processing files that are too big to fit into memory"
author: John Z. Li
date:   2025-09-07 10:00:00 +0800
categories: programming
tags: command-lin, linux, awk
---
Someone shared an interview question he got asked during a programming skills interview session the other day.
The question is like this
> There is this production system, it splits out metadata about each client request into a csv file.
> At the end of each day, the file needs to be processed to remove duplicate lines as part of pre-processing steps.
> But the file is very large with its size exceeding the size of the main memory.
> The task: propose a feasible and efficient solution to this problem.

Usually, the interviewer is expecting something along the below:
1. Split the input file into small chunks that fits into memory.
2. Sort each small chuck, removing duplicated lines along the way.
3. Merge the sorted small chunks into larger ones, doing this without triggering an Out-of-memory condition.
4. Repeat step 3 until the whole file is sorted without duplicated lines.
Basically, the task can be reduced to implementing an *External Merge Sort*.
An optimized implementation might also use multi-threading.

However, this task has a much simpler solution, that is the `sort` command of Linux.
A simple command line as below will work
```bash
sort -u  file.csv > output.csv
```
The way it works is much like the approach described at the beginning of the post,
except that `sort` already does that for us. When dealing with large input files,
`sort` is smart enough to switch into its *External Merge Sort* implementations.
One can even enable multi-threading by simply
```bash
# the --parallel option tells sort how many thread to use internally
sort --parallel=$(nproc) -u file.csv > output.csv
```
The default buffer size of `sort` is about `1MiB`, this is often sub-optimal.
We can specify a large buffer size that fits into main memory:
```bash
# -S option to specify the in-memory buffer size
sort --parallel=$(nproc) -S 2G -u file.csv > output.csv
```

With a few key strokes, the problem is solved. But wait, then the interview said
> It is good that you can remove duplicated lines by sorting lines first.
> But this approach alters the order of lines.
> If the metadata does not contain a timestamp, we won't be able to know whether this client connection comes before or after that one.
Now what?

Recall the old trick in `awk` to remove duplicated lines, that is
```bash
awk '!seen[$0]++' input.txt > output.txt
```
If you don't already know `awk`, here is a brief explanation what happens:
1. `$0` is "current line" when `awk` processes `input.txt` line by line.
2. `seen` is a user defined hash table, and `seen[$0]` is the associated value of using `$0` as key, which is a number in this case.
3. `++` increases the mapped-to number by `1`, and `!` negates it.
The end result is, if the current line is already in the hash table, `!seen[$0]++`
will be evaluated into `False`, otherwise `True`. As the default action for `awk` is print the current line,
each unique line will only be printed out once.

We are almost there, but not quitely.
The problem is, if the input file is so large that can't even fit into main memory,
the hash table `seen` will use up all the available memory and cause the script to abort because of OOM.
If only there is some way that we can offload that big hash table into some disk-backed storage.
It turns out `awk` already supports this.

Newer versions of `awk` (5.2 or higher) has the ability to use "persistent memory",
that is `awk`'s term to call memory mapped files using `mmap`. The idea is simple,
the user provides a big on-disk file as the back store beforehand. When `awk` needs to allocate
memory at runtime, it uses the memory mapped region as a big memory buffer. As the
file is memory mapped, the OS is responsible to handle page faults and page swapping.
If a page become cold (not accessed by the program), the OS might decide to swap it out to the disk.
If the program access a page but it is not in kernel's page cache, the OS will load the page from disk into main memory by handling page fault.

Usage is simple:
```bash
# this only creats a sparse file. It won't occupy actual disk space until needed
truncate -s 20G data.pma
# let awk be aware the existence of the file
export GAWK_PERSIST_FILE=data.pma
# awk will use the memory mapped file to allocate memory
awk '!seen[$0]++' file.csv > output.csv
# we don't need the file anymore
rm data.pma
```

Conclusion: Linux command line tools, especially those alredy bundled with OS,
are more powerful than most people think. Try simple solutions before rolling up
your sleeves and implement your super fast solution. (Hint: It is usually not
super fast without a lot of optimization, and most likely not worth the effort).
