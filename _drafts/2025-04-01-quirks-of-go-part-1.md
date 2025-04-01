---
layout: post
title: "Quirks of go - Part 1"
subtitle: "Lets go over some quirky behaviour of golang..."
date:   2025-04-01 13:00:00 +0200
categories: general
---

# Quirks of go - Part 1
In my recent years of developing golang applications I found ran into some weird behaviours a few times. Recently I started collecting them and will present you some of the weird go quirks that I found and how to work around them.

## time.AddDate
Lets start with `time.AddDate`. Quite a simple function you would think, it takes your current date and adds or subtracts the given number of days, months or years. But what happens if the target date does not exists? Lets go with an example:
```
t1 := time.Date(2025, 3, 31, 0, 0, 0, 0, time.UTC)
t2 := t1.AddDate(0, -1, 0)
```
So we take a the last day of march and go back for one month. So what would you expect t2 to be? 31st of February? 28th of February? All wrong, the result is 3rd of March. But WTF? did we not tell it to go back one month? What did happen?
What happened there is actually pretty simple to explain but not intuitive (at least to me). What is happening essentially is that it subtracts the number of days that are in the month that you want to subtract.
So in this case we removing the days of february which is 28 days from the current date which results in 3rd of March.

This quirk also exists if we do it the other way round:
```
t1 := time.Date(2025, 3, 31, 0, 0, 0, 0, time.UTC)
t2 := t1.AddDate(0, -1, 0)
```
When we execute that we actually get 1st of May. So we added the number of days in March to the time.

In my experience there is no easy general way of working around that. So you can use functions like:
```
func daysIn(m time.Month, year int) int {
    return time.Date(year, m+1, 0, 0, 0, 0, 0, time.UTC).Day()
}
```
To check how many days a specific month has to then handle the dates that are falling out of that range.

But in general the better approach is to rethink your solution to eliminate those weird cases because also users could get confused for example "what happens in February if I schedule this to run at the 31st of each month?".
For example if you allow your users to create a schedule it might make sense to instead of giving them the option to freely select a start-date for a monthly interval to give them the option to choose between "X days after month start" and "X days before month ends".
Both of those can be easily and reliably implemented and are also clear to the user:
```
3daysAfterMonthStarts := time.Date(year, month, 3, 0, 0, 0, 0, time.UTC)
3daysBeforeMonthEnds := time.Date(year, month+1, -3, 0, 0, 0, 0, time.UTC)
```

## Nil-Types
I was working on an old project and the error handling there was not done super well and we had a lot of instances where we would simply return the function result without checking like this:
```
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
    fmt.Println(err.Error())
}
```
So if you are not seeing the problem I feel you, as I did not see any problems with that as well but somehow the println was failing with a nil-pointer panic. But didn't we check for nil just a few lines before that? Yes, but that is where nil-types are coming into perspective.
Every value in golang has a type attached to it. Even nil-values. So if we look at the if-condition and annotate the types to it it looks like this:
```
if [*wireError, nil] == [error, nil] {
    ...
}
```
and suddenly it is clear that those values are not the same. But how did this happen? 
The problem lies within the `f2` function. In this function we are just returning the result of `f1` without checking for a nil value. So if the `[*wire.Error, nil]` value gets returned from `f1` we are just passing it down, which causes our unexpected type-missmatch later on.

So how do we fix it? Pretty simple by not piping functions but handling them and returning only the result or nil. The `f2` function was rewritten to this:
```
func f2() error {
    err = f1()
    if err != nil {
        return err
    }
    return nil // implicitly typed by function signature
}
```
and with that everything suddenly behaved as expected. So be careful of what you are returning in your functions as nil-values might cause an issue down the line.
