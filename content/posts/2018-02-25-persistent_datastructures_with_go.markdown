---
date: "2018-02-25T17:30:00Z"
tags:
- golang
- programming
aliases:
- "/2018/02/25/persistent_datastructures_with_go.html"
omitFeed: true
title: Persistent datastructures with Go
summary: I present a couple of patterns that help modelling persistent datastructures
  in Go. I also apply them to three examples.
---

I've recently taken a liking to [persistent datastructures](https://en.wikipedia.org/wiki/Persistent_data_structure).
These are datastructures where instead of mutating data in-place, you are
creating a new version of the datastructures, that shares most of its state
with the previous version. Not all datastructures can be implemented
efficiently like this, but those that do get a couple of immediate benefits -
keeping old versions around allows you to get cheap snapshotting and copying.
It is trivial to pass a copy to a different thread and you don't have to worry
about concurrent writes, as neither actually mutates any shared state.

Persistent datastructures are popular in functional programming languages, but
I also found the idea a useful tool to model datastructures in Go. Go's
interfaces provide a nice way to model them and make them easy to reason about.
In this post, I will try to illustrate this with a couple of examples.

There are four key ideas I'd like you to walk away with:

* Modeling datastructures as persistent (*if possible*) makes them easier to
  reason about.
* When you want to use sum types, try to think of the common properties you are
  trying to abstract over instead - put those in an interface.
* Separate out the required from the provided interface. Make the former an
  interface type, provide the latter as functions or a wrapper.
* Doing these allows you to add more efficient implementations later, when you
  discover they are necessary.

## Linked lists

This is more of an illustrative example, to demonstrate the techniques, than
actually useful. But one of the simplest datastructures existing are linked
lists: A list of nodes, where each node has a value and possibly a next node
(unless we are at the end of the List). In functional languages, you'd use a
sum type to express this:

```haskell
type List a = Node a (List a) -- either it's a node with a value and the rest of the list
            | End             -- or it's the end of the list
```

Go infamously does not have sum types, but we can use interfaces to instead.
The classical way would be something like

```go
type List interface {
  // We use an unexported marker-method. As nothing outside the current package
  // can implement this unexported method, we get control over all
  // implementations of List and can thus de-facto close the set of possible
  // types.
  list()
}

type Node struct {
  Value int
  Next List
}

func (Node) list() {}

type End struct {}

func (End) list() {}

func Value(l List) (v int, ok bool) {
  switch l := l.(type) {
  case Node:
    return l.Value, true
  case End:
    return 0, false
  default:
    // This should never happen. Someone violated our sum-type assumption.
    panic(fmt.Errorf("unknown type %T", l))
  }
}
```

This works, but it is not really idiomatic Go code. It is error-prone and easy
to misuse, leading to potential panics. But there is a different way to model
this using interfaces, closer to how they are intended. Instead of expressing
what a list is

> A list *is* either a value and a next element, or the end of the list

we say what we want a list to be able to *do*:

> A list has a current element and may have a tail

```go
type List interface {
  // Value returns the current value of the list
  Value() int
  // Next returns the tail of the list, or nil, if this is the last node.
  Next() List
}

type node struct {
  value int
  next  List
}

func (n node) Value() int {
  return n.value
}

func (n node) Next() List {
  return n.next
}

func New(v int) List {
  return node{v, nil}
}

func Prepend(l List, v int) List {
  return node{v, l}
}
```

This is a far more elegant abstraction. The empty list is represented by the
`nil` interface. We have only one implementation of that interface, for the
nodes. We offer exported functions to create new lists - potentially from
existing ones.

Note that the methods actually have `node` as a receiver, not `*node`, as we
often tend to do with structs. This fact makes this implementation a
*persistent* linked list. None of the methods can modify the list. So after
creation, the linked list will stay forever immutable. Even if you type-assert
to get to the underlying data, that would only provide you with a *copy* of the
data - the original would stay unmodified. The memory layout, however, is the
same - the value gets put on the heap and you are only passing pointers to it
around.

The beauty of this way to think about linked lists, is that it allows us to
amend it after the fact. For example, say we notice that our program is slow,
due to excessive cache-misses (as linked lists are not contiguous in memory).
We can easily add a function, that packs a list:

```go
type packed []int

func (p packed) Value() int {
  return p[0]
}

func (p packed) Next() List {
  if len(p) == 0 {
    return nil
  }
  return p[1:]
}

func Pack(l List) List {
  if l == nil {
    return nil
  }
  var p packed
  for ; l != nil; l = l.Next() {
    p = append(p, l.Value())
  }
  return p
}
```

The cool thing about this is that we can mix and match the two: For example,
we could prepend new elements and once the list gets too long, pack it and
continue to prepend to the packed list. And since `List` is an interface, users
can implement it themselves and use it with our existing implementation. So,
for example, a user could build us a list that calculates fibonacci numbers:

```go
type fib [2]int

func (l fib) Value() int {
  return l[0]
}

func (l fib) Next() List {
  return fib{l[1], l[0]+l[1]}
}
```

and then use that with functions that take a `List`. Or they could have a
lazily evaluated list:

```go
type lazy struct {
  o sync.Once
  f func() (int, List)
  v int
  next List
}

func (l *lazy) Value() int {
  l.o.Do(func() { l.v, l.next = l.f() })
  return l.v
}

func (l *lazy) Next() List {
  l.o.Do(func() { l.v, l.next = l.f() })
  return l.next
}
```

Note that in this case the methods need to be on a pointer-receiver. This
(technically) leaves the realm of persistent data-structures. While they
motivated our interface-based abstraction and helped us come up with a safe
implementation, we are not actually *bound* to them. If we later decide, that
for performance reasons we want to add a mutable implementation, we can do so
(of course, we still have to make sure that we maintain the safety of the
original). And we can intermix the two, allowing us to only apply this
optimization to part of our data structure.

I find this a pretty helpful way to think about datastructures.

## Associative lists

Building on linked lists, we can build a map based on [Association Lists](https://en.wikipedia.org/wiki/Association_list).
It's a similar idea as before:

```go
type Map interface {
  Value(k interface{}) interface{}
  Set(k, v interface{}) Map
}

type empty struct{}

func (empty) Value(_ interface{}) interface{} {
  return nil
}

func (empty) Set(k, v interface{}) Map {
  return pair{k, v, empty{}}
}

func Make() Map {
  return empty{}
}

type pair struct {
  k, v interface{}
  parent Map
}

func (p pair) Value(k interface{}) interface{} {
  if k == p.k {
    return p.v
  }
  return p.parent.Value(k)
}

func (p pair) Set(k, v interface{}) Map {
  return pair{k, v, p}
}
```

This time, we don't represent an empty map as `nil`, but add a separate
implementation of the interface for an empty map.  That makes the
implementation of `Value` cleaner, as it doesn't have to check the parent map
for `nil` -- but it requires users to call `Make`.

There is a problem with our `Map`, though: We cannot iterate over it. The
interface does not give us access to any parent maps. We could use
type-assertion, but that would preclude users from implementing their own. What
if we added a method to the interface to support iteration?

```go
type Map interface {
  Value(k interface{}) interface{}

  // Iterate calls f with all key-value pairs in the map.
  Iterate(f func(k, v interface{}))
}

func (empty) Iterate(func(k, v interface{})) {
}

func (p pair) Iterate(f func(k, v interface{})) {
  f(p.k, p.v)
  p.parent.Iterate(f)
}
```

Unfortunately, this still doesn't really work though: If we write multiple
times to the same key, `Iterate` as implemented would call `f` with all
key-value-pairs. This is likely not what we want.

The heart of the issue here, is the difference between the *required* interface
and the *provided* interface. We can also see that with `Set`. Both of the
implementations of that method look essentially the same and neither actually
depends on the used type. We could instead provide `Set` as a function:

```go
func Set(m Map, k, v interface{}) Map {
  return pair{k,v,m}
}
```

The lesson is, that some operations need support from the implementation, while
other operations can be implemented without it. The provided interface is the
set of operations we provide to the user, whereas the required interface is the
set of operations that we rely on. We can split the two and get something like this:

```go
// Interface is the set of operations required to implement a persistent map.
type Interface interface {
  Value(k interface{}) interface{}
  Iterate(func(k, v interface{}))
}

type Map struct {
  Interface
}

func (m Map) Iterate(f func(k, v interface{})) {
  seen := make(map[interface{}]bool)
  m.Interface.Iterate(func(k, v interface{}) {
    if !seen[k] {
      f(k, v)
    }
  })
}

func (m Map) Set(k, v interface{}) Map {
  return Map{pair{k, v, m.Interface}}
}
```

Using this, we could again implement a packed variant of `Map`:

```go
type packed map[interface{}]interface{}

func (p packed) Value(k interface{}) interface{} {
  return p[k]
}

func (p packed) Iterate(f func(k, v interface{})) {
  for k, v := range p {
    f(k, v)
  }
}

func Pack(m Map) Map {
  p := make(packed)
  m.Iterate(func(k,v interface{}) {
    p[k] = v
  })
  return m
}
```

## Ropes

A [Rope](https://en.wikipedia.org/wiki/Rope_(data_structure)) is a data
structure to store a string in a way that is efficiently editable. They are
often used in editors, as it is too slow to copy the complete content on every
insert operation. Editors also benefit from implementing them as persistent data
structures, as that makes it very easy to implement multi-level undo: Just have
a stack (or ringbuffer) of Ropes, representing the states the file was in after
each edit. Given that they all share most of their structure, this is very
efficient. Implementing ropes is what really bought me into the patterns
I'm presenting here. Let's see, how we could represent them.

<a href="https://en.wikipedia.org/wiki/Rope_(data_structure)#/media/File:Vector_Rope_example.svg">
  <img src="/assets/Vector_Rope_example.svg" alt='Rope representing the string "Hello_my_name_is_Simon"'>
</a>

A Rope is a binary tree with strings as leafs. The represented string
is what you get when you do a depth-first traversal and concatenate all the
leafs. Every node in the tree also has a *weight*, which corresponds to the
length of the string for leafs and the length of the left subtree for inner
nodes. This allows easy recursive lookup of the `i`th character: If `i` is less
than the weight of a node, we look into the left subtree, otherwise into the
right. Let's represent this:

```go
type Base interface {
  Index(i int) byte
  Length() int
}

type leaf string

func (l leaf) Index(i int) byte {
  return l[i]
}

func (l leaf) Length() int {
  return len(l)
}

type node struct {
  left, right Base
}

func (n node) Index(i int) byte {
  if w := n.left.Length(); i >= w {
    // The string represented by the right child starts at position w,
    // so we subtract it when recursing to the right
    return n.right.Index(i-w)
  }
  return n.left.Index(i)
}

func (n node) Length() int {
  return n.left.Length() + n.right.Length()
}

type Rope struct {
  Base
}

func New(s string) Rope {
  return Rope{leaf(s)}
}

func (r Rope) Append(r2 Rope) Rope {
  return Rope{node{r.Base, r2.Base}}
}
```

Note, how we did not actually add a `Weight`-method to our interface: Given
that it's only used by the traversal on inner nodes, we can just directly
calculate it from its definition as the length of the left child tree. In
practice, we might want to pre-calculate `Length` on creation, though, as it
currently is a costly recursive operation.

The next operation we'd have to support, is splitting a Rope at an index. We
can't implement that with our current interface though, we need to add it:

```go
type Base interface {
  Index(i int) byte
  Length() int
  Split(i int) (left, right Base)
}

func (l leaf) Split(i int) (Base, Base) {
  return l[:i], l[i:]
}

func (n node) Split(i int) (Base, Base) {
  if w := n.left.Length(); i >= w {
    left, right := n.right.Split(i-w)
    return node{n.left, left}, right
  }
  left, right := n.left.Split(i)
  return left, node{n.right, right}
}

func (r Rope) Split(i int) (Rope, Rope) {
  // Note that we return the wrapping struct, as opposed to Base.
  // This is so users work with the provided interface, not the required one.
  left, right := r.Split(i)
  return Rope{left}, Rope{right}
}
```

I think this code is remarkably readable and easy to understand - and that is
mostly due to the fact that we are reusing subtrees whenever we can. What's
more, given these operations we can implement the remaining three from the
wikipedia article easily:

```go
func (r Rope) Insert(r2 Rope, i int) Rope {
  left, right := r.Split(i)
  return left.Append(r2).Append(right)
}

func (r Rope) Delete(i, j int) Rope {
  left, right := r.Split(j)
  left, _ = left.Split(i)
  return left.Append(right)
}

func (r Rope) Slice(i, j int) Rope {
  r, _ = r.Split(j)
  _, r = r.Split(i)
  return r
}
```

This provides us with a fully functioning Rope implementation. It doesn't
support everything we'd need to write an editor, but it's a good start that was
quick to write. It is also reasonably simple to extend with more functionality.
For example, you could imagine having an implementation that can rebalance
itself, when operations start taking too long. Or adding traversal, or
random-access unicode support that is still backed by compact UTF-8. And I
found it reasonably simple (though it required some usage of unsafe) to write
an implementation of `Base` that used an `mmap`ed file (thus you'd only need to
keep the actual edits in RAM, the rest would be read directly from disk with
the OS managing caching for you).

## Closing remarks

None of these ideas are revolutionary (especially to functional programmers).
But I find that considering if a datastructure I need can be implemented as a
persistent/immutable one helps me to come up with clear abstractions that work
well. And I also believe that Go's interfaces provide a good way to express
these abstractions - because they allow you to start with a simple, immutable
implementation and then compose it with mutable ones - if and only if there are
clear efficiency benefits. Lastly, I think there is an interesting idea here of
how to substitute sum-types by interfaces - not in a direct manner, but instead
by thinking about the common behavior you want to provide over the sum.

I hope you find that this inspires you to think differently about these problems too.
