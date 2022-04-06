---
date: "2018-01-21T23:40:00Z"
tags:
- golang
- programming
- thoughtleading
aliases:
- "/2018/01/21/what_even_is_error_handling.html"
feedID: "https://blog.merovius.de//2018/01/21/what_even_is_error_handling.html"
title: What even is error handling?
summary: I philosophize about error handling, what it actually means and how to characterize
  Go's approach to it.
---

**tl;dr: Error handling shouldn't be about how to best propagate an error
value, but how to make it destroy it (or make it irrelevant). To encourage
myself to do that, I started removing errors from function returns wherever I
found it at all feasible**

Error handling in Go is a contentious and often criticized issue. There is no
shortage on articles criticizing the approach taken, no shortage on articles
giving advice on how to deal with it (or defending it) and also no shortage on
proposals on how to improve it.

During these discussion, I always feel there is something missing. The
proposals for improvement usually deal with syntactical issues, how to avoid
boilerplate. Then there is the other school of thought - where it's not about
syntax, but about how to best pass errors around. Dave Chaney wrote [an often
quoted blog post on the
subject](https://dave.cheney.net/2016/04/27/dont-just-check-errors-handle-them-gracefully),
where he lists all the ways error information can be mapped into the Go type
system, why he considers them flawed and what he suggests instead.
This school of thought regularly comes up with helper packages, to make
wrapping or annotating errors easier.
[pkg/errors](https://github.com/pkg/errors) is very popular (and is grown out
of the approach of above blog post) but [upspin's
incarnation](https://godoc.org/github.com/upspin/upspin/errors#Error) also
gathered some attention.

I am dissatisfied with both schools of thought. Overall, neither seems to
explicitly address, what to me is the underlying question: What *is* error
handling? In this post, I'm trying to describe how I interpret the term and
why, to me, the existing approaches and discussions mostly miss the mark. Note,
that I don't claim this understanding to be universal - just how *I* would put
into words my understanding of the topic.

---

Let's start with a maybe weird question: Why is the entry point into the
program `func main()` and not `func main() error`? Personally, I start most of
my programs writing

```go
func main() {
  if err := run(); err != nil {
    log.Fatal(err)
  }
}

func run() error {
  // …
}
```

This allows me to use `defer`, pass on errors and all that good stuff. So, why
doesn't the language just do that for me?

We can find part of the answer in [this old golang-nuts thread](https://groups.google.com/d/topic/golang-nuts/6xl02B_MxdA/discussion).
It is about return codes, instead of an `error`, but the principle is the
same. And the best answer - in my opinion - is this:

> I think the returned status is OS-specific, and so Go the language should not
> define its type (Maybe some OS can only report 8-bit result while some other
> OS support arbitrary string as program status, there is considerable
> differences between that; there might even be environment that don't support
> returning status code or the concept of status code simply doesn't exist)
>
> I imagine some Plan 9 users might be disagree with the signature of
> `os.Exit()`.

So, in essence: Not all implementations would necessarily be able to assign a
reasonable meaning to a return code (or error) from `main`. For example, an
embedded device likely couldn't really do anything with it. It thus seems
preferable to not couple the language to this decision which only *really* makes
semantic sense on a limited subset of implementations. Instead, we provide
mechanisms in the standard library to exit the program or take any other
reasonable action and then let the developer decide, under what circumstances
they want to exit the program and with what code. Being coupled to a decision
in the standard library is better than being coupled in the language itself.
And a developer who targets a platform where an exit code doesn't make sense,
can take a different action instead.

Of course, this leaves the programmer with a problem: What to do with errors?
We could write it to stderr, but `fmt.Fprintf` *also* returns an error, so what
to do with that one? Above I used `log.Fatal`, which does *not* return an error.
What happens if the underlying `io.Writer` fails to write, though? What
does `log` do with the resulting error? The answer is, of course: It ignores
any errors.

The point is, that passing on the error is not a solution. *Eventually* every
program will return to `main` (or `os.Exit` or panic) and the buck stops there.
It needs to get *handled* and the signature of `main` enforces that the only
way to do that is via side-effects - and if they fail, you just have to deal
with that one too.

---

Let's continue with a similar question, that has a similar answer, that
occasionally comes up: Why doesn't `ServeHTTP` return an `error`? Sooner or
later, people face the question of what to do with errors in their HTTP
Handlers. For example, what if you are writing out a JSON object and
`Marshal` fails? In fact, a lot of HTTP frameworks out there will define their
own handler-type, which differs from `http.Handler` in exactly that way. But if
everyone wants to return an `error` from their handler, why doesn't the
interface just add that error return itself? Was that just an oversight?

I'm strongly arguing that no, this was not an oversight, but the correct design
decision. Because the HTTP Server package *can not handle any errors*. An HTTP
server is supposed to stay running, every request demands a response. If
`ServeHTTP` would return an `error`, the server would have to do *something*
with it, but what to do is highly application-specific. You might respond that
it should serve a 500 error code, but in 99% of cases, that is the wrong thing
to do. Instead you should serve a more specific error code, so the client
knows (for example) whether to retry or if the response is cacheable.
`http.Server` could also just ignore the error and instead drop the request on
the floor, but that's even worse. Or it could propagate it up the stack. But as
we determined, eventually it would have to reach `main` and the buck stops
there. You probably don't want your server to come down, every time a request
contains an invalid parameter.

So, given that a) every request needs an answer and b) the right answer is
highly application-specific, the translation from errors into status codes
*has* to happen in application code. And just like `main` enforces you to
handle any errors via side-effects by not allowing you to return an `error`, so
does `http` force you to handle any errors via writing a response by not
allowing you to return an `error`.[^1]

So, what are you supposed to do, when `json.Marshal` fails? Well, that depends
on our application. Increment a metric. Log the error. panic. Write out a 500.
Ignore it and write a 200. Commit to the uncomfortable knowledge, that
sometimes, you can't just pass the decision on what to do with an error to
someone else.

---

These two examples distill, I think, pretty well, what I view as error
*handling*: An error is handled, when you destroy the error value. In that
parlance, `log.Error` handles any errors of the underlying writer by not
returning them. Every program needs to handle any error in *some* way, because
`main` can't return anything and the values need to go *somewhere*. Any HTTP
handler needs to actually *handle* errors, by translating them into HTTP
responses.

And in that parlance, packages like `pkg/errors` have little, really, to do with
error *handling* - instead, they provides you with a strategy for the case where
you are *not* handling your errors. In the same vein, proposals that address
the repetitive checking of errors via extra syntax do not really simplify their
handling at all - they just move it around a bit. I would term that *error
propagation*, instead - no doubt important, but keep in mind, that an error
that was *handled*, doesn't need to be propagated at all. So to me, a good
approach to error handling would be characterized by mostly obviating the need
for convenient error propagation mechanisms.

And to me, at least, it seems that we talk too little about how to handle
errors, in the end.

---

Does Go encourage explicit error handling? This is the phrasing very often used
to justify the repetitive nature, but I tend to disagree. Compare, for example,
Go's approach to checked exceptions in Java: There, errors are propagated via
exceptions. Every exception that could be thrown (theoretically) must be
annotated in the method signature. Any exception that you handle, has to be
mentioned in a try-catch-statement. And the compiler will refuse to compile a
program which does not explicitly mention how exceptions are handled. This, to
me, seems like the pinnacle of *explicit* error handling. Rust, too, requires
this - it introduces a `?` operator to signify propagating an error, but that,
still, is an explicit annotation. And apart from that, you can't use the return
value of a function that might propagate an error, without explicitly handling
that error first.

In Go, on the other hand, it is not only perfectly acceptable to ignore errors
when it makes sense (for example, I will always ignore errors created from
writing to a [`*bytes.Buffer`](https://godoc.org/bytes#Buffer.Write)), it is
actually often the only sensible thing to do. It is fundamentally not only
okay, but 99% of times *correct* to just completely ignore the error returned
by `fmt.Println`. And while it makes sense to check the error returned from
`json.Marshal` in your HTTP handler against `*json.MarshalError` (to
panic/log/complain loudly, because your code is buggy), any other errors
*should 99% of the time just be ignored*. And that's fine.

I believe that to say Go encourages explicit error handling, it would need some
mechanism of checked exceptions, Result types, or a requirement to pass an
[errcheck](https://github.com/kisielk/errcheck) like analysis in the compiler.

I think it would be closer to say, that Go encourages *local* error handling.
That is, the code that handles an error, is close to the code that produced it.
Exceptions encourages the two to be separated: There are usually several
or many lines of code in a single `try`-block, all of which share one
`catch`-block and it is hard to tell which of the lines produced it. And very
often, the actual error location is several stack frames deep. You could
contrast this with Go, where the error return is immediately obvious from the
code and if you have a line of error handling, it is usually immediately
attached to the function call that produced it.

However, that still seems to come short, in my view. After all, there is
nothing to force you to do that. And in fact, one of the most often [cited
articles about Go error handling](https://blog.golang.org/errors-are-values) is
often interpreted to encourage exactly that. Plus, a lot of people end up
writing `return err` far too often, simply propagating the error to be
*handled* elsewhere. And the proliferation of error-wrapping libraries happens
in the same vein: What their proponents phrase as "adding context to the error
value", I interpret as "adding back some of the information as a crutch, that
you removed when passing the error to non-local handling code". Sadly, far too
often, the error then ends up not being handled at all, as everyone just takes
advantage of that crutch. This leaves the end-user with an error message that is
essentially a poorly formatted, non-contiguous stacktrace.

Personally, I'd characterize Go's approach like this: In Go, error handling is
simply first-class code. By forcing you to use exactly the same control-flow
mechanisms and treat errors like any other data, Go encourages you to code your
error handling. Often that means a bunch of control flow to catch and recover
from any errors where they occur. But that's not "clutter", just as it is not
"clutter" to write `if n < 1 { return 1 }` when writing a Fibonacci function
(to choose a trivial example). It is just code. And yes, sometimes that code
might also store the error away or propagate it out-of-band to reduce
repetition *where it makes sense* - like in above blog post. But focussing on
the "happy path" is a bit of a distraction: Your *users* will definitely be
more happy about those parts of the control flow that make the errors disappear
or transform them into clean, actionable advise on how to solve the problem.

So, in my reading, the title of the Go blog post puts the emphasis in slightly
the wrong place - and often, people take the wrong message from it, in my
opinion. Not "errors are values", but "error handling is code".

---

So, what *would* be my advise for handling errors? To be honest, I don't know
yet - and I'm probably in no place to lecture anyone anyway.

Personally, I've been trying for the last couple of months to take a page out
of `http.Handler`s playbook and try, as much as possible, to completely avoid
returning an error. Instead of thinking "I should return an error here, in case
I ever do any operation that fails", I instead think "is there *any way at
all* I can get away with not returning an error here?". It doesn't always work
and sometimes you *do* have to pass errors around or wrap them. But I am
forcing myself to think very hard about handling my errors and it encourages a
programming-style of isolating failing components. The constraint of not being
able to return an error tends to make you creative in how to handle it.

---

[^1]: You might be tempted to suggest, that you could
define an `HTTPError`, containing the necessary info. Indeed, that's what the
[official Go blog](https://blog.golang.org/error-handling-and-go#TOC_3.) does,
so it can't be bad? And indeed, that *is* what they do, but note that they do
*not* actually return an `error` in the end - they return an `appError`, which
contains the necessary information. Exactly *because* they don't know how to
deal with general errors. So they translate any errors into a domain specific
type that carries the response. So, that is *not* the same as returning an
`error`.

I think *this* particular pattern is fine, though, personally, I don't
really see the point. Anything that builds an `appError` needs to provide
the complete response anyway, so you might as well just write it out
directly. YMMV.
