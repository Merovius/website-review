---
date: "2014-09-12T17:10:28Z"
tags:
- golang
- programming
aliases:
- "/2014/09/12/the-four-things-i-miss-from-go.html"
feedID: "https://blog.merovius.de//2014/09/12/the-four-things-i-miss-from-go.html"
title: The four things I miss about go
summary: A short list of four things that I might want to add to go (but probably wouldn't).
---
As people who know me know, my current favourite language is
[go](http://golang.org/). One of the best features of go is the lack of
features. This is actually the reason I preferred C over most scripting
languages for a long time – it does not overburden you with language-features
that you first have to wrap your head around. You don't have to think for a
while about what classes or modules or whatever you want to have, you just
write your code down and the (more or less) entire language can easily fit
inside your head. One of the best writeups of this (contrasting it with python)
was done by Gustavo Niemeyer in a
[blogpost](http://blog.labix.org/2012/06/26/less-is-more-and-is-not-always-straightforward)
a few years back.

So when I say, there are a few things popping up I miss about go, this does not
mean I wish them to be included. I subjectively miss them and it would
definitely make me happy, if they existed. But I still very much like the go
devs for prioritizing simplicity over making me happy.

So let's dig in.

1. [Generics](#generics)
2. [Weak references](#weak-references)
3. [Dynamic loading of go code](#dynamic-loading-of-go-code)
4. [Garbage-collected goroutines](#garbage-collected-goroutines)

## Generics

So let's get this elephant out of the room first. I think this is the most
named feature lacking from go. They are asked so often, they have their own
entry in the [go FAQ](http://golang.org/doc/faq#generics). The usual answers
are anything from "maybe they will get in" to "I don't understand why people
want generics, go has generic programming using interfaces". To illustrate one
shortcoming of the (current) interface approach, consider writing a (simple)
graph-algorithm:

```go
type Graph [][]int

func DFS(g Graph, start int, visitor func(int)) {
	visited := make([]bool, len(g))

	var dfs func(int)
	dfs = func(i int) {
		if visited[i] {
			return
		}
		visitor(i)
		visited[i] = true
		for _, j := range g[i] {
			dfs(j)
		}
	}

	dfs(start)
}
```

This uses an adjacency list to represent the graph and does a recursive
depth-first-search on it. Now imagine, you want to implement this algorithm
generically (given, a DFS is not really hard enough to justify this, but you
could just as easily have a more complex algorithm). This could be done like this:

```go
type Node interface{}

type Graph interface {
	Neighbors(Node) []Node
}

func DFS(g Graph, start Node, visitor func(Node)) {
	visited := make(map[Node]bool)

	var dfs func(Node)
	dfs = func(n Node) {
		if visited[n] {
			return
		}
		visitor(n)
		visited[n] = true
		for _, n2 := range g.Neighbors(n) {
			dfs(n2)
		}
	}

	dfs(start)
}
```

This seems simple enough, but it has a lot of problems. For example, we loose
type-safety: Even if we write `Neighbors(Node) []Node` there is no way to tell
the compiler, that these instances of `Node` will actually always be the same.
So an implementation of the graph interface would have to do type-assertions
all over the place. Another problem is:

```go
type AdjacencyList [][]int

func (l AdjacencyList) Neighbors(n Node) []Node {
	i := n.(int)
	var result []Node
	for _, j := range l[i] {
		result = append(result, j)
	}
	return result
}
```

An implementation of this interface as an adjacency-list actually performs
pretty badly, because it can not return an `[]int`, but must return a `[]Node`,
and even though `int` satisfies `Node`, `[]int` is not assignable to `[]Node`
(for good reasons that lie in the implementation of interfaces, but still).

The way to solve this, is to always map your nodes to integers. This is what
the standard library does in the
[sort-package](http://golang.org/pkg/sort/#Interface). It is exactly the same
problem. But it might not always be possible, let alone straightforward, to do
this for Graphs, for example if they do not fit into memory (e.g. a
web-crawler). The answer is to have the caller maintain this mapping via a
`map[Node]int` or something similar, but… meh.

## Weak references

I have to admit, that I am not sure, my use case here is really an important or
even very nice one, but let's assume I want to have a database abstraction that
transparently handles pointer-indirection. So let's say I have two tables T1
and T2 and T2 has a foreign key referencing T1. I think it would be pretty
neat, if a database abstraction could automatically deserialize this into a
pointer to a T1-value `A`. But to do this. we would a) need to be able to
recognize `A` a later Put (so if the user changes `A` and later stores it, the
database knows what row in T1 to update) and b) hand out the *same* pointer, if
another row in T2 references the same id.

The only way I can think how to do this is to maintain a `map[Id]*T1` (or
similar), but this would prevent the handed out values to ever be
garbage-collected. Even though there a
[hacks](https://groups.google.com/forum/#!topic/golang-nuts/1ItNOOj8yW8/discussion)
that would allow some use cases for weak references to be emulated, I don't see
how they would work here.

So, as in the case of generics, this mainly means that some elegant APIs are
not possible in go for library authors (and as I said, in this specific case it
probably isn't a very good idea. For example you would have to think about what
happens, if the user gets the same value in two different goroutines from the
database).

## Dynamic loading of go code

It would be useful to be able to dynamically load go code at runtime, to build
plugins for go software. Specifically I want a good go replacement for
[jekyll](http://jekyllrb.com/) because I went through some ruby-version-hell
with it lately (for example `jekyll serve -w` still does not work for me with
the version packaged in debian) and I think a statically linked go-binary would
take a lot of possible pain-points out here. But plugins are a really important
feature of jekyll for me, so I still want to be able to customize a page with
plugins (how to avoid introducing the same version hell with this is another
topic).

The currently recommended ways to do plugins are a) as go-packages and
recompiling the whole binary for every change of a plugin and b) using
sub-processes and [net/rpc](http://golang.org/pkg/net/rpc).

I don't feel a) being a good fit here, because it means maintaining a separate
binary for every jekyll-site you have which just sounds like a smallish
nightmare for binary distributions (plus I have use cases for plugins where even
the relatively small compilation times of go would result in an intolerable
increase in startup-time).

b) on the other hand results in a lot of runtime-penalty: For example I can not
really pass interfaces between plugins, let alone use channels or something and
every function call has to have its parameters and results serialized and
deserialized.  Where in the same process I can just define a transformation
between different formats as a `func(r io.Reader) io.Reader` or something, in
the RPC-context I first have to transmit the entire file over a socket, or have
the plugin-author implement a `net/rpc` server himself and somehow pass a
reference to it over the wire. This increases the burden on the plugin-authors
too much, I think.

Luckily, it seems there seems to be
[some thought](https://groups.google.com/forum/#!topic/golang-dev/0_N7DLmrUFA)
put forward recently on how to implement this, so maybe we see this in the
nearish future.

## Garbage-collected goroutines

Now, this is the only thing I really don't understand why it is not part of the
language. Concurrency in go is a first-class citizen and garbage-collection is
a feature emphasized all the time by the go-authors as an advantage. Yet, they
both seem to not play entirely well together, making concurrency worse than it
has to be.

Something like the standard example of how goroutines and channels work goes a
little bit like this:

```go
func Foo() {
	ch := make(chan int)
	go func() {
		i := 0
		for {
			ch <- i
			i++
		}
	}()

	for {
		fmt.Println(<-ch)
	}
}
```

Now, this is all well, but what if we want to exit the loop prematurely? We
have to do something like this:

```go
func Foo() {
	ch := make(chan int)
	done := make(chan bool)
	go func() {
		i := 0
		for {
			select {
				case ch <- i:
					i++
				case <-done:
					return
			}
		}
	}()
	for {
		i := <-ch
		if i > 1000 {
			break
		}
		fmt.Println(i)
	}
}
```

Because otherwise the goroutine would just stay around for all eternity,
effectively being leaked memory. There are
[entire](http://youtu.be/f6kdp27TYZs) [talks](http://youtu.be/QDDwwePbDtw)
build around this and similar problems, where I don't really understand why. If
we add a `break` to our first version, `Foo` returns and suddenly, all other
references to `ch`, except the one the goroutine is blocking on writing to are
gone and can be garbage-collected. The runtime can already detect if all
goroutines are sleeping and we have a deadlock, the garbage-collector can
accurately see what references there are to a given channel, why can we not
combine the two to just see "there is absolutely *no* way, this channel-write
can *ever* succeed, so let's just kill it and gc all it's memory"? This would
have zero impact on existing programs (because as you can not get any
references to goroutines, a deadlocked one can have no side-effect on the rest
of the program), but it would make channels *so* much more fun to work with. It
would make channels as iterators a truly elegant pattern, it would simplify
[pipelines](http://blog.golang.org/pipelines) and it would possibly allow a
myriad other use cases for channels I can not think of right now. Heck, you
could even think about (not sure if this is possible or desirable) running any
deferred statements, when a goroutine is garbage-collected, so all other
resources held by it will be correctly released.

This is the *one* thing I really wish to be added to the language. Really
diving into channels and concurrency right now is very much spoiled for me
because I always have to think about draining every channel, always think about
what goroutine closes what channels, passing cancellation-channels…
