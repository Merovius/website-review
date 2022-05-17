---
date: "2018-01-08T00:30:00Z"
tags:
- haskell
- math
- programming
aliases:
- "/2018/01/08/monads-are-just-monoids.html"
omitFeed: true
title: Monads are just monoids in the category of endofunctors
summary: I explain the mathematical background of a joke-explanation of monads. Contains
  lots of math and a hasty introduction to category theory
---

**tl;dr: I explain the mathematical background of a joke-explanation of monads. Contains lots of math and a hasty introduction to category theory.**

There is a running gag in the programming community, that newcomers will often
be confused by the concept of monads (which is how sequential computations are
modeled in purely functional languages) and getting the explanation "it is
simple, really: Monads are just monoids in the category of endofunctors". This
is not meant as an actual explanation, but rather to poke a bit of fun at the
habit of functional programmers to give quite abstract and theoretical
explanations at times, that are not all that helpful.

However, given my background in mathematics, I decided that I wanted to
actually approach Haskell from this point of view: I am interested in how it
uses math to model programming and also to, after several years of doing mostly
engineering focused programming work, flex my math muscles again - as there is
quite a bit of interesting math behind these concepts.

The quote is from a pretty popular [book about category
theory](http://www.maths.ed.ac.uk/~aar/papers/maclanecat.pdf) and is, in full:

> All told, a monad in \\(X\\) is just a monoid in the category of endofunctors
> of \\(X\\), with product \\(\times\\) replaced by composition of endofunctors
> and unit set by the identity endofunctor.

This, of course, is an explanation of the *mathematical* concept of monads,
not meant for programmers. Most explanations of the quote that I found either
assumed quite a bit of knowledge in Haskell or took a lot of liberties with the
mathematical concepts (and relied a lot on "squinting") or both. This write up
is my attempt, to walk through all the concepts needed to explain monads as a
mathematical concept and how it relates to Haskell - with as little squinting
as possible.

Of course, there are a couple of disclaimers, I should start with:

1. This is not the best way to understand what monads are, if you are actually
   interested in using them to program. In fact, it is literally the worst way.
   I would recommend [this intro](http://homepages.inf.ed.ac.uk/wadler/papers/marktoberdorf/baastad.pdf),
   which takes a much more practical approach.
2. This is not the best way to understand how category theory works, if you are
   actually interested in learning mathematics. In fact, it is literally the
   worst way. I would recommend [the book the quote is from](http://www.maths.ed.ac.uk/~aar/papers/maclanecat.pdf),
   it's quite good (but assumes a math audience).
3. I haven't done mathematics in years. I also don't know much Haskell either.
   So I might be getting a bunch of stuff wrong to varying degrees. I'm sure I
   will hear all about it :)
4. Even if I would *understand* everything correctly, there are still a lot of
   details, mostly of technical nature, I had to omit, to keep this "short".
   Not that it is short.

Originally, I intended this to be the ultimate explanation, which would teach
Haskellers category theory, mathematicians Haskell and people who know neither
both. Unsurprisingly, this is not what this is, at all. It ended up mostly a
write up to assure myself that I understood the path myself. If anything, you
can treat this as a kind of "reading companion": If you want to understand this
topic of the intersection between category theory and functional programming,
this post can lead you through the correct terms to search for and give you a
good idea what to focus on, in the respective Wikipedia articles.

With all that out of the way, let's begin.

## Categories

In mathematics, a category is (roughly) a collection of objects and a
collection of arrows between them. There is not a lot of meaning behind these,
but it will probably help you to think of objects as sets and arrows as
mappings.  Every arrow goes from an object (the *domain*) to an object (the
*codomain*) and we write an arrow as \\(f:X\to Y\\), where \\(f\\) is the name
of the arrow, \\(X\\) is the domain and \\(Y\\) is the codomain. Just like with
mappings, there can be many arrows between any given pair of objects - or there
may be none.

We do need *some* restrictions: First, we require a specific *identity* arrow
\\(\mathrm{id}:X\to X\\) attached to every object \\(X\\), which has \\(X\\) as
both domain and codomain. Secondly, we require (some) arrows to be
*composable*. That is if we have two arrows \\(f:X\to Y,g:Y\to Z\\) - so,
whenever the domain of \\(g\\) is the codomain of \\(f\\) - there should also
be a composed arrow[^1]
\\(g\circ f: X\to Z\\), that shares the domain with \\(f\\) and the codomain with
\\(g\\).

Furthermore, the identity arrows must act as a *unit* for composition, that is,
for every arrow \\(f\\) we require \\(\mathrm{id}\circ f = f = f
\circ\mathrm{id}\\). We also require composition to be *associative*, that is
\\((f\circ g)\circ h = f\circ(g\circ h)\\) (whenever all compositions exist)[^2].

When we talk about a category, we often draw diagrams like this:

<div>
\[
\require{AMScd}
\begin{CD}
X       @>{f}>> Y       \\
@V{g}VV         @VV{p}V \\
Z       @>>{q}> W       \\
\end{CD}
\]
</div>

They show some of the objects and arrows from the category in a compact way.
This particular diagram indicates that there are four objects and four arrows
involved, with obvious domains and codomains. We only draw a subset of the
objects and arrows, that is interesting for the point we are trying to make -
for example, above diagram could also contain, of course, identity arrows and
compositions \\(p\circ f\\) and \\(q\circ g\\)), but we didn't draw them. In
a square like this, we can take two paths from \\(X\\) to \\(W\\). If these
paths are identical (that is, \\(p\circ f = q\circ g\\), we say that the
square *commutes*. A *commutative* diagram is a diagram, in which any square
commutes, that is, it does not matter which path we take from any object to
another. Most of the time, when we draw a diagram, we intend it to be
commutative.

So, to summarize, to define a mathematical category, we need to:

1. Specify what our objects are
2. Specify what our arrows are, where each arrow starts and ends at a certain
   object
3. This collection of arrows need to include an arrow \\(\mathrm{id}\_X\\) for
   every object \\(X\\), which starts and ends at \\(X\\)
4. And we need to be able to glue together arrows \\(f:X\to Y\\) and \\(g:Y\to
   Z\\) to an arrow \\(g\circ f: X\to Z\\)

In Haskell, we work on the category **Hask**, which consists of:

1. The objects are *types*: `Int` is an object, `String` is an object but also
   `Int | String`, `String -> Int` and any other complicated type you can think
   of.
2. The arrows are *functions*: `f :: a -> b` is a function taking an `a` as an
   input and returning a `b` and is represented by an arrow `f`, which has `a`
   as its domain and `b` as its codomain.  So, for example, `length :: String
   -> Int` would start at the type `String` and end at `Int`.
3. Haskell has a function `id :: a -> a` which gives us the identity arrow
   for any type `a`.
4. We can compose functions with the operator `(.) :: (b -> c) -> (a -> b) ->
   (a -> c)`. Note, that this follows the swapped notation of \\(\circ\\), where
   the input type of the left function is the output type of the right function.

In general, category theory is concerned with the *relationship between*
categories, whereas in functional programming, we usually only deal with this
one category. This turns out to be both a blessing and a curse: It means that
our object of study is much simpler, but it also means, that it is sometimes
hard to see how to apply the general concepts to the limited environment of
functional programming.

## Monoids

Understanding categories puts us in the position to understand *monoids*. A
monoid is the generalized structure underlying concepts like the natural
numbers: We can *add* two natural numbers, but we can't (in general) *subtract*
them, as there are no negative numbers. We also have the number \\(0\\), which,
when added to any number, does nothing - it acts as a *unit* for addition. And
we also observe, that addition is *associative*, that is, when doing a bunch of
additions, the order we do them in doesn't matter.

The same properties also apply to other constructs. For example, if we take all
maps from a given set to itself, they can be composed and that composition is
associative and there is a unit element (the identity map).

This provides us with the following elements to define a monoid:

1. A set \\(M\\)
2. An operation \\(\star\colon M\times M\to M\\), which "adds" together two elements to
   make a new one
3. We need a special unit element \\(u\in M\\), which acts neutrally when added to
   any other element, that is \\(m\star u=m=u\star m\\)
4. The operation needs to be associative, that is we always require
   \\(m\star(n\star k)=(m\star n)\star k\\)

There is another way to frame this, which is closer in line with category theory.
If we take \\(1 := \\{0\\}\\) to be a 1-element set, we can see that the
elements of \\(M\\) are in a one-to-one correspondence to functions \\(1\to M\\):
Every such function chooses an element of \\(M\\) (the image of \\(0\\)) and
every element \\(m\in M\\) fixes such a function, by using \\(f(0) := m\\).
Thus, instead of saying "we need a special element of \\(M\\)", we can also
choose a special *function* \\(\eta: 1\to M\\). And instead of talking about an
"operation", we can talk about a function \\(\mu: M\times M\to M\\). Which
means, we can define a monoid via a commutative diagram like so:

<div>
\[
\begin{CD}
1 \\
@V{\eta}VV \\
M \\
\end{CD}
\hspace{1em}
\begin{CD}
M\times M \\
@V{\mu}VV \\
M \\
\end{CD}
\hspace{1em}
\begin{CD}
M\times 1 @>{\mathrm{id}\times\eta}>> M\times M @<{\eta\times\mathrm{id}}<< 1\times M \\
@V{\pi_1}VV                           @V{\mu}VV                             @V{\pi_2}VV \\
M         @>{\mathrm{id}}>>           M         @<{\mathrm{id}}<<           M \\
\end{CD}
\hspace{1em}
\begin{CD}
M\times M\times M @>{\mu\times\mathrm{id}}>> M\times M \\
@V{\mathrm{id}\times\mu}VV                   @V{\mu}VV \\
M\times M         @>{\mu}>>                  M \\
\end{CD}
\]
</div>

\\(\pi\_1\\) and \\(\pi\_2\\) here, are the functions that project to the first
or second component of a cross product respectively (that is \\(\pi\_1(a, b) :=
a, \pi\_2(a, b) := b\\)) and e.g. \\(\mathrm{id}\times\eta\\) is the map that
applies \\(\mathrm{id}\\) to the first component of a cross-product and
\\(\eta\\) to the second: \\(\mathrm{id}\times\eta(m, 0) = (m, \eta(0))\\).

There are four sub-diagrams here:

1. The first diagram just says, that we need an arrow \\(\eta:1\to M\\). This
   chooses a unit element for us.
2. Likewise, the second diagram just says, that we need an arrow
   \\(\mu:M\times M\to M\\). This is the operation.
3. The third diagram tells us that the chosen by \\(\eta\\) should be a unit
   for \\(\mu\\). The commutativity of the left square tells us, that it should
   be right-neutral, that is
   \\[ \forall m\in M: m = \pi\_1(m, 0) = \mu(\mathrm{id}\times\eta(m, 0)) = \mu(m, \eta(0)) \\]
   and the commutativity of the right square tells us, that it should be left-neutral, that is
   \\[ \forall m\in M: m = \pi\_2(0,m) = \mu(\eta\times\mathrm{id}(0, m)) = \mu(\eta(0), m) \\]

Thus, the first diagram is saying that the element chosen by \\(\eta\\) should
act like a unit. For example, the left square says

\\[\pi\_1(m,0) = \mu((\mathrm{id}\times\eta)(m,0)) = \mu(m,\eta(0))\\]

Now, writing \\(\mu(m,n) = m\star n\\) and \\(\eta(0) = u\\), this is equivalent to saying \\(m = u\star m\\).

The second diagram is saying that \\(\mu\\) should be associative: The top arrow
combines the first two elements, the left arrow combines the second two. The right and
bottom arrows then combine the result with the remaining element respectively,
so commutativity of that square means the familiar \\(m\star (n\star k) = (m\star n)\star k\\).

Haskell has the concept of a monoid too. While it's not really relevant to the
discussion, it might be enlightening to see, how it's modeled. A monoid in
Haskell is a type-class with two (required) methods:

```haskell
class Monoid a where
  mempty :: a
  mappend :: a -> a -> a
```

Now, this gives us the operation (`mappend`) and the unit (`a`), but where are
the requirements of associativity and the unit acting neutrally? The Haskell
type system is unable to codify these requirements, so they are instead given
as a "law", that is, any implementation of a monoid is supposed to have these
properties, to be manually checked by the programmer:

* `mappend mempty x = x` (the unit is left-neutral)
* `mappend x mempty = x` (the unit is right-neutral)
* `mappend x (mappend y z) = mappend (mappend x y) z` (the operation is associative)

## Functors

I mentioned that category theory investigates the relationship between
categories - but so far, everything we've seen only happens inside a single
category. Functors are, how we relate categories to each other. Given two
categories \\(\mathcal{B}\\) and \\(\mathcal{C}\\), a *functor*
\\(F:\mathcal{B}\to \mathcal{C}\\) assigns to every object \\(X\\) of
\\(\mathcal{B}\\), an object \\(F(X)\\) of \\(\mathcal{C}\\). It also assigns
to every arrow \\(f:X\to Y\\) in \\(\mathcal{B}\\) a corresponding arrow
\\(F(f): F(X)\to F(Y)\\) in \\(\mathcal{C}\\)[^3]. So, a functor transfers
arrows from one category to another, preserving domain and codomain. To
actually preserve the structure, we also need it to preserve the extra requirements
of a category, identities and composition. So we need, in total:

1. An object map, \\(F:O\_\mathcal{B} \to O\_\mathcal{C}\\)
2. An arrow map, \\(F:A\_\mathcal{B}\to A\_\mathcal{C}\\), which preserves
   start and end object, that is the image of an arrow \\(X\to Y\\) starts at
   \\(F(X)\\) and ends at \\(F(Y)\\)
3. The arrow map has to preserve identities, that is \\(F(\mathrm{id}\_X) =
   \mathrm{id}\_{F(X)}\\)
4. The arrow map has to preserve composition, that is \\(F(g\circ f) =
   F(g)\circ F(f)\\).

A trivial example of a functor is the *identity functor* (which we will call
\\(I\\)), which assigns each object to itself and each arrow to itself - that
is, it doesn't change the category at all.

A simple example is the construction of the *free monoid*, which maps from the
category of sets to the category of monoids. The Free monoid \\(S^\*\\) on a
set \\(S\\) is the set of all finite length strings of elements of \\(S\\),
with concatenation as the operation and the empty string as the unit. Our
object map then assigns to each set \\(S\\) its free monoid \\(S^\*\\). And our
arrow map assigns to each function \\(f:S\to T\\) the function \\(f^\*:S^\*\to
T^\*\\), that applies \\(f\\) to each element of the input string.

There is an interesting side note here: Mathematicians love to abstract.
Categories arose from the observation, that in many branches of mathematics we
are researching some class of objects with some associated structure and those
maps between them, that preserve this structure. It turns out that category
theory is a branch of mathematics that is researching the objects of
categories, with some associated structure (identity arrows and composition)
and maps (functors) between them, that preserve that structure. So it seems
obvious that we should be able to view categories *as objects of a category*,
with functors as arrows. Functors can be composed (in the obvious way) and
every category has an identity functor, that just maps every object and arrow
to itself.

Now, in Haskell, Functors are again a type class:

```haskell
class Functor f where
  fmap :: (a -> b) -> (f a -> f b)
```

This looks like our arrow map: It assigns to each function `g :: a -> b` a
function `fmap g :: f a -> f b`. The object map is implicit: When we write `f a`,
we are referring to a new type, that depends on `a` - so we "map" `a` to `f a`
[^4].

Again, there are additional requirements the type system of Haskell can not
capture. So we provide them as laws the programmer has to check manually:

* `fmap id  ==  id` (preserves identities)
* `fmap (f . g)  ==  fmap f . fmap g` (preserves composition)

There is one thing to note here: As mentioned, in Haskell we only really deal
with one category, the category of types. That means that a functor always maps
from the category of types to *itself*. In mathematics, we call such a functor,
that maps a category to itself, an *endofunctor*. So we can tell, that in
Haskell, every functor is automatically an endofunctor.

## Natural transformations

We now understand categories and we understand functors. We also understand,
that we can look at something like the category of categories. But the
definition of a monad given to us talks about the *category of endofunctors*.
So we seem to have to step up yet another level in the abstraction hierarchy
and somehow build this category. As objects, we'd like to have endofunctors -
and arrows will be *natural transformations*, which take one functor to
another, while preserving its internal structure (the mapping of arrows). If
that sounds complicated and abstract, that's because it is.

We need two functors \\(F,G:\mathcal{B}\to \mathcal{C}\\) of the same "kind"
(that is, mapping to and from the same categories). A natural transformation
\\(\eta:F\dot\to G\\) assigns an arrow[^5] \\(\eta\_X: F(X)\to G(X)\\) (called
a *component* of \\(\eta\\)) to every object in \\(\mathcal{B}\\). So a component
\\(\eta\_X\\) describes, how we can translate the action of \\(F\\) on \\(X\\)
into the action of \\(G\\) on \\(X\\) - i.e. how to translate their object maps.
We also have to talk about the translation of the arrow maps. For that, we observe
that for any arrow \\(f:X\to Y\\) in \\(\mathcal{B}\\), we get four new arrows in
\\(\mathcal{C}\\):

<div>
\[
\begin{CD}
X       \\
@V{f}VV \\
Y       \\
\end{CD}
\hspace{1em}
\begin{CD}
F(X)        @>{\eta_X}>> G(X)       \\
@V{F(f)}VV               @VV{G(f)}V \\
F(Y)        @>>{\eta_Y}> G(Y)       \\
\end{CD}
\]
</div>

For a natural transformation, we require the resulting square to commute.

So, to recap: To create a natural transformation, we need

1. Two functors \\(F,G:\mathcal{B}\to\mathcal{C}\\)
2. For every object \\(X\\) in \\(\mathcal{B}\\), an arrow \\(\eta\_X: F(X)\to
   G(X)\\)
3. The components need to be compatible with the arrow maps of the functors:
   \\(\eta\_Y\circ F(f) = G(f)\circ \eta\_X\\).

In Haskell, we can define a natural transformation like so:

```haskell
class (Functor f, Functor g) => Transformation f g where
    eta :: f a -> g a
```

`f` and `g` are functors and a natural transformation from `f` to `g` provides
a map `f a -> g a` for every type `a`. Again, the requirement of compatibility
with the actions of the functors is not expressible as a type signature, but we
can require it as a law:

* `eta (fmap fn a) = fmap fn (eta a)`

## Monads

This, finally, puts us in the position to define monads. Let's look at our quote above:

> All told, a monad in \\(X\\) is just a monoid in the category of endofunctors
> of \\(X\\), with product \\(\times\\) replaced by composition of endofunctors
> and unit set by the identity endofunctor.

It should be clear, how we can *compose* endofunctors. But it is important,
that this is a different view of these things than if we'd look at the category
of categories - there, objects are categories and functors are arrows, while
here, objects are *functors* and arrows are natural transformations. That
shows, how composition of functors can take the role of the cross-product of
sets: In a set-category, the cross product makes a new set out of two other
set. In the category of endofunctors, composition makes a new endofunctor out
of two other endofunctors.

When we defined monoids diagrammatically, we also needed a cross product of
mappings, that is, given a map \\(f:X\_1\to Y\_1\\) and a map \\(g:X\_2\to
Y\_2\\), we needed the map \\(f\times g: X\_1\times X\_2\to Y\_1\times
Y\_2\\), which operated on the individual constituents. If we want to replace
the cross product with composition of endofunctors, we need an equivalent for
natural transformations. That is, given two natural transformations
\\(\eta:F\to G\\) and \\(\epsilon:J\to K\\), we want to construct a natural
transformation \\(\eta\epsilon:J\circ F\to K\circ G\\). This diagram
illustrates how we get there (working on components):

<div>
\[
\begin{CD}
F(X)    @>{\eta_X}>>    G(X)                  @.                            \\
@V{J}VV                 @VV{J}V               @.                            \\
J(F(X)) @>{J(\eta_X)}>> J(G(X))               @>{\epsilon_{G(X)}}>> K(G(X)) \\
\end{CD}
\]
</div>

As we can see, we can build an arrow \\(\epsilon\_{G(X)}\circ J(\eta\_X):
J(F(X)) \to K(G(X))\\), which we can use as the components of our natural
transformation \\(\eta\epsilon:J\circ F\to K\circ G\\). This construction is
called the *horizontal composition* of natural transformations. We should
verify that this is indeed a natural transformation - for now, let's just
accept that it follows from the naturality of \\(\eta\\) and \\(\epsilon\\).

Lastly, there is an obvious natural transformation taking a functor to itself;
each component being just the identity arrow. We call that natural
transformation \\(\iota\\), staying with the convention of using Greek letters
for natural transformations.

With this, we can redraw the diagram we used to define monoids above, the
replacements indicated by the quotes:

<div>
\[
\begin{CD}
I \\
@V{\eta}VV \\
M \\
\end{CD}
\hspace{1em}
\begin{CD}
M\circ M \\
@V{\mu}VV \\
M \\
\end{CD}
\hspace{1em}
\begin{CD}
M\circ I @>{\iota\ \eta}>> M\circ M  @<{\eta\ \iota}<< I\circ M \\
@VVV                       @V{\mu}VV                   @VVV \\
M        @>{\iota}>>       M         @<{\iota}<<       M \\
\end{CD}
\hspace{1em}
\begin{CD}
M\circ M\circ M @>{\mu\ \iota}>> M\circ M  \\
@V{\iota\ \mu}VV                 @V{\mu}VV \\
M\circ M        @>{\mu}>>      M         \\
\end{CD}
\]
</div>

The vertical arrows in the middle diagram now simply apply the composition of
functors, using that the identity functor is a unit.

These diagrams encode these conditions on our natural transformations[^6]:

* \\(\mu\circ\eta\iota = \mu = \iota\eta\circ\mu\\), that is \\(\eta\\) serves as a unit
* \\(\mu\circ\mu\iota = \mu\circ\iota\mu\\), that is \\(\mu\\) is associative

To recap, a monad, in category theory, is

* An endofunctor \\(M\\)
* A natural transformation \\(\eta: I\to M\\), which serves as an identity for
  horizontal composition.
* A natural transformation \\(\mu: M\circ M\to M\\), which is associative in
  respect to horizontal composition.

Now, let's see, how this maps to Haskell monads.

First, what is the identity functor in Haskell? As we pointed out above, the
object function of functors is implicit, when we write `f a` instead of `a`. As
such, the identity functor is simply `a` - i.e. we map any type to itself.
`fmap` of that functor would thus also just be the identity
`fmap :: (a -> a) -> (a -> a)`.

So, what would our natural transformation \\(\eta\\) look like? As we said, a
natural transformation between two functors is just a map `f a -> g a`. So (if
we call our endofunctor `m`) the identity transformation of our monoid is
`eta :: a -> m a`
mapping the identity functor to `m`. We also need our monoidal operation, which
should map `m` applied twice to `m`:
`mu :: m (m a) -> m a`.

Now, Haskellers write `return` instead of `eta` and write `join` instead of
`mu`, giving us the type class[^7]

```haskell
class (Functor m) => Monad where
  return :: a -> m a
  join :: m (m a) -> m a
```

As a last note, it is worth pointing out that you usually won't implement
`join`, but instead a different function, called "monadic bind":

```haskell
(>>=) :: m a -> (a -> m b) -> m b
```

The reason is, that this more closely maps to what monads are actually *used*
for in functional programming. But we can move between `join` and `>>=` via

```haskell
(>>=) :: m a -> (a -> m b) -> m b
v >>= f = join ((fmap f) v)

join :: m (m a) -> m a
join v = v >>= id
```

## Conclusion

This certainly was a bit of a long ride. It took me *much* longer than
anticipated both to understand all the steps necessary and to write them down.
I hope you found it helpful and I hope I didn't make too many, too glaring
mistakes. If so (either), feel free to let me know on
[Twitter](https://twitter.com/TheMerovius),
[reddit](https://www.reddit.com/r/haskell/comments/7oudxd/monads_are_just_monoids_in_the_category_of/)
or [Hacker News](https://news.ycombinator.com/item?id=16093508) - but please
remember to be kind :)

*I want to thank [Tim Adler](https://twitter.com/r4dler) and
[mxf+](https://twitter.com/9b17fe) for proof-reading this absurdly long post
and for making many helpful suggestions for improvements*

---

[^1]: It is often confusing to people, that the way the
arrows point in the notation and the order they are written seems to contradict
each other: When writing \\(f:X\to Y\\) and \\(g:Y\to Z\\) you might reasonably
expect their composite to work like \\(f\circ g: X\to Z\\), that is, you glue
together the arrows in the order you are writing them.

The fact that we are not doing that is a completely justified criticism,
that is due to a historical accident - we write function application from
right to left, that is we write \\(f(x)\\), for applying \\(f\\) to \\(x\\).
Accordingly, we write \\(g(f(x))\\), when applying \\(g\\) to the result of
applying \\(f\\) to \\(x\\). And we chose to have the composite-notation be
consistent with *that*, instead of the arrow-notation.

I chose to just eat the unfortunate confusion, as it turns out Haskell is
doing exactly the same thing, so swapping things around would just increase
the confusion.

Sorry.

[^2]: Keep in mind that this is a different notion from the
ones for monoids, which we come to a bit later: While the formulas seem the
same and the identities look like a unit, the difference is that only certain
arrows can be composed, not all. And that there are many identity arrows, not
just one. However, if we would have only *one* object, it would have to be the
domain and codomain of every arrow and there would be exactly one identity
arrow. In that case, the notions *would* be the same and indeed, "a category
with exactly one object" is yet another way to define monoids.

[^3]: It is customary, to use the same name for the object
and arrow map, even though that may seem confusing. A slight justification of
that would be, that the object map is already given by the arrow map anyway: If
\\(F\\) is the arrow map, we can define the object map as \\(X\mapsto
\mathrm{dom}(F(\mathrm{id}\_X))\\). So, given that they are always occurring
together and you can make one from the other, we tend to just drop the
distinction and save some symbols.

What was that? Oh, you thought Mathematicians where precise? Ha!

[^4]: It is important to note, that this is not really a
*function*. Functions operate on values of a given type. But here, we are
operating on *types* and Haskell has no concept of a "type of types" built in
that a function could operate on. There are constructs operating on types to
construct new types, like `data`, `type`, `newtype` or even `deriving`. But
they are special syntactical constructs that exist outside of the realm of
functions.

This is one of the things that was tripping me up for a while: I was trying to
figure out, how I would map types to other types in Haskell or even talk about
the object map. But the most useful answer is "you don't".

[^5]: An important note here, is that the \\(\eta\_X\\) are
*arrows*. Where the object map of a functor is just a general association which
could look anything we like, the components of a natural transformation need to
preserve the internal structure of the category we are working in.

[^6]: You will often see these conditions written
differently, namely written e.g. \\(\mu M\\) instead of \\(\mu\iota\\). You can
treat that as a notational shorthand, it really means the same thing.

[^7]: There is a technicality here, that Haskell also has
an intermediate between functor and monad called "applicative". As I understand
it, this does not have a clear category theoretical analogue. I'm not sure why
it exits, but I believe it has been added into the hierarchy after the fact.

<script defer crossorigin="anonymous" src="https://cdnjs.cloudflare.com/ajax/libs/mathjax/2.7.2/MathJax.js?config=TeX-MML-AM_CHTML"></script>
