
+++
title = "golang build tags"
date = "2016-10-18T12:00:00+08:00"
tags = ["golang"]
categories = ["programming","golang"]
menu = ""
banner = "banners/go.jpeg"
+++

golang build tags introduction,and some tricks for it.

<!--more-->

# golang build tags

## Introduction
- A build constraint, also known as a build tag, is a line comment that begins

	```
	// +build
	```
**that lists the conditions under which a file should be included in the package**. Constraints may appear in any kind of source file (not just Go), but they must appear near the top of the file, preceded only by blank lines and other line comments. These rules mean that in Go files a build constraint must appear before the package clause.

- To distinguish build constraints from package documentation, a series of build constraints must be followed by a blank line.
- A build constraint is evaluated as the OR of space-separated options; each option evaluates as the AND of its comma-separated terms; and each term is an alphanumeric word or, preceded by !, its negation. That is, the build constraint:
	
	```
	// +build linux,386 darwin,!cgo

	```
corresponds to the boolean formula: 

	```
	// +build (linux AND 386) OR (darwin AND (NOT cgo))
	```
- A file may have multiple build constraints. The overall constraint is the AND of the individual constraints. That is, the build constraints:

	```
	// +build linux darwin
	// +build 386
	```
corresponds to the boolean formula:

	```
	// +build (linux OR darwin) AND 386
	```
- If a file's name, after stripping the extension and a possible _test suffix, matches any of the following patterns:
	
	```go
	*_GOOS
	*_GOARCH
	*_GOOS_GOARCH
	```
(example: source_windows_amd64.go) where **GOOS and GOARCH represent any known operating system and architecture values respectively, then the file is considered to have an implicit build constraint requiring those terms (in addition to any explicit constraints in the file)**.

- To keep a file from being considered for the build:

	```
	// +build ignore
	```
(any other unsatisfied word will work as well, but “ignore” is conventional.)

## Common Usage
- build the source for different platform.all the platforms defined by the `GOOS` and `GOARCH`are included by the `src/go/build/syslist.go`:

	```	golang
	package build

	const goosList = "android darwin dragonfly freebsd linux nacl netbsd openbsd plan9 solaris windows "
	const goarchList = "386 amd64 amd64p32 arm armbe arm64 arm64be ppc64 ppc64le mips mipsle mips64 mips64le mips64p32 mips64p32le ppc s390 s390x sparc sparc64 "
	```
	
## Tricks

- prodivde different implementation for different environment,but the environment may not be the `GOOS` or `GOARCH`.
- `appengine` the build tag has been used by many lib.it's neither `GOOS` or `GOARCH`.it's self-defined.
- we can define the build tag to build different binary software according to our self-defined build tags.like the appengine do .but we can not use the implicit way( `GOOS` or `GOARCH` included in the file name),it will not be recognized by the go compiler.
- we can use the explicit build tag comment.
- like this example:

main.go

```go 
package main
	
import (
	"fanfu-golang/buildtags/lib"
)
func main(){
	lib.Tell()
}

```
 
lib/dev.go

```go
// +build !prod
	
package lib
	
import "fmt"
	
func Tell(){
	fmt.Println("hello from dev")
}

```

lib/prod.go

```golang
// +build prod
	
package lib
	
import "fmt"
	
func Tell(){
	fmt.Println("hello from dev")
}

```
we can build with `go build` and execute.it will output.

```bash
$ go build
$ ./buildtags
hello from dev

```
but when we build with `go build --tags prod` and execute the binary,the output will be:

```bash
$ go build --tags prod
$ ./buildtags
hello from prod

```
 As to the above example,we use the !prod to match our default env build.Also,we can setup different build tags for different environment,and specify the build tags with every build without default to make it more explicit.

## EOF




