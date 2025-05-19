---
layout: post
title: "Improving Error Propagation in Go"
subtitle: "A Practical Approach"
date:   2025-05-19 15:00:00 +0200
categories: general golang error handling
---

The way error propagation is commonly done in Go can be a bit noisy and often doesn't fully meet the needs of most developers.

In traditional Go error handling, we often use `fmt.Errorf` with `%w` to wrap errors. Over time, this can create very long and unreadable error messages. For example:

```go
err := fmt.Errorf("failed to load config: %w", 
            fmt.Errorf("could not read file: %w", 
                fmt.Errorf("permission denied")))

// Error: failed to load config: could not read file: permission denied
```

This flattens into a long, nested message that's difficult to parse — especially once you add multiple layers deep into a production system.

In contrast, a structured error approach could separate concerns neatly:

```go
err := errors.New("ConfigError", "failed to load config")
err = errors.Annotate(err, "filename", "/etc/config.yaml")
```

Here, you get:
- A **clear message** (`failed to load config`)
- A **kind/type** (`ConfigError`)
- **Extra context** (`filename: /etc/config.yaml`)

Instead of digging through wrapped strings, you can query structured fields or use them directly in logs, UIs, or debugging tools. Combined with a stacktrace this would contain all the information required to debug issues effectively.

Recently, I came across an article by Michael Olofinjana: [Golang Error Handling: A Practical and Robust Solution](https://dev.to/michaelolof_/golang-error-handling-a-practical-and-robust-solution-3am1).  
There were some very good ideas in it, especially the "Good error reporting" checklist:

A good error report should inform me of:
1. **What went wrong** (Required — usually the error message)
2. **Where it went wrong** (Required — ideally a stack trace or identifier)
3. **What type of error it is** (Optional — could be error kinds or custom types)
4. **Useful additional data** (Optional)

While I agree with the checklist, I wasn't convinced by the proposed implementation.  
For example, the use of timestamps felt unnecessary — you can simply use `runtime.Callers` to generate a stack trace automatically, which is much more helpful and requires less manual work.

Also, for point 4 (attaching useful data), I expected something like an `error.Annotate(name string, data any)` functionality — but that was missing entirely.

## What Do We Actually Need?

In my view, a clean solution would involve creating a **small library** that wraps Go's standard `errors` package and adds a few essential features.

At its core, we would need a struct that stores:
- the error **message**,
- an optional **kind** (type of error),
- a collection of **annotations** (extra context),
- a **stack trace**,
- and the **parent** error if it wraps another error.

Here's a rough sketch of what that could look like:

```go
type internalError struct {
    Kind        string         `json:"kind"`
    Message     string         `json:"message"`
    Annotations map[string]any `json:"annotations,omitempty"`
    Stacktrace  string         `json:"stacktrace,omitempty"`
    Parent      error          `json:"parent,omitempty"`
}
```

To create a new error, we need a simple `New` function that automatically generates a stack trace:

```go
func New(kind string, msg string) error {
    return &internalError{
        Kind:       kind,
        Message:    msg,
        Stacktrace: stackFromCallers(),
    }
}
```

The stack trace can be generated using the `runtime` package:

```go
func stackFromCallers() string {
    const depth = 32
    var pcs [depth]uintptr
    n := runtime.Callers(4, pcs[:])

    frames := runtime.CallersFrames(pcs[0:n])
    var stacktrace string
    for {
        frame, more := frames.Next()
        if !more {
            break
        }
        stacktrace += fmt.Sprintf("%s\n\t%s:%d\n", frame.Function, frame.File, frame.Line)
    }
    return stacktrace
}
```

Additionally, we can add an `Annotate` function to enrich errors with extra context:

```go
func Annotate(err error, key string, value any) error {
    if err == nil {
        return nil
    }
    internalErr, ok := err.(*internalError)
    if !ok {
        internalErr = New("", err.Error()).(*internalError)
        internalErr.Parent = err
    }
    if internalErr.Annotations == nil {
        internalErr.Annotations = make(map[string]any)
    }
    internalErr.Annotations[key] = value

    return internalErr
}
```

Finally, we'd just need to implement the necessary methods to satisfy Go's `error` interface and ensure compatibility with `fmt`, `slog`, and other packages.

With only about **150 lines of code**, we'd have a much more powerful and structured error handling library — without introducing too much complexity.

---

If you're interested in a complete implementation with tests and documentation, check out the finished version here: [github.com/Neokil/errors](https://github.com/Neokil/errors).  
The repository includes everything you need to get started — feel free to explore, use it in your own projects, or contribute!
