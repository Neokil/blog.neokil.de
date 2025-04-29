---
layout: post
title: "Better error handling"
subtitle: "..."
date:   2025-04-01 16:00:00 +0200
categories: general golang
---

The general idea of how error propagation is done in golang is a bit noisy and does not really fit the needs of most developers.
Wrapping the error message and adding some additional information can work, but having a stacktrace instantly makes the path much clearer.
Also being limited to string messages makes it hard to provide some context, so developers are adding more "debug" and "info" logs which is not a good way because those are not linked to the error itself and they pollute the logs if everything goes well.
I read this article fro Michael Olofinjana https://dev.to/michaelolof_/golang-error-handling-a-practical-and-robust-solution-3am1 

There are some some good ideas that got me thinking, especially the list of "Good error reporting":
A good error report should inform me of the following:
1. What went wrong. (Required. This is usually the error message)
2. Where it went wrong. (Required. This is where you'd need a stack trace or some form of identifier)
3. Type of error it is. (Optional. Could be error kinds, custom error types etc.)
4. Useful data or information attached to the error (Optional)

but I don't like the implementation at all. Those timestamp things are not really useful as you can simple use the `runtime.Callers` function to get a stacktrace which is much more helpful and requires less manual work.
Also with point 4 I would have expected something like `error.Annotate(name string, data any)` to add more information to the error which is missing completely.

So what do we need?
A new library called errors that wraps the standard errors package and adds some functionality on top.
So save the data we need a struct that contains the Message, Kind, Annotations, Stacktrace and the parent-error when we wrap errors.
```golang
type internalError struct {
	Kind        string         `json:"kind"`
	Message     string         `json:"message"`
	Annotations map[string]any `json:"annotations,omitempty"`
	Stacktrace  string         `json:"stacktrace,omitempty"`
	Parent      error          `json:"parent,omitempty"`
}
```
Then we need a `New` function to create an error which automatically generates a stacktrace.
```golang
func New(kind string, msg string) error {
	return &internalError{
		Kind:       kind,
		Message:    msg,
		Stacktrace: stackFromCallers(),
	}
}
```

The stacktrace can be retrieved using the runtime package:
```golang
func stackFromCallers() string {
	// get callers, skipping the first 4 frames, as those frames contain the errors-internals
	const depth = 32
	var pcs [depth]uintptr
	n := runtime.Callers(4, pcs[:])

	// get frames for the callers and build stacktrace
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

Then we add a new `Annotate` function to add more context to the error
```golang
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

At least we have to implement some functions to satisfy the errors interface and to make it compatible with `slog` and `fmt` and with only around 150 lines of code we have our own errors library that brings a lot of benefits to the table.
