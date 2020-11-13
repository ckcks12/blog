---
layout: post
title: Oh My ZSH show elapsed time from just executed command
author: Eunchan Lee
---

# For OSX User
1. Install `coreutils` since OSX doesn't have millisecond-date.
```bash
brew install coreutils
```
2. Add hook functions to `~/.zshrc`
```bash
function preexec() {
 timer=$(($(gdate +%s%0N)/1000000))
}
function precmd() {
 if [ $timer ]; then
   now=$(($(gdate +%s%0N)/1000000))
   elapsed=$(($now-$timer))
   export RPROMPT="%F{cyan}${elapsed}ms %{$reset_color%}"
   unset timer
 fi
}
```

# For Linux User
1. Add hook functions to `~/.zshrc`
```bash
function preexec() {
 timer=$(($(date +%s%0N)/1000000))
}
function precmd() {
 if [ $timer ]; then
   now=$(($(date +%s%0N)/1000000))
   elapsed=$(($now-$timer))
   export RPROMPT="%F{cyan}${elapsed}ms %{$reset_color%}"
   unset timer
 fi
}
```

# For Windows User
🤔 ????

