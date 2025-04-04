---
layout: post
title: "Quirks of go - Part 2"
subtitle: "..."
date:   2025-04-01 16:00:00 +0200
categories: general golang quirks-of-go
---

1. function calls on nil-pointers
One thing that was really weird to me coming from an object oriented language was the fact that a function that a method on a pointer in go can be called even if the pointer is nil.
Let have an example:
```
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
What I would expect to happen is that in line 22 we get a nil-pointer-panic becuase f is nil but thats not the case.
What we actually get is a nil-pointer-panic in line 17. Because it is not a probem to call a function on a nil-pointer, the panic only occurs when we try to access properties of nil.
So what can we learn from this? first don't use pointers if you don't have to and second if you use pointers always check for nil.

2. appending to slices that were created from part of other slices
When working with slices this can be a real head-scratcher. Lets imagin we have the following scenario:
```
func getElementsAndAppend(originalSlice []int, numOfElementsToGet int, elementToAppend int) {
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
What would you expect?
```
[1 2 3 4 5]
[1 2 99]
```
But the real result is
```
[1 2 99 4 5]
[1 2 99]
```

So what happened there? When copying the slice the underlying array was reused. So the same memory is now used by both slices but they have a different sizes.
But that means if we append to the shorter slice it will overwrite items in the longer slice.

How do we solve that?
Instead of reusing the original slice we have to create a new one and actually copy over the elements. We can do that by rewriting the function as follows:
```
func getElementsAndAppend(originalSlice []int, numOfElementsToGet int, elementToAppend int) []int {
	result := make([]int, numOfElementsToGet)
	copy(result, originalSlice[:numOfElementsToGet])
	result = append(result, elementToAppend)

	return result
}
```
with that we have the expected output of 
```
[1 2 3 4 5]
[1 2 99]
```
