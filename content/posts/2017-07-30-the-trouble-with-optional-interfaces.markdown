---
date: "2017-07-30T18:39:00Z"
tags:
- golang
- programming
aliases:
- "/2017/07/30/the-trouble-with-optional-interfaces.html"
feedID: "https://blog.merovius.de//2017/07/30/the-trouble-with-optional-interfaces.html"
title: The trouble with optional interfaces
summary: 'I take a look at the pattern of optional interfaces in Go: what they are used
  for, why they are bad and what we can do about it.'
---

**tl;dr: I take a look at the pattern of optional interfaces in Go: what they
are used for, why they are bad and what we can do about it.**

*Note: I wrote most of this article on Wednesday, with the intention to finish
and publish it on the weekend. While I was sleeping, Jack Lindamood published
[this
post](https://medium.com/@cep21/interface-wrapping-method-erasure-c523b3549912),
which talks about much of the same problems.
[This](https://twitter.com/TheMerovius/status/890472264931708928) was the exact
moment I saw that post :) I decided, to publish this anyway; it contains, in my
opinion, enough additional content, to be worth it. But I do encourage to
(also?) read his post.*

## What are optional interfaces?

Optional interfaces are interfaces which can optionally be extended by
implementing some other interface. A good example is
[http.Flusher](http://godoc.org/net/http#Flusher) (and similar), which is
optionally implemented by an
[http.ResponseWriter](http://godoc.org/net/http#ResponseWriter). If a request
comes in via HTTP/2, the ResponseWriter will implement this interface to
support [HTTP/2 Server Push](https://en.wikipedia.org/wiki/HTTP/2_Server_Push).
But as not all requests will be over HTTP/2, this isn't part of the normal
ResponseWriter interface and instead provided via an optional interface that
needs to be type-asserted at runtime.

In general, whenever some piece of code is doing a type-assertion with an
interface type (that is, use an expression `v.(T)`, where `T` is an interface
type), it is very likely offering an optional interface.

A far from exhaustive list of where the optional interface pattern is used (to
roughly illustrate the scope of the pattern):

* [io](http://godoc.org/io#Copy)
* [net/http](http://godoc.org/net/http#ResponseWriter#Flusher)
* [database/sql/driver](http://godoc.org/database/sql/driver#ConnBeginTx)
* [go/types](http://godoc.org/go/types#Importer)
* Dave Chaney's [errors package](http://godoc.org/github.com/pkg/errors#Cause)

## What are people using them for?

There are multiple reasons to use optional interfaces. Let's find examples for
them. Note that this list neither claims to be exhaustive (there are probably
use cases I don't know about) nor disjunct (in some cases, optional interfaces
will carry more than one of these use cases). But I think it's a good rough
partition to discuss.

### Passing behavior through API boundaries

This is the case of `ResponseWriter` and its optional interfaces. The API, in
this case, is the `http.Handler` interface that users of the package implement
and that the package accepts. As features like HTTP/2 Push or connection
hijacking are not available to all connections, this interface needs to use the
lowest common denominator between all possible behaviors. So, if more features
need to be supported, we must somehow be able to pass this optional behavior
through the `http.Handler` interface.

### Enabling optional optimizations/features

[io.Copy](http://godoc.org/io#Copy) serves as a good example of this. The
required interfaces for it to work are just `io.Reader` and `io.Writer`. But it
can be made more efficient, if the passed values also implement `io.WriterTo`
or `io.ReaderFrom`, respectively. For example, a
[bytes.Reader](http://godoc.org/bytes#Reader.WriteTo) implements `WriteTo`.
This means, you need less copying if the source of an `io.Copy` is a
`bytes.Reader`. Compare these two (somewhat naive) implementations:

```go
func Copy(w io.Writer, r io.Reader) (n int64, err error) {
	buf := make([]byte, 4096)
	for {
		rn, rerr := r.Read(buf)
		wn, werr := w.Write(buf[:rn])
		n += int64(wn)
		if rerr == io.EOF {
			return n, nil
		}
		if rerr != nil {
			return n, rerr
		}
		if werr != nil {
			return n, werr
		}
	}
}

func CopyTo(w io.Writer, r io.WriterTo) (n int64, err error) {
	return r.WriteTo(w)
}

type Reader []byte

func (r *Reader) Read(b []byte) (n int, err error) {
	n = copy(b, *r)
	*r = (*r)[n:]
	if n == 0 {
		err = io.EOF
	}
	return n, err
}

func (r *Reader) WriteTo(w io.Writer) (int64, error) {
	n, err := w.Write(*r)
	*r = (*r)[n:]
	return int64(n), err
}
```

`Copy` needs to first allocate a buffer, then copy all the data from the
`*Reader` to that buffer, then pass it to the Writer. `CopyTo`, on the other
hand, can directly pass the byte-slice to the Writer, saving an allocation and
a copy.

Some of that cost can be amortized, but in general, its existence is a forced
consequence of the API. By using optional interfaces, `io.Copy` can use the
more efficient method, if supported, and fall back to the slow method, if not.

### Backwards compatible API changes

When `database/sql` upgraded to use `context`, it needed help from the drivers
to actually implement cancellation and the like. So it needed to add contexts
to the methods of [driver.Conn](http://godoc.org/database/sql/driver#Conn). But
it can't just do that change; it would be a backwards incompatible API change,
violating the Go1 compatibility guarantee. It also can't add a new method to
the interface to be used, as there are third-party implementations for drivers,
which would be broken as they don't implement the new method.

So it instead resorted to
[deprecate](https://golang.org/src/database/sql/driver/driver.go#L159) the old
methods and instead encourage driver implementers to add optional methods
including the context.

## Why are they bad?

There are several problems with using optional interfaces. Some of them have
workarounds (see below), but all of them have drawbacks on their own.

### They violate static type safety

In a lot of cases, the consumer of an optional interface can't really treat it
as optional. For example, `http.Hijacker` is usually used to support
WebSockets. A handler for WebSockets will, in general, not be able to do
anything useful, when called with a `ResponseWriter` that does not implement
`Hijacker`. Even when it correctly does a comma-ok type assertion to check
for it, it can't do anything but serve an error in that case.

The http.Hijacker type conveys the necessity of hijacking a connection, but
since it is provided as an optional interface, there is no possibility to
require this type statically. In that way, optional interfaces hide static type
information.

### They remove a lot of the power of interfaces

Go's interfaces are really powerful by being very small; in general, the
advice is to only add one method, maybe a small handful. This advice enables
easy and powerful composition. `io.Reader` and `io.Writer` have a myriad of
implementations inside and outside of the standard library. This makes it
really easy to, say, read uncompressed data from a compressed network
connection, while streaming it to a file and hashing it at the same time to
write to some content-addressed blob storage.

Now, this composition will, in general, destroy any optional interfaces of
those values. Say, we have an HTTP middleware to log requests. It wants to wrap
an `http.Handler` and log the requests method, path, response code and duration
(or, equivalently, collect them as metrics to export). This is, in principle,
easy to do:

```go
type logResponder struct {
	http.ResponseWriter
	code int
	set bool
}

func (rw *logResponder) WriteHeader(code int) {
	rw.code = code
	rw.set = bool
	rw.ResponseWriter.WriteHeader(code)
}

func LogRequests(h http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		lr := &logResponder{ResponseWriter: w}
		m, p, start := r.Method, r.Path, time.Now()
		defer func() {
			log.Printf("%s %s -> %d (%v)", m, p, lr.code, time.Now().Sub(start))
		}()
		h(lr, r)
	})
}
```

But `*logResponder` will now *only* support the methods declared by
`http.ResponseWriter`, even if the wrapped `ResponseWriter` also supports some
of the optional interfaces. That is because method sets of a type are
determined at compile time.

Thus, by using this middleware, the wrapped handler is suddenly unable to use
websockets, or HTTP/2 server push or any of the other use cases of optional
interfaces. Even worse: this deficiency will only be discovered at runtime.

Optimistically adding the optional interface's methods and type-asserting the
underlying ResponseWriter at runtime doesn't work either: handlers would
incorrectly conclude the optional interface is always present. If the
underlying `ResponseWriter` does not support adding at the underlying
connection there just is no useful way to implement `http.Hijacker`.

There is one way around this, which is to dynamically check the wrapped
interface and create a type with the correct method set, e.g.:

```go
func Wrap(wrap, with http.ResponseWriter) http.ResponseWriter {
	var (
		flusher http.Flusher
		pusher http.Pusher
		// ...
	)
	flusher, _ = wrap.(http.Flusher)
	pusher, _ = wrap.(http.Pusher)
	// ...
	if flusher == nil && pusher == nil {
		return with
	}
	if flusher == nil && pusher != nil {
		return struct{
			http.ResponseWriter
			http.Pusher
		}{with, pusher}
	}
	if flusher != nil && pusher == nil {
		return struct{
			http.ResponseWriter
			http.Flusher
		}{with, flusher}
	}
	return struct{
		http.ResponseWriter
		http.Flusher
		http.Pusher
	}{with, flusher, pusher}
}
```

This has two major drawbacks:

* Both code-size and running time of this will increase exponentially with the
  number of optional interfaces you have to support (even if you generate the
  code).
* You need to know every single optional interface that might be used. While
  supporting everything in `net/http` is certainly tenable, there might be
  other optional interfaces, defined by some framework unbeknownst to you. If
  you don't know about it, you can't wrap it.

## What can we use instead?

My general advice is, to avoid optional interfaces as much as possible. There
are alternatives, though they also are not entirely satisfying.

### Context.Value

`context` was added after most of the optional interfaces where already
defined, but its `Value` method was meant exactly for this kind of thing: to
pass optional behavior past API boundaries. This will still not solve the
static type safety issue of optional interfaces, but it does mean you can
easily wrap them.

For example, `net/http` could instead do

```go
var ctxFlusher = ctxKey("flusher")

func GetFlusher(ctx context.Context) (f Flusher, ok bool) {
	f, ok = ctx.Value(ctxFlusher).(Flusher)
	return f, ok
}
```

This would enable you to do

```go
func ServeHTTP(w http.ResponseWriter, r *http.Request) {
	f, ok := http.GetFlusher(r.Context())
	if ok {
		f.Flush()
	}
}
```

If now a middleware wants to wrap `ResponseWriter`, that's not a problem, as it
will not touch the Context. If a middleware wants to add some other optional
behavior, it can do so easily:

```go
type Frobnicator interface{
	Frobnicate()
}

var ctxFrobnicator = ctxKey("frobnicator")

func GetFrobnicator(ctx context.Context) (f Frobnicator, ok bool) {
	f, ok = ctx.Value(ctxFrobnicator).(Frobnicator)
	return f, ok
}
```

As contexts form a linked list of key-value-pairs, this will interact nicely
with whatever optional behavior is already defined.

There are good reasons to frown upon the usage of `Context.Value`; but they
apply just as much to optional interfaces.

### Extraction methods

If you know an interface type that is probable to be wrapped and *also* has
optional interfaces associated it is possible to enforce the possibility of
dynamic extension in the optional type. So, e.g.:

```go
package http

type ResponseWriter interface {
	// Methods…
}

type ResponseWriterWrapper interface {
	ResponseWriter

	WrappedResponseWriter() ResponseWriter
}

// GetFlusher returns an http.Flusher, if res wraps one.
// Otherwise, it returns nil.
func GetFlusher(res ResponseWriter) Flusher {
	if f, ok := res.(Flusher); ok {
		return f
	}
	if w, ok := res.(ResponseWriterWrapper); ok {
		return GetFlusher(w.WrappedResponseWriter())
	}
	return nil
}

package main

type logger struct {
	res ResponseWriter
	req *http.Request
	log *log.Logger
	start time.Time
}

func (l *logger) WriteHeader(code int) {
	d := time.Now().Since(l.start)
	l.log.Write("%s %s -> %d (%v)",	l.req.Method, l.req.Path, code, d)
	l.res.WriteHeader(code)
}

func (l *logger) WrappedResponseWriter() http.ResponseWriter {
	return l.res
}

func LogRequests(h http.Handler, l *log.Logger) http.Hander {
	return http.HandlerFunc(res http.ResponseWriter, req *http.Request) {
		res = &logger{
			res: res,
			req: req,
			log: l,
			start: time.Now(),
		}
		h.ServeHTTP(res, req)
	}
}

func ServeHTTP(res http.ResponseWriter, req *http.Request) {
	if f := http.GetFlusher(res); f != nil {
		f.Flush()
	}
}
```

This still doesn't address the static typing issue and explicit dependencies,
but at least it enables you to wrap the interface conveniently.

Note, that this is conceptually similar to the [errors
package](https://github.com/pkg/errors), which calls the wrapper-method
"Cause". This package also shows an issue with this pattern; it only
works if *all* wrappers use it. That's why I think it's important for the
wrapping interface to live in the same package as the wrapped interface; it
provides an authoritative way to do that wrapping, preventing fragmentation.

### Provide statically typed APIs

`net/http` could provide alternative APIs for optional interfaces that
explicitly include them. For example:

```go
type Hijacker interface {
	ResponseWriter
	Hijack() (net.Conn, *bufio.ReadWriter, error)
}

type HijackHandler interface{
	ServeHijacker(w Hijacker, r *http.Request)
}

func HandleHijacker(pattern string, h HijackHandler) {
	// ...
}
```

For some use cases, this provides a good way to side-step the issue of unsafe
types. Especially if you can come up with a limited set of scenarios that would
rely on the optional behavior, putting them into their own type would be
viable.

The `net/http` package could, for example, provide separate `ResponseWriter`
types for different connection types (for example `HTTP2Response`). It could
then provide a `func(HTTP2Handler) http.Handler`, that serves an error if it is
asked to serve an unsuitable connection and otherwise delegates to the passed
Handler. Now, the programmer needs to explicitly wire a handler that requires
HTTP/2 up accordingly. They can rely on the additional features, while also
making clear what paths must be used over HTTP/2.

### Gradual repair

I think the use of optional interfaces as in `database/sql/driver` is perfectly
fine - *if* you plan to eventually remove the original interface. Otherwise,
users will have to continue to implement both interfaces to be usable with your
API, which is especially painful when wrapping interfaces. For example, I
recently wanted to wrap
[importer.Default](http://godoc.org/go/importer#Default) to add behavior and
logging. I also needed [ImporterFrom](http://godoc.org/go/types#ImporterFrom),
which required separate implementations, depending on whether the importer
returned by Default implements it or not. Most modern code, however, shouldn't
need that.

So, for third party packages (the stdlib can't do that, because of
compatibility guarantees), you should consider using the methodology described
in Russ Cox' excellent [Codebase Refactoring](https://talks.golang.org/2016/refactor.article)
article and actually *deprecate* and eventually *remove* the old interface. Use
optional interfaces as a transition mechanism, not a fix.

## How could Go improve the situation?

### Make it possible for reflect to create methods

There are currently at least two GitHub issues which would make it possible to
do extend interfaces dynamically:
[reflect: NamedOf](https://github.com/golang/go/issues/16522), [reflect: MakeInterface](https://github.com/golang/go/issues/4146).
I believe this would be the easiest solution - it is backwards compatible and
doesn't require any language changes.

### Provide a language mechanism for extension

The language could provide a native mechanism to express extension, either by
adding a
[keyword](https://medium.com/@cep21/interface-wrapping-method-erasure-c523b3549912#13bc)
for it or, for Go2, by considering to make extension the default behavior for
`interface->struct` embedding. I'm not sure either is a good idea, though. I
would probably prefer the latter, because of my distaste for keywords. Note,
that it would still be possible to then compose an interface into a struct,
just not via embedding but by adding a field and delegation-methods.
Personally, I'm not a huge fan of embedding interfaces in structs anyway except
when I'm explicitly trying to extend them with additional behavior.  Their
zero-value is not usable, so it requires additional hoops to jump through.

## Conclusion

I recommend:

* If at all possible, avoid optional interfaces in APIs you provide. They are
  just too inconvenient and un-Go-ish.
* Be careful when wrapping interfaces, in particular when there are known
  optional interfaces for them.

Using optional interfaces correctly is inconvenient and cumbersome. That should
signal that you are fighting the language. The workarounds needed all try to
circumvent one or more design decision of Go: to value composition over
inheritance, to prefer static typing and to make computation and behavior
obvious from code. To me, that signifies that optional interfaces are
fundamentally not a good fit for the language.
