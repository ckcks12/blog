---
layout: post
title: How To Read User Input in Makefile
author: Eunchan Lee
tags: ["programming", "make", "build", "ci/cd"]
---

# TL;DR
Use doubled dollar sign. `$$` becomes `$` for shell.
```
build:
  @read -p "tag? : " TAG \
  && echo "tag : $${TAG}"
```

# Explain
`make` has own syntax for variable.

|Example|`make` -> `shell`|
|-|-|
|var=test||
|echo $(var)|echo test|
|echo $var|echo test|
|echo ${var}| echo |
|echo $$var| echo $var|

A doubled dollar($$) sign becomes a single dollar($). Just like a backslash(\).
`make` has its own **variable syntax**. It interpretes with values from `-e` arguments.
