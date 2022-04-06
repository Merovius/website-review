---
date: "2014-02-19T02:17:59Z"
tags:
- golang
- programming
aliases:
- "/2014/02/19/go-stacktraces.html"
feedID: "https://blog.merovius.de//2014/02/19/go-stacktraces.html"
title: go stacktraces
summary: A small pattern to add stacktraces to errors.
---

Let's say you write a library in [go](http://golang.org/) and want an easy way
to get debugging information from your users. Sure, you return `error`s from
everything, but it is sometimes hard to pinpoint where a particular error
occured and what caused it. If your package `panic`s, that will give you a
stacktrace, but as you probably know you shouldn't `panic` in case of an error,
but just gracefull recover and return the error to your caller.

I recently discovered a pattern which I am quite happy with (for now). You can
include a stacktrace when returning an error. If you disable this behaviour by
default you should have as good as no impact for normal users, while making it
much easier to debug problems. Neat.

```
package awesomelib

import (
	"os"
	"runtime"
)

type tracedError struct {
	err   error
	trace string
}

var (
	stacktrace bool
	traceSize = 16*1024
)

func init() {
	if os.Getenv("AWESOMELIB_ENABLE_STACKTRACE") == "true" {
		stacktrace = true
	}
}

func wrapErr(err error) error {
	// If stacktraces are disabled, we return the error as is
	if !stacktrace {
		return err
	}

	// This is a convenience, so that we can just throw a wrapErr at every
	// point we return an error and don't get layered useless wrappers
	if Err, ok := err.(*tracedError); ok {
		return Err
	}

	buf := make([]byte, traceSize)
	n := runtime.Stack(buf, false)
	return &tracedError{ err: err, trace: string(buf[:n]) }
}

func (err *tracedError) Error() string {
	return fmt.Sprintf("%v\n%s", err.err, err.trace)
}

func DoFancyStuff(path string) error {
	file, err := os.Open(path)
	if err != nil {
		return wrapErr(err)
	}
	// fancy stuff
}
```
