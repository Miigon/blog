---
title: "Golang: Packages - Introduction"
date: 2020-12-22 13:46:00 +0800
categories: [Language Learning, Golang]
tags: [golang, learning]
---

# Packages - Introduction

__Every Go program is made up of packages.__

> _Note: Do not get confused with Go modules, which is Go's dependency management_
> _system. A Go modules usually contains one or more Go packages._  
> _We will discuss Go modules later, for now, just remember that we are talking about_
> _packages and just pretend modules don't exist._

Programs start running in package `main`.

```go
package main

import (
	"fmt"
	"math/rand"
)

func main() {
	fmt.Println("My favorite number is", rand.Intn(10))
}
```
__Use `package` at the beginning of the code to specify the current package.__  
__Use `import` to import other packages.__


## Import

`import` can take the following two styles:
```go
import (
	"fmt"
	"math/rand"
)
```
```go
import "fmt"
import "math/rand"
```
The prior grouped style is the recommeded "good" style by Go.


## Exported names and non-exported names

In Go packages, only exported names can be accessed outside the same package.

__A name is exported if it *begins with a capital letter*.__

Examples:
- `Pizza` is an exported name since it starts with capital letter `P`
- `Pi` starts with a capital letter thus is an exported name from package `math`.
	(Accessible with `math.Pi`)
- `Println` is an exported name of package `fmt`
- `Foobar` will be an exported name if you ever use it in your 
	package.
- `foobar` however, will **NOT** be exported if you use it in your package. It'll be 
	"unexported" thus inaccessable from outside the package.

By designing the language this way, Go effectively forced you to name your functions
 and types using a standard and preselected naming style.  
("CamelCase" for exported names, "lower camelCase" for internal unexported names)

> Whether that's a good thing or a bad thing is up for debate, but this design
> decision ditched the need of the keyword `public`, `export` or `extern` commonly
> found in other languages, making programmers' lives easier. It also makes the naming
> in all packages more unified and predictable, which is a good thing.


## Package Scope

There's no "global" scope in Go, instead, all functions and variables that is not local is
in the package scope.

```go
package myPackage

var v1, v2, v3 int			// in package scope 'myPackage'
var Var1 int				// in package scope 'myPackage', exported

func myfunction() int {}    // also in package scope 'myPackage'

func AnotherFunc() int {}	// in package scope 'myPackage', but it's also "exported" so 
							// it is accessible outside of package 'myPackage' by using
							// `myPackage.AnotherFunc()`
```

Functions and variables within package scope is accessible within the whole package.
Whether a name is accessible from **outside** the package however, is determined by the
naming style of the name.  
See [Exported names and non-exported names](#exported-names-and-non-exported-names).

In this case, `AnotherFunc()` can be accessed in another package like this: 
```go
package main

import (
	"fmt"
	"path/to/myPackage"				// we will talk about this later
)

func main() {
	myPackage.AnotherFunc()			// works
	fmt.Println(myPackage.Var1)		// works
	// myPackage.myFunction()		// this doesn't work since `myFunction` is not exported
}
```
This is just for showcasing the difference between exported names and non-exported names,
as well as the idea that you can access functions and variables in other packages.

# Conclusion

We'll be covering more about packages in a future article. For now, knowing _how to use_
packages is enough.
