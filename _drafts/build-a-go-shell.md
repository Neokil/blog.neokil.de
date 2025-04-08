---
layout: post
title: "Build a shell in go"
subtitle: "..."
date:   2025-04-01 16:00:00 +0200
categories: general golang
---

# Why?
- interesting project that includes some interesting topics like a tokanizer to parse the commands
- can be used in daily tasks and could be extended with more functionality

# What is required for MVP?
- loop: read input in the form of "command arg --arg2", call the binary with the arguments and show the output.
- if a file is piped into the application it should execute each line in the same way as if it was typed
- add basic builtin commands
    - `cd` to change directory
    - `exit` quit the shell
    - `pwd` output current directory
    - `echo` output text
    - `alias` to add an alias for a command
- signals like SIGINT should be forwarded to the command that is currently executing

# Features
- Add Environment-Variable support
    - `export FOO=BAR` sets it
    - `unset FOO` unsets it
    - all environment variables need to be passed down to the executing commands
    - substitute `$FOO` with the value of foo
- Add `.goshrc` support
    - add support for a file to be automatically read on start of the shell
- Add Pipe operator
    - `|` should pipe the output of one command into the input of the other command
- Add IO operations
    - `>` pipe output into file
    - `>>` append output to file
    - `<` read input from file
- Add subcommand capabilities
    - `$(...)` should be executed fore the outer command
- Add loops and conditions
    - add some `if, then, else` keywords that can be used for conditional execution
    - add some `for, do, while` keywords that can be used for loops
    - parallel loop execution maybe?
- Add History
    - command history should be saved to a file 
    - command starting with whitespace should be excluded
    - arrow up and down should allow you to go through the history to replay existing commands
