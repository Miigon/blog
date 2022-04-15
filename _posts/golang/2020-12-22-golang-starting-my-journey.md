---
title: "Golang: Starting my journey"
date: 2020-12-22 10:22:00 +0800
categories: [Language, Golang]
tags: [golang, learning]
---

# Golang

> Disclaimer: This series is meant to compare the features and quirks of Golang with 
> other languages like C, because that's basically how I learned the language. However, 
> this means that the series might not be complete and detailed enough to be a tutorial 
> for beginners.

[Golang Wikipedia](https://en.wikipedia.org/wiki/Go_(programming_language))  
[Golang Official Documentation](https://golang.org/doc/)  
[How to Write Go Code](https://golang.org/doc/code.html)  
For advanced devs:  [The Go Programming Language Specification](https://golang.org/ref/spec)  
For beginners: 		[Golang Official Tour (Highly recommended)](https://tour.golang.org/)  
For best practices: [Effective Go (Highly recommended)](https://golang.org/doc/effective_go.html)  

Go is a statically typed, compiled programming language with memory safety, garbage
collection, structual typing, and CSP-style concurrency.

It's simple, robust and efficient. We see more and more adaptation in recent years,
in big companies like Tencent, Bilibili, Alibaba and ByteDance.


# About the series

In this article series, I will document my learning process of the Go language.
At the time of writing this article, I have very little Go language experience.  
However, I do have a good amount of C++, C# and Python experiences.

I do think that the best place to get information regarding a new language, library
or framework is the official documentation and/or wiki, so in this series, most of the
information will come from official sources, instead of from someone else's blog posts
or other third party sources.  

Noted that this series is merely my own learning process, instead of a full tutorial
for the language. I might write a full tutorial in English and/or Chinese after this
one.

__In this particular article, We'll go through the process of installing Golang, as__
__well as compiling our first program.__


# Downloading and Installing

Here's the official download link of Golang: https://golang.org/dl/

If you are running Windows, you can download the installer at the link above.  
But if you are using Linux or macOS, it's highly recommended to install via
a **package manager**:  

For macOS with Homebrew:
```bash
> brew install go
```
For Ubuntu or Debian:
```bash
> sudo apt-get install golang
```

After installing, you should be able to check the installed version:
```
> go version
go version go1.15.6 darwin/amd64
```

_At the time of writting this article, the newest version is go 1.15.6_
_That's also the version this series will be based upon._

# How to compile and run programs

Here's a demo program from https://golang.org:
```go
package main

import "fmt"

func main() {
	fmt.Println("Hello, 世界")
}
```
Save the above code to file `helloworld.go`, then run the code by one of
two ways: 

## 1. go build

```bash
> go build helloworld.go        # default output: `./helloworld`
> ./helloworld
Hello, 世界
```

The default output filename is the same as the source file, just without the
`.go` extension.

You can also specify the output file name by using `-o`:
```bash
> go build -o hello helloworld.go
> ./hello
Hello, 世界
```

## 2. go run

`go run` can be see as a shortcut of `go build` and then `./helloworld`.  
Note: It still compiles the program just like normal.

```bash
> go run helloworld.go
Hello, 世界
```

__Don't mistake it as interpreting the code, it is NOT!__
It is just a shortcut command to **compile** and **run** the code in one step.

# Conclusion

In ths article we discussed how to install Go and compile our first program.  
Next article we will start discussing the syntax and grammar of Go.
