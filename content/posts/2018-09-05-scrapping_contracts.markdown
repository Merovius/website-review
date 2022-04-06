---
date: "2018-09-05T04:00:00Z"
tags:
- golang
- programming
aliases:
- "/2018/09/05/scrapping_contracts.html"
feedID: "https://blog.merovius.de//2018/09/05/scrapping_contracts.html"
title: Scrapping contracts
summary: I describe a way to simplify the generics design. The ideas are not particularly
  novel and have been expressed to various degrees by other people as well. I hope
  to provide a more complete view of the design though.
---

**tl;dr: I describe a way to simplify the generics design. The ideas are not
particularly novel and have been expressed to various degrees by other people
as well. I hope to provide a more complete view of the design though.**

Recently a [Problem Overview](https://go.googlesource.com/proposal/+/master/design/go2draft-generics-overview.md)
and [Draft Design](https://go.googlesource.com/proposal/+/master/design/go2draft-contracts.md)
for generics in Go have dropped. Since then, predictably,
there has been a bunch of chatter on the intertubez about it. This is a
summary of my thoughts, so far, on the subject - after a bunch of discussions
on Twitter and Reddit.

Note: The design is called "Contracts", but I will refer to it as "the design
doc" here.  When I say "contracts", I will refer to the specific part of the
design to express constraints.

## Contracts vs. Interfaces

First, there is a common observation of overlap between generics and interfaces.
To untangle that, we can say that when we use "generics", what we mean is
*constrained parametric polymorphism*. Go already allows polymorphism by using
interfaces. This desgn doc adds two things: One, a way to add *type-parameters* to
functions and types. And two, a syntax to constrain those type-parameters to a
subset that allows specific operations, via *contracts*.

The latter is where the overlap lies: Interfaces *already* allow you to
constrain arguments to types that allow certain operations. In a way, what
contracts add to this, is that those operations can not only be method calls,
but also allow (and constrain) builtin operators and functions to be used and
to allow or disallow certain composite types (though that mainly affects `map`).

Contracts allow that by the way they are specified: You write a function body
(including arguments, whose notational type becomes the type-variable of the
contract) containing all the statements/expressions you wish to be able
to do. When instantiating a generic type/function with a given set of
type-arguments, the compiler will try to substitute the corresponding
type-variable in the contract body and allow the instantiation, if that body
type-checks.

## The cost of contracts

After talking a bit through some examples, I feel that contracts optimize for
the wrong thing. The analogy I came up with is vocabulary vs. grammar.

The contracts design is appealing to a good degree, because it uses familiar
*syntax*: You don't have to learn any new syntax or language to express your
contract. Just write natural Go code and have that express your constraints for
you. I call this the "grammar" of constraints: The structure that you use to
input them.

On the other hand, for the *user* of Go, the relevant question is what
constraints are possible to express and how to express them. They might be
interested in deduplicating values in their algorithm, which requires
equality-operations. Or they might want to do comparisons (e.g. `Max`), which
requires `>`. I call this the *vocabulary*: What is the correct way to express
the set of constraints that my algorithm needs?

The issue now, is that while the grammar of constraints might be obvious, it is
not always clear what the actual semantic constraints that generates *are*. A
simple example is map-keys. The design doc uses the contract

```
contract comparable (t T) {
   t == t
}
```

to specify types that are valid map-keyes. But to a beginner, it is not
immediately obvious, what comparisons have to do with maps. An alternative
would be

```
contract mapkey (t T) {
  var _ map[t]bool
}
```

But which is better? Similarly, these two contracts

```
contract mult (t T) {
  t = t * t
}

contract add (t T) {
  t = t + t
}
```

seem very similar, but they are, in theory at least, fundamentally different.
Not only because `add` allows `string`, while `mult` doesn't. But also, because
*technically* any type that supports `*` also supports `-` and `/`. And then there's

```
contract div (t T) {
  t = t % t
}
```

which creates another completely different set of types and allowed operators.

A third example is

```
contract stringlike (t T) {
  append([]byte(nil), t...)
}
```

This allows any type with underlying type `string` or `[]byte`, but nothing
else. And again, technically that would imply allowing index-operations and
`len`. But does the compiler understand that?

Lastly, it's not really clear how `len`, `cap`, `make` or `range` would work.
For example, all these contracts are superficially valid:

```
contract rangeable (t T) {
  for x := range t {
    fmt.Println(x)
  }
}

contract lengthed (t T) {
  var _ int = len(t)
}

contract capped (t T) {
  var _ int = cap(t)
}

contract makeable (t T) {
  t = make(T)
}

contract makeable2 (t T) {
  t = make(T, 0)
}
```

But in all these cases, they allow some subset of channel, map, slice and array
types, with vastly different interpretations of these operations, depending on
the kind of type used - to the degree, that code using them would usually be
nonsensical. Disallowing these, however, opens questions about the claim of
familiar Go syntax, as we now have to make decisions what sort of expressions
and statements we do or don't allow in a contract.

This is why I say contracts optimize for grammar, instead of vocabulary. The
programmer is interested in the vocabulary - what does the contract actually
*mean* and what contract should they use? But the vocabulary is obscured by the
grammar - because we use Go syntax, to understand a given contract we need to
know a bunch of things about what the compiler is and is not able to infer from
it.

This is why I don't really buy the argument of not wanting to learn a bunch of
new syntax or new identifiers for constraints: You *still* have to learn that
vocabulary, but you express it in an obscure and unnatural grammar. I hope to
show that we can introduce the power of generics while also using familiar
grammar and with minimal addition of vocabulary.

## Scrapping contracts

Now, I'm not the first person to suggest this, but I think we should consider
scrapping contracts from the design. We can still retain type-parameters and we
can still have constraints, but we express them via interfaces instead. I
should point out, that - for now - I'm intentionally optimizing for simplicity
of the design, at the cost of some boilerplate and some loss of power. I will
later try and provide some alternatives to compensate for that in part. But
there is still likely going to remain a net cost in expressiveness. Personally,
I think that tradeoff is worth exploring.

The new design would retain type-parameters and most of their syntax. The
difference is that type-parameters are a full argument list. The type of an
argument has to be an interface type. It can be ellided, in which case it
defaults to the type of the following type-parameter. The last type-parameter
defaults to `interface{}`. As a bonus, this allows providing multiple sets of
constraints on one declaration:

```
func Map(type A, B) (s []A, f func(A) B) []B {
  var out []B
  for _, a := range s {
    out = f(a)
  }
  return out
}

func Stringify(type A fmt.Stringer) (s []A) []string {
  // Because of the signature of fmt.Stringer.String, we can infer all the
  // type-arguments here. Note that A does not *have* to be passed boxed in an
  // interface. A.String is still a valid method-expression for any fmt.Stringer.
  return Map(s, A.String)
}
```

We still want to be able to express multiple, interdependent parameters, which
we can, via parametric interfaces:

```
type Graph(type Node, Edge) interface {
  Nodes(Edge) []Node
  Edges(Node) []Edge
}

func ShortestPath(type Node, Edge) (g Graph(Node, Edge), from, to Node) []Edge {
  // …
}

// Undirected Graph as an adjacency list. This could be further parameterized,
// to allow for user-defined paylooads.
type AdjacencyList [][]int

func (g AdjacencyList) Nodes(edge [2]int) []int {
  return edge[:]
}

func (g AdjacencyList) Edges(node int) [][2]int {
  var out [][2]int
  for _, v := range g[node] {
    out = append(out, [2]int{node, v}
    if v != node {
      out = append(out, [2]int{v, node})
    }
  }
  return out
}

func main() {
  g := AdjacencyList{…}
  // Types could be infered here, as the names of methods are unique, so we can
  // look at the methods Nodes and Edges of AdjacencyList to infer the
  // type-arguments.
  path := ShortestPath(g, 0, len(g)-1)
  fmt.Println(path)
}
```

The last example is relevant to the difference in power between contracts and
interfaces: Usage of operators. We can still express the concept, but this is
where the increased boilerplate comes in:

```
func Max(type T)(a, b T, less func(T, T) bool) T {
  if less(a, b) {
    return b
  }
  return a
}

func main() {
  fmt.Println(Max(a, b int, func(a, b int) { return a < b }))
}
```

I will try to show some ways to get rid of that boilerplate later. For now,
let's just treat it as a necessary evil of this idea. Though it should be
mentioned, that while this is more *cumbersome*, it's still just as *typesafe*
as contracts (as opposed to, say, a reflect-based generic `Max`).

So, scrapping contracts leaves us with more boilerplate, but just the same set
of concepts we can express - though we do have to pass in any builtin
operations we want to perform as extra functions (or express them in an
interface). In exchange, we get

* Only one way to specify constraints.
* A simpler spec (we don't need to add a new concept, contracts, to the
  language) and a saved (pseudo-)keyword.
* A simpler compiler: We don't need to add a solver to deduce constraints from
  a given contract. The constraint-checker already exists.
* Still a well-known, though less powerfull, language to express constraints,
  with interfaces.
* Simple syntax (same as normal arglists) for having multiple sets of
  constraints in one declaration.
* Trivially good error messages. Types passed in need only be checked for
  consistency and interface satisfaction - the latter is already implemented,
  including good error messages.

## Getting rid of boilerplate

I see two main ways to get rid of boilerplate: Adding methods to builtin types,
or what I call pseudo-interfaces.

### Methods on builtin types

An obvious idea is to not use operators in generic code, but instead use
method-call syntax. That is, we'd do something akin to

```
func Max(type T Ordered) (a, b T) T {
  if a.Less(b) {
    return b
  }
  return a
}
```

To actually reduce the boilerplate, we'd predefine methods for all the
operators on the builtin types. That would allow us to call `Max` with `int`,
for example.

Unfortunately, I can see a bunch of roadblocks to make this work. Methods are
not promoted to derived types, so you couldn't use `Max` with e.g.
`time.Duration`, which has *underlying* type `int64`, but is not the same type.
We'd probably want those methods to be "special" in that they automatically get
promoted to any type whose underlying type is predeclared. That introduces
compatibility issues of clashing Method/Field names.

At the end, to express that `Less` has to take the same argument as the
receiver type, `Ordered` might look something like this:

```go
type Ordered(T) interface {
  Less(T) bool
}

func Max(type T Ordered(T)) (a, b T) T {
  if a.Less(b) {
    return b
  }
  return a
}

// In the universe block:

// Implements Ordered(int).
func (a int) Less(b int) bool {
  retun a < b
}
```

Though it's not clear, whether a parameter like `T Ordered(T)` should be
allowed. And this would technically allow to implement `Ordered(int)` on a
custom type. While that probably won't be very useful (the majority of usecases
will require `T Ordered(T)`), it's not excluded.

### Pseudo-interfaces

Unfortunately I didn't have a lot of time the last couple of days, so I got
beat to the punch on this. Matt Sherman [described the idea first](https://clipperhouse.com/go-generics-typeclasses/)
and called the concept "typeclasses". I will stick with pseudo-interface,
because it fits better in the general concept of this description.

The idea is to introduce a set of types into the language that can be used like
interfaces (including embedding), but instead of providing methods, provide
operators. There is a limited set of base types that need to be provided:

```
pseudo-interface | Allowed operators
-----------------+-------------------
comparable       | ==, !=
ordered          | <, <= > >=
boolean          | ||, &&, !
bitwise          | ^, %, &, &^, <<, >>
arith            | +, -, *, /
concat           | +
complex          | real(z), imag(z)
nilable          | v == nil
```

and a set of derived pseudo-interfaces:

```
pseudo-interface | definition
-----------------+-----------------------------------------------------
num              | interface { comparable; ordered; arith }
integral         | interface { num; bitwise }
stringy          | interface { comparable; ordered; concat; len() int }
iface            | interface { comparable; nilable }
```

The pseudo-interfaces would be declared in the universe block, as predeclared
identifiers. This makes them backwards-compatible (as opposed to methods on
builtin types), because any existing identifier would just shadow these (akin
to how you can have a variable with name `string`).

Bitshift-operators currently are restricted when used with constants
overflowing the width of an integral type. For generic code, this restriction
would be lifted (as the size is not statically known) and instead the behavior
is equivalent to if the right operand is an uint variable with the given
value.

This would allow us to write

```
func Max(type T ordered) (a, b T) T {
  if a < b {
    return b
  }
  return a
}
```

Notably, the list of pseudo-interfaces doesn't include anything related to
channel-, slice- or map-operations (or other composite types). The idea is to
instead use a type literal directly:

```
type Keys(type K, V) (m map[K]V) []K {
  var out []K
  for k := range m {
    out = append(out, k)
  }
  return out
}
```

As every type supporting, e.g. `map` operations, need to have underlying type
`map[K]V`, it's thus assignable to that type and can be passed to `Keys` as is.
That is, this is completely legal:

```
func main() {
  type MyMap map[string]int
  var m = MyMap{
    "foo": 23,
    "bar": 42,
  }
  fmt.Println(Keys(m))
}
```

This also solves another problem with contracts: The ambiguity of `len`, `cap`
and `range`. As the actual kind of the value is not only known during
compilation of the generic function, but even obvious from the code, there is
no question about the intended semantics.

Should Go ever grow operator overloading via operator methods, the
pseudo-interfaces could be changed into actual interfaces, containing the
necessary methods. Of course, that implies that operator overloading would
retain the properties of existing operators, e.g. that having `==` implies
having `!=`, or having `-` implying having `+`. Personally, I consider that a
good thing - it limits the abuse of operator overloading for nonsensical
operations (say, `<<` for writing to an `io.Writer`).

I'm not trying to advocate for operator overloading, but think it's worth
mentioning that this design leaves the door open to that.

### But performance

A possible criticism of either of these approaches is, that operators have
better performance than dynamic dispatch to a method. I believe (vigorous
handwaving ahead) that this is no different in the existing contracts proposal.
If generic code is compiled generically, it still needs to employ some means
of dynamic dispatch for operators. If, on the other hand, it's compiled
instantiated, then the compiler would also be able to devirtualize the
interfaces - and then inline the method definition.

## Conclusion

I've previously said that I'm "meh" on the design doc, which is the strongest
form of endorsement a generics proposal could ever get from me. After some
discussion, I'm more and more convinced that while contracts *seem*
conceptually simple, they create a plethora of implementation- and usage
questions. I'm not sure, the supposed advantage of contracts, of a well-known
syntax, holds up to scrutiny when it comes to mapping that to the actually
derived constraints or writing contracts. There are also many open questions in
regards to contracts, a bunch of them related to the ambiguity of
Go-expressions. As a result, I'm starting to feel more negative towards them -
they *look* like an elegant idea, but in practice, they have a lot of weird
corners.

This design is similar (AIUI) to the [type functions](https://go.googlesource.com/proposal/+/master/design/15292/2010-06-type-functions.md)
proposal, so I assume there are good reasons the Go team does not want this.
The difference is mainly the absence of operator methods in favor of
pseudo-interfaces or explicit method calls. This design also handwaves a
couple of important implementation questions - the justification for that is
that these questions (e.g. type inference and code generation) should be able
to be taken from the design doc with minimal changes. It's entirely
possible that I am overlooking something, though.
