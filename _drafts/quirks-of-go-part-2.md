---
layout: post
title: "Quirks of go - Part 2"
subtitle: "..."
date:   2025-04-01 16:00:00 +0200
categories: general golang quirks-of-go
---

1. function calls on nil-pointers
You can call a method on a nil-pointer (like in `api/talonlib/ruleengine/tracing.go::InitTracing`). It will not fail, but that also means if you define a method like `func (t *TraceProvider) NewTracer(pkg string) Tracer` you always have to do a nil-check for `t`.
Important to note that a the nil-pointer exception does not happen when you call the function but only if you try to access something from `t` within the function

2. appending to slices
Gos slice growth mechanism can lead to unexpected mutations when appending to slices that share the same underlying array.
Example
```
s1 := []int{1, 2, 3}
s2 := s1[:2] // s2 shares the same backing array as s1
s2 = append(s2, 99) // Mutates s1 unexpectedly
fmt.Println(s1) // Output: [1, 2, 99]
```
Explanation: append may modify the original array instead of allocating new memory.