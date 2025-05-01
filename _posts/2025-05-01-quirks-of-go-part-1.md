---
layout: post
title: "Quirks of go - Part 1"
subtitle: "Exploring Go's Subtle Behaviors: Unexpected Edge Cases and How to Navigate Them"
date:   2025-05-01 00:00:00 +0200
categories: general golang quirks-of-go
---

# Quirks of Go - Part 1

Over the years of developing applications in Go, I have encountered some unexpected behaviors that can be quite puzzling at first glance. Recently, I started collecting these quirks and decided to share some of the more intriguing ones, along with potential workarounds.

In this post, I will discuss two specific quirks: unexpected behavior when using `time.AddDate` and how Go's handling of `nil` types can lead to subtle bugs.

## `time.AddDate`

Let's start with `time.AddDate`, a seemingly simple function. It allows you to add or subtract a specified number of years, months, or days from a given date. However, what happens if the target date does not exist? Consider the following example:

```go
t1 := time.Date(2025, 3, 31, 0, 0, 0, 0, time.UTC)
t2 := t1.AddDate(0, -1, 0)
```

Here, we take the last day of March and attempt to subtract one month. What would you expect `t2` to be? The 31st of February? The 28th of February? Surprisingly, the result is the **3rd of March**.

### Why does this happen?

The reason for this behavior lies in how Go's `time` package handles date arithmetic. Instead of performing a strict "month subtraction," it subtracts the number of days in the month being removed. In this case, February 2025 has 28 days, so subtracting one month from March 31st effectively moves the date back by 28 days, landing on March 3rd.

This issue also occurs in the opposite direction:

```go
t1 := time.Date(2025, 3, 31, 0, 0, 0, 0, time.UTC)
t2 := t1.AddDate(0, 1, 0)
```

Here, instead of arriving at April 30th, the result is **May 1st** because 31 days (the length of March) are added.

### Handling the Issue

There is no straightforward way to universally prevent this behavior, but you can check the number of days in a given month to better handle cases where dates might fall outside expected ranges:

```go
func daysIn(m time.Month, year int) int {
    return time.Date(year, m+1, 0, 0, 0, 0, 0, time.UTC).Day()
}
```

However, in most cases, it's better to **rethink the logic** that leads to such edge cases. For instance, if your application involves scheduling, consider giving users the option to schedule tasks as **"X days after the start of the month"** or **"X days before the end of the month"** instead of allowing arbitrary day selections.

This approach ensures clarity for users and avoids unexpected shifts in date calculations:

```go
threeDaysAfterStart := time.Date(year, month, 3, 0, 0, 0, 0, time.UTC)
threeDaysBeforeEnd := time.Date(year, month+1, -3, 0, 0, 0, 0, time.UTC)
```

---

## `nil` and Type Mismatches

While working on an older project, I encountered an issue where error handling was not done properly. The code followed this pattern:

```go
func f1() *wire.Error {
    //...
    return nil
}

func f2() error {
    return f1()
}

func main() {
    err := f2()
    if err == nil {
        fmt.Println("error is nil")
        return
    }
    fmt.Println(err.Error()) // Panic occurs here
}
```

At first glance, there seems to be nothing wrong. The `if err == nil` check should prevent any issues, right? However, the program **still panicked with a nil-pointer error**.

### Understanding `wire.Error`

Before diving into the root cause, let's define `wire.Error`. This is a custom error type that provides additional information beyond a standard error:

```go
type wire.Error struct {
    Code    int
    Message string
}

func (e *wire.Error) Error() string {
    return fmt.Sprintf("Error %d: %s", e.Code, e.Message)
}
```

### Understanding the Issue

The problem stems from Go's handling of `nil` values and interfaces. In Go, every value has a specific type, including `nil`. If we annotate the types in the conditional check, it looks like this:

```go
if [*wire.Error, nil] == [error, nil] {
    ...
}
```

Since `error` is an interface type and `*wire.Error` is a concrete pointer type, these values are **not** considered equal, even though both hold `nil`.

### Root Cause

The issue originates in `f2`, which simply returns the result of `f1()` without checking for `nil`. When `f1()` returns `(*wire.Error)(nil)`, `f2` casts it to `error`. At this point, the **interface itself is not nil**, even though the underlying pointer is. This subtle difference leads to the panic when trying to call `err.Error()`.

### The Fix

To avoid this issue, **always check for nil before returning an interface type**. The corrected version of `f2` ensures that a true `nil` value is returned if `f1()` returns `nil`:

```go
func f2() error {
    err := f1()
    if err != nil {
        return err
    }
    return nil // Explicitly return a true nil
}
```

With this fix, `err == nil` behaves as expected, and the program will no longer panic.

---

## Conclusion

Go is a powerful and efficient language, but it has its quirks—especially when dealing with date calculations and `nil` types. Understanding these behaviors can help prevent subtle bugs and unexpected issues in your applications.

These are just two of the many interesting quirks I have encountered. Stay tuned for the next part, where I'll cover more unique behaviors in Go!
