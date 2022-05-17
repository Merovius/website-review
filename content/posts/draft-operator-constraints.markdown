---
date: "2022-05-16 17:11:00"
title: "Operator constraints in Go"
---

Let's say you want to implement a sorting function in Go. Or perhaps a data
structure like a
[binary search tree](https://en.wikipedia.org/wiki/Binary_search_tree),
providing ordered access to its elements. Because you want your code to be
re-usable and type safe, you want to use type parameters. So you need a way to
order user-provided types.

There are multiple methods of doing that, with different trade-offs. Let's talk
about four in particular here:

1. `constraints.Ordered`
2. A method constraint
3. Taking a comparison function
4. Comparator types

I believe the fourth is a novel idea. At least I don't think I have seen it
come up in a discussion about Go so far. Though I think I have seen it used in
other languages.

## `constraints.Ordered`

Go 1.18 has a mechanism to constraint a type parameter to all types which have
the `<` operator defined on them. The types which have this operator are
exactly all types whose underlying type is `string` or one of the predeclared
integer and float types. So we can write a type set expressing that:

```go
type Integer interface {
  ~int | ~int8 | ~int16 | ~int32 | ~int64 | ~uint | ~uint8 | ~uint16 | ~uint32 | ~uint64 | ~uintptr
}

type Float interface {
  ~float32 | ~float64
}

type Ordered interface {
  Integer | Float | ~string
}
```

Because that's a fairly common thing to want to do,
[there is already a package which contains these kinds of type sets](https://pkg.go.dev/golang.org/x/exp/constraints#Ordered).

With this, you can write the signature of your sorting function or the
definition of your search tree as:

```go
func Sort[T constraints.Ordered](s []T) {
  // …
}

type SearchTree[T constraints.Ordered] struct {
  // …
}
```

The main advantage of this is that it works directly with predeclared types and
simple types like `time.Duration`. It also is very clear.

The main disadvantage is that it does not allow composite types like `struct`s.
And what if a user want a different sorting order than the one implied by `<`?
For example, if they want to reverse the order or want specialized string
collation. For example, a multimedia library might want to sort “The Expanse”
under E. Or some letters sort differently depending on the language setting.

`constraints.Ordered` is simple, but it also is inflexible.

## Method constraints

To allow more flexibility, we can use method constraints. This allows a user to
implement whatever sorting order they want as a method on their type.

We can write that constraint like this:

```go
type Lesser[T any] interface {
  // Less returns if the receiver is less than v.
  Less(v T) bool
}
```

The type parameter is necessary, because we have to refer to the type itself in
the `Less` method. This is hopefully clearer when we look at how this is used:

```go
func Sort[T Lesser[T]](s []T) {
  // …
}

func SearchTree[T Lesser[T]](s []T) {
  // …
}
```

This allows the user of our library to customize the sorting order by defining
a new type with a `Less` method:

```go
type ReverseInt int

func (i ReverseInt) Less(j ReverseInt) bool {
  return j < i // order is reversed
}
```

You can also make this easier by providing some helper types. E.g.

```go
// Reversed[T] is T in reverse sorting order.
type Reversed[T Lesser[T]] T

func Less(a Reversed[T]) (b Reversed[T]) bool {
  return b < a
}
```

Which can then be used as

```go
type MyInt int

func (i MyInt) Less(j MyInt) bool {
  return i < j
}

func main() {
  a := []MyInt{42,23,1337}
  Sort[Reversed[MyInt]](a)
}
```

The disadvantage of this is that it requires some boiler plate on part of your
user. Using a custom sorting order always requires defining a type with a method.

They can't use your code with predeclared types like `int` or `string`, but
always have to wrap it into a new type.

Likewise, if a type already has a natural comparison method, but it is not
called `Less` (for example, `time.Before`) there needs to be a wrapper to
rename the method.

Whenever one of these wrappings happens (the same applies to `Reversed[T]`),
your user might have to convert back and forth when passing data to or from
your code.

It also is a little bit more confusing than `constraints.Ordered`, because your
user has to understand the purpose of the extra type parameter on `Lesser`.

## Passing a comparison function

A simple way to get flexibility is to have the user pass us a function used for
comparison directly:

```go
func Sort[T any](s []T, less func(T, T) bool) {
  // …
}

type SearchTree[T any] struct {
  Less func(T, T) bool
  // …
}

func NewSearchTree(less func(T, T) bool) *SearchTree[T] {
  // …
  return &SearchTree[T]{
    Less: less,
    // …
  }
}
```

This essentially abandons the idea of type constraints altogether. Our code
works with *any* type and we directly pass around the custom behavior as
`func`s. Type parameters are only used to ensure that the arguments to those
`func`s are compatible.

The advantage of this is maximum flexibility. Any type which already has a
`Less` method like above can simply be used with this directly, by using
[method expressions](https://go.dev/ref/spec#Method_expressions). Regardless of
how the method is actually named:

```go
func main() {
  a := []time.Time{ /* … */ }
  Sort(a, time.Time.Before)
}
```

There is also no boilerplate needed to customize sorting behavior:

```go
func main() {
  a := []int{42,23,1337}
  Sort(a, func(i, j int) bool {
    return j < i // reversed order
  })
}
```

And again, you can provide helpers for common customizations:

```go
func Reversed[T any](less func(T, T) bool) (greater func(T, T) bool) {
  return func(a, b T) bool { return less(b, a) }
}
```

This approach is arguably also more correct than the one above because it
decouples the type from the comparison used. If I use a `SearchTree` as a set
datatype, there is no real reason why the elements in the set would be specific
to the comparison used. It should be “a set of `string`” not “a set of
`MyCustomlyOrderedString`”. This reflects the fact that with the method
constraint, we have to convert back-and-forth when putting things into the
container or taking it out again.

The main *disadvantage* of this approach is that it means you can not have
useful zero values. Your `SearchTree` type needs the `Less` field to be
populated to work.

You can not even lazily initialize it (which is a common trick to make types
which need initialization have a useful zero value), because *you don't know
what it should be*.

## Comparator types

There is a way to pass a function “statically”. That is, instead of passing
around a `func` value, we can pass it as a type argument. The way to do that is
to attach it as a method to a `struct{}` type:

```go
import "golang.org/x/exp/slices"

type IntComparator struct{}

func (IntComparator) Less(a, b int) bool {
  return a < b
}

func main() {
  a := []int{42,23,1337}
  less := IntComparator{}.Less // has type func(int, int) bool
  slices.SortFunc(a, less)
}
```

Based on this, we can devise a mechanism to allow custom comparisons:

```go
// Comparator is a helper type used to compare two T values.
type Comparator[T any] interface {
  ~struct{}
  Less(a, b T) bool
}

func Sort[T any, C Comparator[T]](a []T) {
  var c C
  less := c.Less // has type func(T, T) bool
  // …
}

type SearchTree[T any, C Comparator[T]] struct {
  // …
}
```

The `~struct{}` constraints any implementation of `Comparator[T]` to have
underlying type `struct{}`. It is not strictly necessary, but it serves two
purposes here:

1. It makes clear that `Comparator[T]` itself is not supposed to carry any
   state. It only exists to have its method called.
2. It ensures (as much as possible) that the zero value of `C` is safe to use.
   In particular, it prevents `Comparator[T]` itself to instantiate a type
   parameter constrained on it, which would have a `nil` zero value which would
   panic when `c.Less` is called.

You can again provide helpers, for example to combine this approach with the
above ones:

```go
type LessOperator[T constraints.Ordered] struct{}

func (LessOperator[T]) Less(a, b T) bool {
  return a < b
}

type LessMethod[T Lesser[T]] struct{}

func (LessMethod[T]) Less(a, b T) bool {
  return a.Less(b)
}

type Reversed[T any, C Comparator[T]] struct{}

func (Reversed[T, C]) Less(a, b T) bool {
  var c C
  return c.Less(b, a)
}
```

The advantage of this approach is that it makes the zero value of
`SearchTree[T, C]` useful. For example, a `SearchTree[int, LessOperator[int]]`
can be used directly, without any initialization.

It also carries over the advantage of decoupling the comparison from the
element type, which we got from taking comparison functions.

One disadvantage is that the comparator can never be inferred. It always has to
be specified in the instantiation explicitly (though we could infer the element
type from the comparator). That's similar to how we always had to pass a `less`
function explicitly above.

Another disadvantage is that this *always* requires defining a type for
comparisons. Where with the comparison function we could define customizations
(like reversing the order) inline with a `func` literal, this mechanism always
requires a method.

Lastly, this is arguably too clever for its own good. Understanding the purpose
and idea behind the `Comparator` type is likely to trip up your users when
reading the documentation.

## Summary

We are left with these trade-offs:

|| `constraints.Ordered` | `Lesser[T]` | `func(T,T) bool` | `Comparator[T]` |
|--------------------------|----|--------|----|----|
| Predeclared types        | 👍 | 👎     | 👎 | 👎 |
| Composite types          | 👎 | 👍     | 👍 | 👍 |
| Custom order             | 👎 | 👍     | 👍 | 👍 |
| Type boilerplate         | 👍 | 👎     | 👍 | 👎 |
| Useful zero value        | 👍 | 👍     | 👎 | 👍 |
| Type inference           | 👍 | 👍     | 👍 | 👎 |
| Coupled Type/Order       | 👎 | 👎     | 👍 | 👍 |
| Clarity                  | 👍 | 🤷[^1] | 👍 | 👎 |

[^1]: It's a *little* bit worse, but probably fine.

One thing standing out in this table is that there is no way to *both* support
predeclared types *and* support user defined types.

It would be great if there was a way to support multiple of these mechanisms
using the same code. That is, it would be great if we could write something
like

```go
// Ordered is a constraint to allow a type to be sorted.
// If a Less method is present, it has precedent.
type Ordered[T any] interface {
  constraints.Ordered | Lesser[T]
}
```

Unfortunately, allowing this
[is harder than one might think](https://blog.merovius.de/posts/2022-05-16-calculating-type-sets/).

Until then, you might want to provide multiple APIs to allow your users more
flexibility. The standard library currently seems to be converging on providing
a `constraints.Ordered` version and a comparison function version.  The latter
gets a `Func` suffix to the name. See
[the experimental `slices` package](https://pkg.go.dev/golang.org/x/exp/slices)
for an example.
