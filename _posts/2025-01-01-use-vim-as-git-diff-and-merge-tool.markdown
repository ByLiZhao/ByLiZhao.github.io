---
layout: post
mathjax: true
comments: true
title:  "Use Vim as git diff and merge tool"
author: John Z. Li
date:   2025-01-01 20:00:00 +0800
categories: vim
tags: vim, git, diff
---
When working on remote servers, GUI-based diff and merge tools are uaually
a luxury we cannot afford. In such a situation, `vim` is often what we can rely
on, because it is installed by default by almost all distributions.
It is useful to learn how to do `git diff` and `git merge` from terminal with only
`vim`.
## Configuration
1. Put the following in `.bashrc`, so that `vimdiff` always uses the `vim`
found in `$PATH`, instead of the `vimdiff` command in `$PATH`.
```bashrc
alias vimdiff="vim -d"
```
This is needed because `vimdiff` opens files in readonly mode by default,
This is usually what one needs.
if a file is already opened in readonly mode,
one can use `:set noro` to enable editing inside `vim`.)

2. Put the following in `.gitconfig`:
```
[diff]
        tool = vimdiff
[difftool]
        prompt = false
[merge]
        tool = vimdiff
        conflictstyle = diff3
[mergetool]
        prompt = false
```

## Invoke `vimdiff` from command line.
1. Compare two or more files
```bash
vimdiff file1 file2 [file3 [file4]]
```
If you are already in `vim`, and you want compare the current buffer with another file:
```
:diffsplit [filename]  # same as diffs
: vert diffsplit [filename] # same as above but windows are vertically splitted.
# The below command set the current window part of the diff windows, as if the
# window is opened with `vimdiff`.
:diffthis # same as difft
```
When you are done, use `diffoff` or `diffoff!` to switch off diff-mode.
The former disables diff mode in the current window, the latter disables diff
mode in all the windows under the current Tab.

2. Jump between diff-conflictstyle
- Use `[c` to jump to the previous diff-conflict.
- Use `]c` to jump to the next diff-conflict.

3. Dealing with diff-conflicts
- `:[range] diffget [bufspec]`, or `:diffg` when dealing with the current diff-conflict and with only two files.
This modifies the current buffer to remove difference against another buffer.
- `:[range] diffput [bufspec]`, or `diffpu` when dealing with the current diff-conflict and with only two files,
This modifies other buffers to remove difference againt the current buffer.
- `do`, same as `:diffget` without range or bufspec, `o` stands for obtain.
- `dp`, same as `:diffput` without range or bufspec, `p` stands for put.

4. Re-synchronize diff-view
If diff-view is out-of-sync, use `:diffupdate` or `:diffu` to

## Start `vimdiff` from `git`
1. Compare a file with staged or unstaged changes:
```bash
git difftool [<filename>*]
```

2. Compare a file with the the N-commit-before version of the same file
```bash
# show all changed files between head 1-commit before
git diff --name-only HEAD^ HEAD
# like --name-only but has modifier charactor to indicated what
# has happened to the file
git diff --name-status HEAD^ HEAD
# see file change history
git log --name-status --oneline
# the following syntax is equivalent,
# but with 2-commit before and file names
git difftool HEAD^^ HEAD [<filename>*]
git difftool HEAD^^..HEAD -- [<filename>*]
git difftool HEAD~2 HEAD -- [<filename>*]
```

3. Compare a file with the specific commit version of the same
file
```
# spceify tag name
git difftool tagname HEAD -- [<filename>*]
git difftool tagname..HEAD -- [<filename>*]
# specify commits by commit-has
git difftool [commit-hash1] [commit-hash2] -- [<filename>*]
git difftool [commit-hash1]..[commit-hash2] -- [<filename>*]
```

4. Compare a file across diffrent branch
```bash
git difftool [branch1] [branch2] -- [filename]
git difftool [branch1]..[branch2] -- [filename]
```

## Use `git mergetool` to solve merge conflict
When there is a merge conflict,
1. Use `git mergetool` to lauch vim in merge mode.
2. From the left to right, three windows are
- 'LOCAL', the merged-to branch
- 'BASE', common acestor
- 'REMOTE', the merged-from branch
- The bottom window is mergeconflict file in `diff3` format.
The mergeconflict file is the file we will commit when all conflicts are solved.
3. We can use `]c` and `[c` to jump forward and backward on the merge conflict file.
4. Use the following to select which change is applied to the merged file:
- `:diffget RE` or `:diffg RE` to select change from 'REMOTE'.
- `:diffget BA` or `:diffg BA` to select change from 'BASE'.
- `:diffget LO` or `:diffg LO` to select change from 'LOCAL'

## Invoke `git` tools from `vim` using tools like `vim-fugitive`
1. `:Git diff` to check staged or unstaged changes of the current file.
2. `:Git difftool` works like `:Git diff` but put differences in a quickfix window. (*not very useful*).
3. `:Gvsplit HEAD~3:%` to open the version of this file 3-commit-ago in a vertically split window.
4. `:Gedit HEAD~3:%` and `Gsplit HEAD~3:%` do the same but current window or horizontal new window.
5. `:Gvdiffsplit` to compare the current file with the staged version of the same file.
6. `:Git mergetool` is broken.

