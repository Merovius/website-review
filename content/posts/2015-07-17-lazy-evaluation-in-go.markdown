---
date: "2015-07-17T19:31:10Z"
tags:
- golang
- programming
aliases:
- "/2015/07/17/lazy-evaluation-in-go.html"
feedID: "https://blog.merovius.de//2015/07/17/lazy-evaluation-in-go.html"
title: Lazy evaluation in go
summary: I did lazy evaluation in go.
---

**tl;dr: I did [lazy evaluation in go](https://godoc.org/merovius.de/go-misc/lazy)**

A small pattern that is usefull for some algorithms is [lazy
evaluation](https://en.wikipedia.org/wiki/Lazy_evaluation). Haskell is famous
for making extensive use of it. One way to emulate goroutine-safe lazy
evaluation is using closures and [the sync-package](https://godoc.org/sync):

```go
type LazyInt func() int

func Make(f func() int) LazyInt {
	var v int
	var once sync.Once
	return func() int {
		once.Do(func() {
			v = f()
			f = nil // so that f can now be GC'ed
		})
		return v
	}
}

func main() {
	n := Make(func() { return 23 }) // Or something more expensive…
	fmt.Println(n())                // Calculates the 23
	fmt.Println(n() + 42)           // Reuses the calculated value
}
```

This is not the fastest possible code, but it already has less overhead than
one would think (and it is pretty simple to deduce a faster implementation from
this). I have implemented a [simple command](https://godoc.org/merovius.de/go-misc/cmd/go-lazy),
that generates these implementations (or rather, more optimized ones based on
the same idea) for different
[types](https://godoc.org/merovius.de/go-misc/lazy).

This is of course just the simplest use-case for lazynes. In practice, you might also want Implementations of Expressions

```go
func LazyAdd(a, b LazyInt) LazyInt {
	return Make(func() { return a() + b() })
}
```

or lazy slices (slightly more complicated to implement, but possible) but I
left that for a later improvement of the package (plus, it makes the already
quite big API even bigger) :)
