---
layout: post
title: "Quirks of Go - Part 2"
subtitle: "Unexpected behavior with nil pointers and slice appends"
date:   2025-04-01 16:00:00 +0200
categories: general golang quirks-of-go
---

# 1. Method Calls on Nil Pointers
Coming from an object-oriented background, one of the quirks in Go that really caught me off guard was that it's possible to call a method on a nil pointer. Let's look at an example:

```go
type foo struct {
    Text string
}

func (f *foo) Bar() string {
    return f.Text
}

func main() {
    var f *foo
    text := f.Bar()
    fmt.Println(text)
}
```

You might expect the call to `f.Bar()` in `main()` to immediately trigger a nil pointer panic, since `f` is clearly nil. But surprisingly, that's not what happens.

In Go, calling a method on a nil receiver is valid—**as long as the method doesn't dereference the pointer**. In the example above, the panic doesn't occur at the point of calling `f.Bar()`, but inside the `Bar()` method when it tries to access `f.Text`. That's where the nil dereference happens.

**What's the takeaway?**  
If you don't explicitly need pointer semantics, avoid them. But if you do use pointer receivers, always check for nil before accessing struct fields or calling other methods.


# 2. Appending to Slices Created from Sub-Slices
Another head-scratcher in Go involves appending to slices that are derived from parts of other slices. It's easy to make assumptions here that can lead to subtle bugs. Consider the following code:

```go
func getElementsAndAppend(originalSlice []int, numOfElementsToGet int, elementToAppend int) []int {
    result := originalSlice[:numOfElementsToGet]
    result = append(result, elementToAppend)
    return result
}

func main() {
    numbers := []int{1, 2, 3, 4, 5}
    result := getElementsAndAppend(numbers, 2, 99)
    fmt.Println(numbers)
    fmt.Println(result)
}
```

You might expect the output to be:
```
[1 2 3 4 5]
[1 2 99]
```

But the actual result is:
```
[1 2 99 4 5]
[1 2 99]
```

Why? Because when you slice `originalSlice[:2]`, you're creating a new slice that shares the **same underlying array** as `originalSlice`. When `append` is called and there's still capacity left in that array, Go reuses it. So instead of allocating a new array, `append` modifies the existing one, in this case overwriting the third element with `99`.

To avoid this, you need to explicitly create a new slice with its own backing array and copy the contents over:

```go
func getElementsAndAppend(originalSlice []int, numOfElementsToGet int, elementToAppend int) []int {
    result := make([]int, numOfElementsToGet)
    copy(result, originalSlice[:numOfElementsToGet])
    result = append(result, elementToAppend)
    return result
}
```

Now, the output is exactly as expected:
```
[1 2 3 4 5]
[1 2 99]
```

# Conclusion
Go is simple, but it's not without its traps, especially if you're coming from other languages. Understanding how pointers and slices behave under the hood can help you avoid unexpected bugs and write safer, more predictable code.

Stay tuned for Part 3!
