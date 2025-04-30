---
layout: post
title: "Build a Shell in Go"
subtitle: "A hands-on project to learn about tokenization, system calls, and shell features"
date:   2025-04-01 16:00:00 +0200
categories: general golang
---

# Why Build a Shell in Go?
Writing your own shell is a fun and practical way to deepen your understanding of how operating systems interact with user commands. It's a rewarding project that touches on a wide range of topics like parsing and tokenizing user input, handling system calls, and managing processes. And beyond being educational, the result is a useful tool that you can extend for your daily tasks.

# What could an MVP Look Like?
To get a working shell off the ground, here's what's needed:

- **Input loop:** Continuously read user input (like `command arg --flag`) and execute the corresponding binary with arguments.
- **File input:** If the shell receives a file via standard input, it should execute each line as if typed manually.
- **Built-in commands:** A few essential shell functions should be implemented directly:
  - `cd`: change the current working directory
  - `exit`: quit the shell
  - `pwd`: print the current directory
  - `echo`: print to standard output
  - `alias`: define shortcuts for commands
- **Signal handling:** Signals like `SIGINT` (Ctrl+C) should be properly forwarded to running commands.

# Planned Features
To make the shell more robust and user-friendly, consider adding:

### Environment Variable Support
- Use `export FOO=BAR` to set variables, and `unset FOO` to remove them.
- Substitute `$FOO` in commands with its value.
- Ensure all variables are passed to child processes.

### `.goshrc` File
- Load and execute commands from a `.goshrc` file on startup, similar to `.bashrc`.

### Pipes
- Use the `|` operator to pipe output from one command into another's input.

### I/O Redirection
- `>`: Redirect output to a file.
- `>>`: Append output to a file.
- `<`: Read input from a file.

### Subcommands
- Support command substitution using `$(...)`, executing the inner command first and passing its output to the outer one.

### Control Flow
- Add conditional execution using `if`, `then`, `else`.
- Support loops using `for`, `do`, `while`.
- Consider supporting parallel loop execution for advanced scripting.

### Command History
- Store command history in a file for later reuse.
- Skip logging commands that begin with a space.
- Use arrow keys to navigate through previous commands interactively.

---

With these goals in mind, you'll not only learn a ton about Go and UNIX internals, but you'll also end up with a powerful, extensible tool tailored to your needs. Stay tuned as I document the journey of building this shell step by step.
