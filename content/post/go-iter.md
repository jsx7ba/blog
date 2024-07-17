---
title: "Iterators in Go 1.23"
summary: "Go support for user defined iterators."
date: "2024-07-07"
tags:
    - "go"
    - "iterators"
    - "development"
ShowCodeCopyButtons: true
ShowToc: true
---

## Introduction

Go 1.23 introduces iterator functions as a mechanism to support user defined iteration compatible with `range`.  This article examines the previous way of iterating and how the new way, "function iterators", work. 

## The Old Ways

In previous Go versions, no mechanism existed for user defined types to utilize iteration with `range`.  This means iteration over custom types was accomplished in two ways: returning a slice over a complete data set or ranging over a channel.  

Slicing over an entire data set can be memory intensive, and may not be possible in all cases.  Ranging over channels can easily create goroutine leaks.  Keep in mind that go doesn't garbage collect goroutines. Goroutines need to exit on their own or terminate due to a panic.

```go {lineNos=table,hl_lines=[3,13]} 
func testIter() chan int {
    intChan := make(chan int)
    go func() {
        defer close(intChan)
        intChan <- 1
        intChan <- 2
        intChan <- 3
    }()
    return intChan
}

for s := range testIter() {
    if s == 2 {
        break  // don't do this
    }
    fmt.Println(s)
}
```

The code above leaks a goroutine. The goroutine(line 3) is blocked sending a value to a channel that will never be 
received after the _break_ statement(line 13).

## The New Way
Go 1.23 introduces a formal way to implement user-defined iterators in the [`iter`](https://pkg.go.dev/iter@go1.23rc1) package:

> "An iterator is a function that passes successive elements of a sequence to a callback function, conventionally named yield."

[Seq](https://pkg.go.dev/iter@go1.23rc1#Seq) is one of the types the `iter` package provides, and break it down based on the definition above.

```go
type Seq[V any] func(yield func(V) bool)
```

The `Seq` type has a type parameter `V`, which the call back function `yield` expects to receive as a parameter.  It is the developer's responsibility to implement the function (_'outer function'_ from here on) that receives the yield callback function.   

Here's an example:

```go {lineNos=table}
func simpleIter() iter.Seq[int] {
    // implement iter.Seq[int]
    return func(yield func(int) bool) { 
        if !yield(1) {
            return
        }

        if !yield(2) {
            return
        }
    }
}

for x := range simpleIter() {
    if x == 2 { 
        break;
    }
    fmt.Println(x)
}
```

At a high level, is that the compiler is taking the for-loop body and using that as a call back to the outer function.  More on this in the next section.  Note that any `break` statement, `return` statement, `goto` statement that leaves the loop, or panic in the `for` loop will cause the yield function to return false. The iterator implementation must return when this happens. Failure to handle false return values will cause a panic: 

`runtime error: range function continued iteration after function for loop body returned false.`

### Rewrites Make it Work
The go compiler rewrites for loops with an iterator function to code without a iterator function.  The contents of the iterator function will be called by a yield function generated from the `for` loop body. The code in the previous example is rewritten to something like this:

```go {lineNos=table}
{
    yield := func(v int) bool {
        if x == 2 {      // the body of the for loop
            return false // break converted return false
        }
        fmt.Println(v)  
        return true      // added by the compiler
    }
    
    if !yield(1) {
        goto end
    }

    if !yield(2) {
        goto end
    }

end:
}
```

Note this example is very trivial but it does serve as a decent mental model when writing iterators.  See the [rewrite code](https://go.googlesource.com/go/+/refs/changes/41/510541/7/src/cmd/compile/internal/rangefunc/rewrite.go) for a great explanation of the complexity left out here.

### Example Single Variable Push Iteration
Here's a more substantial version of an iterator that splits a string.  Each call to the iterator yields another substring. This is a better solution than returning a slice for any moderately large string. 

```go  {lineNos=table}
func stringSplit(delim byte, value string) iter.Seq[string] {
    return func(yield func(string) bool) {
        first := 0
        for i := 0; i != len(value); i++ {
            if value[i] == delim {
                if !yield(value[first:i]) {
                    break
                }
                first = i + 1 // move past the delimiter
            }
        }

        if !yield(value[first:]) {
            return
        }
    }
}

for s := range stringSplit(',', "one,two,three") {
    fmt.Println(s)
}
```

The output from the example:
```text
one
two
three
```

### Two variable Push Iteration
The `iter` package also provides a two-parameter function to iterate on a pair of variables.  

```go
type Seq2[K, V any] func(yield func(K, V) bool)
```

This has the same mechanics as the single variable iteration, [described above](#example-single-variable-push-iteration). 

```go {lineNos=table}
func stringSplit2(delim byte, value string) iter.Seq2[int, string] {
    return func(yield func(int, string) bool) {
        first, count := 0, 1
        for i := 0; i != len(value); i++ {
            if value[i] == delim {
                if !yield(count, value[first:i]) {
                    break
                }
                first = i + 1 // add one to move past the delimiter
                count++
            }
        }
        if !yield(count, value[first:]) {
            return
        }
    }
}

for i, s := range stringSplit2(',', "one,fish,two,fish") {
    fmt.Printf(i, s)
}
```

The output from the example:
```text
1: one
2: fish
3: two
4: fish
```

### Pull Iteration

So far, this article has only examined push iteration.  Push iteration occurs when the iterator determines when the loop finishes.  Pull iteration occurs when the loop asks the iterator if it is done.  The `iter` package provides functions [`Pull`](https://pkg.go.dev/iter@go1.23rc1#Pull) and [`Pull2`](https://pkg.go.dev/iter@go1.23rc1#Pull2) which correspond to the [`Seq`](https://pkg.go.dev/iter@go1.23rc1#Seq) and [`Seq2`](https://pkg.go.dev/iter@go1.23rc1#Seq2) types.

```go {lineNos=table}
func simpleIter() iter.Seq[int] {
    return func(yield func(v int) bool) {
        if !yield(1) {
            return
        }
        if !yield(2) {
            return
        }
        if !yield(3) {
            return
        }
    }
}

// convert the push iterator to a pull iterator
next, stop := iter.Pull[int](simpleIter())
defer stop()
for {
    value, ok := next()
    if !ok {
        break
    }
    fmt.Println("value: ", value)
}
```

The `iter` package documentation gives an [example](https://pkg.go.dev/iter@go1.23rc1#hdr-Pulling_Values) of using Pull when composing two Push iterators together.  I can't think of another use case becuase a push iterator must always be written first.

## Summary and key points
Iterator functions are a clever solution to allowing custom iterators in a backwards compatible way.  They're easy to reason about and the compiler shouldn't generate any surprising code to accommodate them.

* Iterator functions should return `iter.Seq` or `iter.Seq2`
* Iterator functions must exit when yield returns false
* Follow the [naming conventions](https://pkg.go.dev/iter@go1.23rc1#hdr-Naming_Conventions) for custom types


## Sources
* [Rangefunc Experiment](https://go.dev/wiki/RangefuncExperiment)
* [iter Package](https://pkg.go.dev/iter@go1.23rc1)
* [The Go Programming Language Specification](https://go.dev/ref/spec)

