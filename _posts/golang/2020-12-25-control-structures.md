---
title: "Golang: Control structures"
date: 2020-12-25 00:07:00 +0800
categories: [Language Learning, Golang]
tags: [golang, learning]
---

# Control structures
Taken from [Effective Go](https://golang.org/doc/effective_go.html#control-structures): 
> The control structures of Go are related to those of C but differ in important ways.
> There is no do or while loop, only a slightly generalized for; switch is more flexible;
> if and switch accept an optional initialization statement like that of for;
> break and continue statements take an optional label to identify what to break or continue;
> and there are new control structures including a type switch and a multiway communications
> multiplexer, select. The syntax is also slightly different:
> there are no parentheses and the bodies must always be brace-delimited.

## If statement

In Go there's no need for parenthesis in If statements. __Open curly braces must be__
__on the same line with the `if` keyword__. (due to Go's "semicolon insertion" feature)

```go
if x> > 0 {
    return y
}
```
```go
if err := myPack.SomeOperation(); err != nil {
    log.Print(err)
    return err
}
```

## Switch statement

### Writing if-else chain as a switch
Go's `switch` is more general than C's. It's possible (and idiomatic) to write 
a `if-else-if-else` chain as a `switch`.

```go
func unhex(c byte) byte {
    switch {
    case '0' <= c && c <= '9':
        return c - '0'
    case 'a' <= c && c <= 'f':
        return c - 'a' + 10
    case 'A' <= c && c <= 'F':
        return c - 'A' + 10
    }
    // notice how there is no `break`. In Go a case wouldn't fall through
    // the next case like C would, even without `break`
    return 0
}
```

### No automatic fall-through

Like we said in the above code, __Go does not have automatic fall-through__.
This means we don't have to write `break` for each and every `case`.

But what if we actually needed multiple case to land on the same piece of code?
Fortunately, Go provides us with 'comma-separated lists'.

### Comma-separated lists

```go
func shouldEscape(c byte) bool {
    switch c {
    case ' ', '?', '&', '=', '#', '+', '%':
        return true
    }
    return false
}
```

This is the equivalence of the following C code:

```c++
bool shouldEscape(char c) {
    switch(c) {
        case ' ':
        case '?':
        case '&':
        case '=':
        case '#':
        case '+':
        case '%':
            return true;
        default:
            return false;
    }
}
```

### Type switch

// Working in Progress...

## 'break' with label

// Working in Progress...

## Defer statement

Go's defer statement schedules a function call (the _deferred_ function) to be run 
immediately before the function executing the defer returns.  
It's an unusual but effective way to deal with situations such as resources that must 
be released regardless of which path a function takes to return.

> This feature is quite unique and C++ doesn't have anything like it.  
> For this reason, we don't really have anything to compare it against.  
> For more advanced usage of the statement definitely checkout
> [Effective Go#Defer](https://golang.org/doc/effective_go.html#defer)

In this example, `f` will always be closed wherever we return in the function as long
as it's after the `defer f.Close()` statement.
```go
func Contents(filename string) (string, error) {
    f, err := os.Open(filename)
    if err != nil {
        return "", err  // before defer statement, so wouldn't trigger `f.Close()`
    }
    defer f.Close()  // f.Close() will run when we're finished.
    
    // do something...

    if /* ... */ {
        return nil, "error!" // f will be closed if we return here.
    }

    return string(result), nil // f will be closed if we return here.
}
```


Taken from [Effective Go](https://golang.org/doc/effective_go.html#defer):
> For programmers accustomed to block-level resource management from other languages,
> defer may seem peculiar, but its most interesting and powerful applications come 
> precisely from the fact that it's not block-based but function-based.



