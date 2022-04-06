---
date: "2015-07-29T01:10:11Z"
tags:
- golang
- programming
aliases:
- "/2015/07/29/backwards-compatibility-in-go.html"
omitFeed: true
title: Backwards compatibility in go
summary: There are next to no "backwards compatible API changes" in go. You should explicitely
  name your compatibility-guarantees.
---

**tl;dr: There are next to no "backwards compatible API changes" in go. You
should explicitely name your compatibility-guarantees.**

I really love go, I really hate vendoring and up until now I didn't really get,
why anyone would think go should need something like that. After all, go seems
to be predestined to be used with automatically checked semantic versioning.
You can enumerate all possible changes to an API in go and the list is quite
short. By looking at vcs-tags giving semantic versions and diffing the API, you
can automatically check that you never break compatibility (the go compiler and
stdlib actually do something like that). Heck, in theory you could even write a
package manager that automatically (without any further annotations) determines
the latest version of a package that still builds all your stuff or gives you
the minimum set of packages that need changes to reconcile conflicts.

This thought lead me to contemplate what makes an API change a breaking
change. After a bit of thought, my conclusion is that almost every API change
is a breaking change, which might surprise you.

For this discussion we first need to make some assumptions about what
constitutes breakage. We will use the [go1 compatibility promise](http://golang.org/doc/go1compat).
The main gist is: Stuff that builds before is guaranteed to build after.
Notable exceptions (apart from necessary breakages due to security or other
bugs) are unkeyed struct literals and dot-imports.

**[edit]**
I should clarify, that whenever I talk about an API-change, I mean
your exported API as defined by Code (as opposed to comments/documentation).
This includes the public identifiers exported by your package, including type
information. It excludes API-requirements specified in documentation, like on
[io.Writer](http://golang.org/pkg/io/#Writer). These are just too complex to
talk about in a meaningful way and must be dealt with separately anyway.
**[/edit]**

So, given this definition of breakage, we can start enumerating all the
possible changes you could do to an API and check whether they are breaking
under the definition of the go1 compatibility promise:

## Adding func/type/var/const at package scope

This is the only thing that seems to be fine under the stability guarantee.  It
turns out the go authors thought about this one and put the exception of
dot-imports into the compatibility promise, which is great.

dot-imports are imports of the form `. import "foo"`. They import every
package-level identifier of package `foo` into the scope of the current file.

Absence of dot-imports means, every identifier at your package scope must be
referenced with a selector-expression (i.e. `foo.Bar`) which can't be redeclared
by downstream. It also means that you should never use dot-imports in your
packages (which is a bad idea for other reasons too). Treat dot-imports as a
historic artifact which is completely deprecated. An exception is the need
to use a separate `foo_test` package for your tests to break dependency cycles.
In that case it is widely deemed acceptable to `. import "foo"` to save typing
and add clarity.

## Removing func/type/var/const at package scope

Downstream might use the removed function/type/variable/constant, so this is
obviously a breaking change.

## Adding a method to an interface

Downstream might want to create an implementation of your interface and try to
pass it. After you add a method, this type doesn't implement your interface
anymore and downstreams code will break.

## Removing a method from an interface

Downstream might want to call this method on a value of your interface type, so
this is obviously a breaking change.

## Adding a field to a struct

This is perhaps surprising, but adding a field to a struct is a breaking
change. The reason is, that downstream might embed two types into a struct. If
one of them has a field or method Bar and the other is a struct you added the
Field Bar to, downstream will fail to build (because of an ambiguous selector
expression).

So, e.g.:

```go
// foo/foo.go
package foo

type Foo struct {
	Foo string
	Bar int // Added after the fact
}

// bar/bar.go
package bar

type Baz struct {
	Bar int
}

type Spam struct {
	foo.Foo
	Baz
}

func Eggs() {
	var s Spam
	s.Bar = 42 // ambiguous selector s.Bar
}
```

This is what the compatibility *might* refer to with the following quote:

> Code that uses unkeyed struct literals (such as pkg.T{3, "x"}) to create values
> of these types would fail to compile after such a change. However, code that
> uses keyed literals (pkg.T{A: 3, B: "x"}) will continue to compile after such a
> change.  We will update such data structures in a way that allows keyed struct
> literals to remain compatible, although unkeyed literals may fail to compile.
> (**There are also more intricate cases involving nested data structures or
> interfaces**, but they have the same resolution.)

(emphasis is mine). By "the same resolution" they *might* refer to only accessing
embedded Fields via a keyed selector (so e.g. `s.Baz.Bar` in above example). If
so, that is pretty obscure and it makes struct-embedding pretty much
useless. Every usage of a field or method of an embedded type must be
explicitly Keyed, which means you can just *not* embed it after all. You need
to write the selector and wrap every embedded method anyway.

I hope we all agree that type embedding is awesome and shouldn't need to be
avoided :)

## Removing a field from a struct

Downstream might use the now removed field, so this is obviously a breaking change.

## Adding a method to a type

The argument is pretty much the same as adding a field to a struct: Downstream
might embed your type and suddenly get ambiguities.

## Removing a method from a type

Downstream might call the now removed method, so this is obviously a breaking change.

## Changing a function/method signature

Most changes are obviously breaking. But as it turns out you can't do *any*
change to a function or method signature. This includes adding a variadic
argument which *looks* backwards compatible on the surface. After all, every
call site will still be correct, right?

The reason is, that downstream might save your function or method in a variable
of the old type, which will break because of nonassignable types.

## Conclusion

It looks to me like anything that isn't just adding a new Identifier to the
package-scope will potentially break *some* downstream. This severely limits
the kind of changes you can do to your API if you want to claim backwards
compatibility.

This of course doesn't mean that you should never ever make any changes to your
API ever. But you should think about it and you should clearly document, what
kind of compatibility guarantees you make. When you do any changes named in
this document, you should check your downstreams, whether they are affected by
it. If you claim a similar level of compatibility as the go standard library,
you should definitely be aware of the implications and what you can and can't
do.

We, the go community, should probably come up with some coherent definition of
what changes we deem backwards compatible and which we don't. A tool to
automatically looks up all your (public) importerts on
[godoc.org](https://godoc.org/), downloads the latest version and tries to
build them with your changes should be fairly simple to write in go (and may
even already exist). We should make it a standard check (like go vet and
golint) for upstream package authors to do that kind of thing before push to
prevent frustrated downstreams.

Of course there is still the possibility, that my reading of the go1
compatibility promise is wrong or inaccurate. I would welcome comments on that,
just like on everything else in this post :)
