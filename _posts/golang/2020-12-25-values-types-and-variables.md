---
title: "Golang: Values, Types and Variables"
date: 2020-12-22 15:02:00 +0800
categories: [Language Learning, Golang]
tags: [golang, learning]
---

# Variables

Noted that in Go, type lies after variable/function name, which is different from all the
other "C-like" languages, eg. C, C++, C#, Java.

> Here's [a great article](https://blog.golang.org/declaration-syntax) on why the Go
> declaration syntax is the way it is (with type after names instead of before).
> In that article Rob Pike has already compared the Go syntax with C syntax so
> we won't be doing that here. Check out the article if you are interested.

## Declaration

Basic usage:
```go
var foo1 int                // variable declaration
var foo2, foo3 int          // multiple variable declaration
var foo2, foo3 int = 1, 2   // ... with initializers
```

With implicit types:
```go
var bar1, bar2 = 3.142, true    // type inferred from the right hand side
// `bar1` -> float64, `bar2` -> bool

// notice how variables with different types can be declared together this way.
```

Variable declaration with implicit types can be substituted for the more elegant
`Short variable declaration`:
```go
bar3 := "hello"             // short variable declaration
// equivalent to:
var bar3 = "hello"

bar4, bar5 := true, "world" // you can even do this
```

Grouped "factored" declaration (like "factored" import):
```go
var (
    foo1 uint = 12
    foo2 int = -3
    isBar bool = true
)
```

## Types

### Basic types
```go
bool

string

int  int8  int16  int32  int64
uint uint8 uint16 uint32 uint64 uintptr

byte // alias for uint8

rune // alias for int32
     // represents a Unicode code point

float32 float64

complex64 complex128
```
The `int`, `uint`, and `uintptr` types are usually 32 bits wide on 32-bit systems and 64 bits wide on 64-bit systems. When you need an integer value you should use `int` unless you have a specific reason to use a sized or unsigned integer type.

### Type conversion (or the lack of it)
__The expression T(v) converts the value v to the type T.__
```go
i := 42             // int
f := float64(i)     // float64
u := uint(f)        // uint
```
Unlike in C,
in Go __assignment between items of different type requires an explicit conversion__.

This means the following code:
```go
var i int = 42
var f float32 = i  // error
```
should not work. Because Go doesn't allow implicit type conversion.

## Constants
__Constants are declared like variables, but with the `const` keyword.__  
Constants can be character, string, boolean, or numeric values.  
Constants cannot be declared using the `:=` syntax.  
