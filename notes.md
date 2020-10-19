---
layout: page
title: Notes
permalink: /notes/
---

## Shell commands

### `tar`

```bash
# Say you want to tar blah/user/app,
# To make an archive *with* the outermost folder inside
tar czf achive.tar.gz -C blah/user app
# To make an archive *without* the othermost folder inside
tar czf archive.tar.gz -C blah/user/app .

# To untar
tar xf archive.tar.gz
# To untar to foo/bar
mkdir -p foo/bar # tar will not make the path
tar xf archive -C foo/bar
```

`tar` is an odd command, where you invoke it from changes the layout of the archive. I would expect `command path1 path2` (where `path1` is the thing to archive and `path2` is the output) to always behave the same, but where you run it from matters. Here are some examples. (Output is pasted as a comment where relevant so you don't need to run.)

```bash
mkdir -p user/app
touch user/file
touch user/app/file

echo '  from top'
tar czf archive user
tar tf archive
# user/
# user/app/
# user/file
# user/app/file

echo '  from inside'
cd user
tar czf ../archive .
cd ..
tar tf archive
# ./
# ./app/
# ./file
# ./app/file
```

Becasue of this weird behavior I prefer to use `-C` with it. Now what matters is if you want the top level folder or not. i.e., when you open it do you want it in a folder or directly in the working directory? If you *do* want the folder then use the first option. If not use the second.

```bash
echo '  with top level folder'
tar czf archive -C user app
tar tf archive
# app
# app/file

echo '  without top level folder'
tar czf archive -C user/app .
tar tf archive
# ./
# ./file
```

When using the second form you may want to do `tar xf archive -C some_dir` to make sure it goes into a directory named `some_dir` instead of directly into the working directory.
