---
title: "Golang: Functions"
date: 2020-12-22 15:02:00 +0800
categories: [Language Learning, Golang]
tags: [golang, learning]
---

# Functions

Reader of this blog is assumed to have some basic programming skills. So in this
series, we will not get into basic things like how function works. Because it's
basically the same for every languages, including Go.

Instead, we will focus on comparing the same feature in Go and in C++ and see where the differences are.

## Declaration

Here's a Go function declaration:
```go
func add(x int, y int) int {
	return x + y
}
```

Noted that in Go, type comes after variable/function name, which is different from all the
other "C-like" languages, eg. C, C++, C#, Java.

## Unique syntaxes and syntax sugars

### 1. Writing type only once for consecutive names with the same type
```go
x int, y int

// can be written as

x, y int
```
The same works for function parameters as well: 
```go
func add(x int, y int) int { return x + y }

// can be written as

func add(x, y int) int { return x + y }
```

### 2. Multiple return value
```go
func split(sum int) (int, int) {
	x := sum * 4 / 9		// short variable declaration
	return x, sum - x		// return two values
}
```
### 3. Named return value
The above example can be written like this:
```go
func split(sum int) (x int, y int) {  // give names to our return values.
	x = sum * 4 / 9			// assign return value just like you would a normal variable
	y = sum - x				// use `x` like an ordinary variable
	return					// return all values assigned before
}
```
you can also omit consecutive identical types except for the last one:
(just like variables)
```go
func split(sum int) (x, y int) { // <-- notice here
	x = sum * 4 / 9
	y = sum - x
	return
}
```
Once you assigned all the return values, do a simple `return` without anything
after it, Go will help you return all the values you assigned before.
