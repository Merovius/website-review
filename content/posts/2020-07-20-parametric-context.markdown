---
date: "2020-07-20T00:00:00Z"
tags:
- golang
- programming
time: "2020-07-20 21:45:00"
aliases:
- "/2020/07/20/parametric-context.html"
feedID: "https://blog.merovius.de//2020/07/20/parametric-context.html"
title: Parametric context
summary: Go's Context.Value is controversial because of a lack of type-safety. I design
  a solution for that based on the new generics design draft.
---

**tl;dr: Go's Context.Value is controversial because of a lack of type-safety. I design a solution for that based on the new generics design draft.**

If you are following what's happening with Go, you are aware that recently
[an updated design draft for generics has dropped][go blog].
What makes this particularly notable is that it comes with an actual prototype
implementation of the draft, including a [playground].
This means for the first time, people get to actually try out how a Go with
generics might feel, once they get in. It is a good opportunity to look at
common Go code lacking type-safety and evaluate if and how generics can help
address them.

One area I'd like to look at here is [Context.Value]. It is often criticized
for not being explicit enough about the dependencies a function has and some
people even go so far as to discourage its use altogether. On the other hand,
I'm on record [saying that it is too useful to ignore][why context]. Generics
might be a way to bring together these viewpoints.

We want to be able to declare dependency on a functionality in
`context.Context` via a function's signature and make it impossible to call it
without providing that functionality, while also preserving the ability to pass
it through APIs that don't know anything about it. As an example of such
functionality, I will use logging. Let's start by creating a fictional little
library to do that (the names are not ideal, but let's not worry about that):

```go
package logctx

import (
    "context"
    "log"
)

type LogContext interface {
    // We embed a context.Context, to say that we are augmenting it with
    // additional functionality.
    context.Context

    // Logf logs the given values in the given format.
    Logf(format string, values ...interface{})
}

func WithLog(ctx context.Context, l *log.Logger) LogContext {
    return logContext{ctx, l}
}

// logContext is unexported, to ensure it can't be modified.
type logContext struct {
    context.Context
    l *log.Logger
}

func (ctx logContext) Logf(format string, values ...interface{}) {
    ctx.l.Printf(format, values...)
}
```

You might notice that we are not actually using `Value()` here. This is
fundamental to the idea of getting compiler-checks - we need some
compiler-known way to "tag" functionality and that can't be `Value`. However,
we provide the same functionality, by essentially adding an [optional
interface] to `context.Context`.

If we want to use this, we could write

```go
func Foo(ctx logctx.LogContext, v int) {
    ctx.Logf("Foo(%v)", v)
}

func main() {
    ctx := logctx.WithLog(context.Background(), log.New(os.Stderr, "", log.LstdFlags))
    Foo(ctx, 42)
}
```

However, this has a huge problem: What if we want more than one functionality
(each not knowing about the other)? We might try the same trick, say

```go
package tracectx

import (
    "context"

    "github.com/opentracing/opentracing-go"
)

type TraceContext interface {
    context.Context
    Tracer() opentracing.Tracer
}

func WithTracer(ctx context.Context, t opentracing.Tracer) TraceContext {
    return traceContext{ctx, t}
}

type traceContext struct {
    context.Context
    t opentracing.Tracer
}

func (ctx traceContext) Tracer() opentracing.Tracer {
    return ctx.t
}
```

But because a `context.Context` is embedded, only those methods explicitly
mentioned in that interface are added to `traceContext`. The `Logf` method is
erased. After all, that is [the trouble with optional interfaces][optional interface].

This is where generics come in. We can change our wrapper-types and -functions like this:

```go
type LogContext(type parent context.Context) struct {
    // the type-parameter is lower case, so the field is not exported.
    parent
    l *log.Logger
}

func WithLog(type Parent context.Context) (ctx Parent, l *log.Logger) LogContext(Parent) {
    return LogContext(parent){ctx, l}
}
```

By adding a type-parameter and embedding it, we actually get *all* methods of
the parent context on `LogContext`. We are no longer erasing them. After giving
the `tracectx` package the same treatment, we can use them like this:

```go
// FooContext encapsulates all the dependencies of Foo in a context.Context.
type FooContext interface {
    context.Context
    Logf(format string, values ...interface{})
    Tracer() opentracing.Tracer
}

func Foo(ctx FooContext, v int) {
    span := ctx.Tracer().StartSpan("Foo")
    defer span.Finish()

    ctx.Logf("Foo(%v)", v)
}

func main() {
    l := log.New(os.Stderr, "", log.LstdFlags)
    t := opentracing.GlobalTracer()
    // ctx has type TraceContext(LogContext(context.Context)),
    //    which embeds a LogContext(context.Context),
    //    which embeds a context.Context
    // So it has all the required methods
    ctx := tracectx.WithTracer(logctx.WithLog(context.Background(), l), t)
    Foo(ctx, 42)
}
```

`Foo` has now fully declared its dependencies on a logger and a tracectx, without
requiring any type-assertions or runtime-checks. The logging- and
tracing-libraries don't know about each other and yet are able to wrap each
other without loss of type-information. Constructing the context is not
particularly ergonomic though. We require a long chained function call, because
the values returned by the functions have no longer a unified type
`context.Context` (so the `ctx` variable can't be re-used).

Another thing to note is that we exported `LogContext` as a struct, instead of
an interface. This is necessary, because we can't embed type-parameters into
interfaces, but we *can* embed them as struct-fields. So this is the only way
we can express that the returned type has all the methods the parameter type
has. The downside is that we are making this a concrete type, which isn't
always what we want[^1].

We have now succeeded in annotating `context.Context` with dependencies, but
this alone is not super useful of course. We also need to be able to pass it
through agnostic APIs (the fundamental problem `Context.Value` solves).
However, this is easy enough to do.

First, let's change the `context` API to use the same form of generic wrappers.
This isn't backwards compatible, of course, but this entire blog post is a
thought experiment, so we are ignoring that. I don't provide the full code
here, for brevity's sake, but the basic API would change into this:

```go
package context

// CancelContext is the generic version of the currently unexported cancelCtx.
type CancelContext(type parent context.Context) struct {
    parent
    // other fields
}

func WithCancel(type Parent context.Context) (ctx Parent) (ctx CancelContext(Parent), cancel CancelFunc) {
    // ...
}
```

This change is necessary to enable `WithCancel` to also preserve methods of the
parent context. We can now use this in an API that passes through a parametric
context.  For example, say we want to have an [errgroup] package, that passes
the context through to the argument to `(*Group).Go`, instead of returning it
from `WithContext`:

```go
// Derived from the current errgroup code.

// A Group is a collection of goroutines working on subtasks that are part of the same overall task.
//
// A zero Group is invalid (as opposed to the original errgroup).
type Group(type Context context.Context) struct {
    ctx    Context
    cancel func()

    wg sync.WaitGroup

    errOnce sync.Once
    err     error
}

func WithContext(type C context.Context) (ctx C) *Group(C) {
    ctx, cancel := context.WithCancel(ctx)
    return &Group(C){ctx: ctx, cancel: cancel}
}

func (g *Group(Context)) Wait() error {
    g.wg.Wait()
    return g.err
}

func (g *Group(Context)) Go(f func(Context) error) {
    g.wg.Add(1)

    go func() {
        defer g.wg.Done()

        if err := f(g.ctx); err != nil {
            g.errOnce.Do(func() {
                g.err = err
            })
        }
    }()
}
```

Note that the code here has barely changed. It can be used as

```go
func Foo(ctx FooContext) error {
    span := ctx.Tracer().StartSpan("Foo")
    defer span.Finish()
    ctx.Logf("Foo was called")
}

func main() {
    var ctx FooContext = newFooContext()
    eg := errgroup.WithContext(ctx)
    for i := 0; i < 20; i++ {
        eg.Go(Foo)
    }
    if err := eg.Wait(); err != nil {
        log.Fatal(err)
    }
}
```

After playing around with this for a couple of days, I feel pretty confident
that these patterns make it possible to get a fully type-safe version of
`context.Context`, while preserving the ability to have APIs that pass it
through untouched or augmented.

A completely different question, of course, is whether all of this is a good
idea. Personally, I am on the fence about it. It is definitely valuable, to
have a type-safe version of `context.Context`. And I think it is impressive how
small the impact of it is on the *users* of APIs written this way. The
type-argument can almost always be inferred and writing code to make use of this
is very natural - you just declare a suitable context-interface and take it as
an argument. You can also freely pass it to functions taking a pure
`context.Context` unimpeded.

On the other hand, I am not completely convinced the cost is worth it. As soon
as you do non-trivial things with a context, it becomes a pretty "infectious"
change. For example, I played around with a [mock gRPC API] to allow
interceptors to take a parametric context and it requires almost all types and
functions involved to take a type-parameter. And this doesn't even touch on the
fact that gRPC itself might want to add annotations to the context, which adds
even more types. I am not sure if the additional machinery is really worth the
benefit of some type-safety - especially as it's not always super intuitive and
easily understandable. And even more so, if it needs to be combined with other
type-parameters, to achieve other goals.

I think this is an example of what I tend to dislike about generics and powerful
type-systems in general. They tempt you to write a lot of extra machinery and
types in a way that isn't necessarily semantically meaningful, but only used to
encode some invariant in a way the compiler understands.

---

[^1]: One *upside* however, is that this could actually address
the *other* criticism of `context.Value`: Its performance. If we consequently embed the
parent-context as values in struct fields, the final context will be a flat
struct. The interface-table of all the extra methods we add will point at the
concrete implementations. There's no longer any need for a linear search to
find a context value.

I don't actually think there is much of a performance problem with
`context.Value` in practice, but if there is, this could solve that.

[playground]: https://go2goplay.golang.org/
[Context.Value]: https://godoc.org/context#Context.Value
[errgroup]: https://godoc.org/golang.org/x/sync/errgroup
[go blog]: https://blog.golang.org/generics-next-step
[mock gRPC API]: https://go2goplay.golang.org/p/9-xQZufcGp_k
[optional interface]: /posts/2017-07-30-the-trouble-with-optional-interfaces/
[why context]: /posts/2017-08-14-why-context-value-matters-and-how-to-improve-it/
