---
date: "2017-08-06T20:08:56Z"
tags:
- golang
- programming
aliases:
- "/2017/08/06/what-i-want-from-a-logging-api.html"
feedID: "https://blog.merovius.de//2017/08/06/what-i-want-from-a-logging-api.html"
title: What I want from a logging API
summary: Logging in Go is a notoriously lacking topic in the standard library. There
  are 3rd-party libraries trying to work around this. I'm trying to explain, why I
  find them still lacking
---

**This is intended as an [Experience
Report](https://github.com/golang/go/wiki/ExperienceReports) about logging in
Go. There are many like it but this one is mine.**

I have been trying for a while now to find (or build) a logging API in Go that
fills my needs. There are several things that make this hard to get "right"
though. This is my attempt to describe them coherently in one place.

When I say "logging", I mean informational text messages for human consumption
used to debug a specific problem. There is an idea currently gaining traction
in the Go community called "structured logging".
[logrus](https://github.com/sirupsen/logrus) is a popular package that
implements this idea. If you haven't heard of it, you might want to skim its
README. And while I definitely agree that log-messages should
contain some structural information that is useful for later filtering (like
the current time or a request ID), I believe the idea as often advocated is
somewhat misguided and conflates different use cases that are better addressed
otherwise. For example, if you are tempted to add a structured field to your
log containing an HTTP response code to alert on too many errors, you probably
want to use [metrics and
timeseries](https://landing.google.com/sre/book/chapters/practical-alerting.html)
instead. If you want to follow a field through a variety of systems, you
probably want to [annotate a
trace](https://research.google.com/pubs/pub36356.html). If you want analytics
like calculating daily active users or what used user-agents are used how
often, you probably want what I like to call [request
annotations](https://research.google.com/pubs/pub36632.html), as these are
properties of a request, not of a log-line. If you exclude all these use cases,
there isn't a lot left for structured logging to address.

The logs I am talking about is to give a user or the operator of a software
more insight into what is going on under the covers. The default assumption is,
that they are not looked at until something goes wrong: Be it a test failing,
an alerting system notifying of an issue or a bug report being investigated or
a CLI not doing what the user expected. As such it is important that they are
verbose to a certain degree. As an operator, I don't want to find out that I
can't troubleshoot a problem because someone did not log a critical piece of
information. An API that requires (or encourages) me to only log structured
data will ultimately only discourage me from logging at all. In the end, some
form of `log.Debugf("Error reading foo: %v", err)` is the perfect API for my use
case. Any structured information needed to make this call practically useful
should be part of the setup phase of whatever `log` is.

The next somewhat contentious question is whether or not the API should support
log levels (and if so, which). My personal short answer is "yes and the log
levels should be Error, Info and Debug". I could try and justify these specific
choices but I don't think that really helps; chalk it up to personal
preference if you like. I believe having *some* variation on the
verbosity of logs is very important. A CLI should be quiet by default but be
able to tell the user more specifically where things went wrong on request. A
service should be debuggable in depth, but unconditionally logging verbosely
would have in unacceptable latency impact in production and too heavy storage
costs. There need to be *some* logs by default though, to get quick insights
during an emergency or in retrospect. So, those three levels seem fine to me.

Lastly what I need from a logging API, is the possibility to set up verbosity
and log sinks both horizontally *and* vertically. What I mean by that is that
software is usually build in layers. They could be individual microservices,
Go packages or types. Requests will then traverse these layers vertically,
possibly branching out and interleaved to various degrees.

![Request forest](/assets/request_forest.svg)

Depending on what and how I am debugging, it makes sense to increase the log
verbosity of a particular layer (say I narrowed down the problem to shared
state in a particular handler and want to see what happens to that state during
multiple requests) or for a particular request (say, I narrowed down a problem
to "requests which have header FOO set to BAR" and want to follow one of them
to get a detailed view of what it does). Same with logging sinks, for example,
a request initiated by a test should get logged to its `*testing.T` with
maximum verbosity, so that I get a detailed context about it if and only if the
test fails to immediately start debugging. These settings should be possible
during runtime without a restart. If I am debugging a production issue, I
don't want to change a command line flag and restart the service.

Let's try to implement such an API.

We can first narrow down the design space a bit, because we want to use
`testing.T` as a logging sink. A `T` has several methods that would suit our
needs well, most notably [Logf](http://godoc.org/testing#T.Logf). This suggest
an interface for logging sinks that looks somewhat like this:

```go
type Logger interface {
	Logf(format string, v ...interface{})
}

type simpleLogger struct {
	w io.Writer
}

func (l simpleLogger) Logf(format string, v ...interface{}) {
	fmt.Fprintf(l.w, format, v...)
}

func NewLogger(w io.Writer) Logger {
	return simpleLogger{w}
}
```

This has the additional advantage, that we can add easily implement a
Discard-sink, that has minimal overhead (not even the allocations of
formatting the message):

```go
type Discard struct{}

func (Discard) Logf(format string, v ...interface{}) {}
```

The next step is to get leveled logging. The easiest way to achieve this is
probably

```go
type Logs struct {
	Debug Logger
	Info Logger
	Error Logger
}

func DiscardAll() Logs {
	return Logs{
		Debug: Discard{},
		Info: Discard{},
		Error: Discard{},
	}
}
```

By putting a struct like this (or its constituent fields) as members of a
handler, type or package, we can get the horizontal configurability we are
interested in.

To get vertical configurability we can use
[context.Value](http://godoc.org/context#Context.Value) - as much as it's
frowned upon by some, it is the canonical way to get request-scoped
behavior/data in Go. So, let's add this to our API:

```go
type ctxLogs struct{}

func WithLogs(ctx context.Context, l Logs) context.Context {
	return context.WithValue(ctx, ctxLogs{}, l)
}

func GetLogs(ctx context.Context, def Logs) Logs {
	// If no Logs are in the context, we default to its zero-value,
	// by using the ,ok version of a type-assertion and throwing away
	// the ok.
	l, _ := ctx.Value(ctxLogs{}).(Logs)
	if l.Debug == nil {
		l.Debug = def.Debug
	}
	if l.Info == nil {
		l.Info = def.Info
	}
	if l.Error == nil {
		l.Error = def.Error
	}
	return l
}
```

So far, this is a sane, simple and easy to use logging API. For example:

```go
type App struct {
	L log.Logs
}

func (a *App) ServeHTTP(res http.ResponseWriter, req *http.Request) {
	l := log.GetLogs(req.Context(), a.L)
	l.Debug.Logf("%s %s", req.Method, req.URL.Path)
	// ...
}
```

The issue with this API, however, is that it is completely inflexible, if we
want to preserve useful information like the file and line number of the
caller. Say, I want to implement the equivalent of
[io.MultiWriter](http://godoc.org/io#MultiWriter). For example, I want to write
logs both to `os.Stderr` and to a file and to a network log service.

I might try to implement that via

```go
func MultiLogger(ls ...Logger) Logger {
	return multiLog{ls}
}

type multiLog struct {
	loggers []Logger
}

func (m *multiLog) Logf(format string, v ...interface{}) {
	for _, l := range m.loggers {
		m.Logf(format, v...)
	}
}
```

However, now the caller of `Logf` of the individual loggers will be the line in
`(*multiLog).Logf`, *not* the line of its caller. Thus, caller information will
be useless. There are two APIs currently existing in the stdlib to work around this:

1. [(testing.T).Helper](https://tip.golang.org/pkg/testing/#T.Helper) (from
   Go 1.9) lets you mark a frame as a test-helper. When the caller-information
   is then added to the log-output, all frames marked as a helper is skipped.
   So, theoretically, we could add a `Helper` method to our Logger interface
   and require that to be called in each wrapper. However, `Helper` *itself*
   uses the same caller-information. So all wrappers must call the `Helper`
   method of the *underlying `*testing.T`*, without any wrapping methods. Even
   embedding doesn't help, as the Go compiler creates an [implicit wrapper](https://play.golang.org/p/Z8MHOrGAAt)
   for that.
2. [(log.Logger).Output](http://godoc.org/log#Logger.Output) lets you
   specify a number of call-frames to skip. We could add a similar method to
   our log sink interface. And wrapping loggers would then need to increment
   the passed in number, when calling a wrapped sink. It's possible to do this,
   but it wouldn't help with test-logs.

This is a very similar problem to the ones I wrote about
[last week](/posts/2017-07-30-the-trouble-with-optional-interfaces/).
For now, I am using the technique I described as [Extraction
Methods](/posts/2017-07-30-the-trouble-with-optional-interfaces/#extraction-methods).
That is, the modified API is now this:

```go
// Logger is a logging sink.
type Logger interface {
	// Logf logs a text message with the given format and values to the sink.
	Logf(format string, v ...interface{})

	// Helpers returns a list of Helpers to call into from all helper methods,
	// when wrapping this Logger. This is used to skip frames of logging
	// helpers when determining caller information.
	Helpers() []Helper
}

type Helper interface {
	// Helper marks the current frame as a helper method. It is then skipped
	// when determining caller information during logging.
	Helper()
}

// Callers can be used as a Helper for log sinks who want to log caller
// information. An empty Callers is valid and ready for use.
type Callers struct {
	// ...
}

// Helper marks the calling method as a helper. When using Callers in a
// Logger, you should also call this to mark your methods as helpers.
func (*Callers) Helper() {
	// ...
}

type Caller struct {
	Name string
	File string
	Line int
}

// Caller can be used to determine the caller of Logf in a Logger, skipping all
// frames marked via Helper.
func (*Callers) Caller() Caller {
	// ...
}

// TestingT is a subset of the methods of *testing.T, so that this package
// doesn't need to import testing.
type TestingT interface {
	Logf(format string, v ...interface{})
	Helper()
}

// Testing returns a Logger that logs to t. Log lines are discarded, if the
// test succeeds.
func Testing(t TestingT) Logger {
	return testLogger{t}
}

type testLogger struct {
	t TestingT
}

func (l testLogger) Logf(format string, v ...interface{}) {
	l.t.Helper()
	l.t.Logf(format, v...)
}

func (l testLogger) Helpers() []Helper {
	return []Helper{l.t}
}

// New returns a logger writing to w, prepending caller-information.
func New(w io.Writer) Logger {
	return simple{w, new(Callers)}
}

type simple struct {
	w io.Writer
	c *Callers
}

func (l *simple) Logf(format string, v ...interface{}) {
	l.c.Helper()
	c := l.c.Caller()
	fmt.Fprintf(l.w, "%s:%d: " + format, append([]interface{}{c.File, c.Line}, v...)...)
}

func (l *simple) Helpers() []Helper {
	return []Helper{l.c}
}

// Discard discards all logs.
func Discard() Logger {
	return discard{}
}

type discard struct{}

func (Discard) Logf(format string, v ...interface{}) {
}

func (Discard) Helpers() []Helper {
	return nil
}

// MultiLogger duplicates all Logf-calls to a list of loggers.
func MultiLogger(ls ...Logger) Logger {
	var m multiLogger
	for _, l := range ls {
		m.helpers = append(m.helpers, l.Helpers()...)
	}
	m.loggers = ls
	return m
}

type multiLogger struct {
	loggers []Logger
	helpers []Helper
}

func (m multiLogger) Logf(format string, v ...interface{}) {
	for _, h := range m.helpers {
		h.Helper()
	}
	for _, l := range m.loggers {
		l.Logf(format, v...)
	}
}

func (m multiLogger) Helpers() []Helper {
	return m.helpers
}

```

It's a kind of clunky API and I have no idea about the performance implications
of all the Helper-code.  But it *does* work, so it is, what I ended up with for
now.  Notably, it puts the implementation complexity into the *implementers* of
Logger, in favor of making the actual consumers of them as simple as possible.
