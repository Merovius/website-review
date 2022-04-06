---
date: "2017-06-18T22:57:21Z"
tags:
- golang
- programming
aliases:
- "/2017/06/16/how-not-to-use-an-http-router.html"
feedID: "https://blog.merovius.de//2017/06/18/how-not-to-use-an-http-router.html"
title: How to not use an http-router in go
summary: Trying to provide some advice on how to do easy, readable, scalable routing
  in go, without relying on any muxers/routers (or writing your own).
---

**If you don't write web-thingies in go you can stop reading now. Also, I am
somewhat snarky in this article. I intend that to be humorous but am probably
failing. Sorry for that**

As everyone™ knows, people need to [stop writing
routers/muxs](https://twitter.com/bketelsen/status/875435750770089984) in go.
Some people
[attribute](https://twitter.com/markbates/status/875517884931473409) the
abundance of routers to the fact that the `net/http` package fails to provide a
sufficiently powerful router, so people roll their own. This is also reflected
in [this post](https://medium.com/@joeybloggs/gos-std-net-http-is-all-you-need-right-1c5555a9f2f6),
in which a gopher complains about how complex and hard to maintain it would be
to route requests using `net/http` alone.

I disagree with both of these. I don't believe the problem is a lack of a
powerful enough router in the stdlib. I also disagree that routing based purely
on `net/http` has to be complicated or hard to maintain.

However, I *do* believe that the community currently lacks good guidance on
*how* to properly route requests using `net/http`. The default result seems to
be that people assume they are supposed to use `http.ServeMux` and get
frustrated by it. In this post I want to explain why routers *in general* -
including `http.ServeMux` - should be avoided and what I consider simple,
maintainable and scalable routing using nothing but the stdlib.

## But why?

![But why?](https://i.giphy.com/1M9fmo1WAFVK0.webp)

Why do I believe that routers should not be used? I have three arguments for
that: They need to be very complex to be useful, they introduce strong coupling
and they make it hard to understand how requests are flowing.

The basic idea of a router/mux is, that you have a single component which
looks at a request and decides what handler to dispatch it to. In your `func
main()` you then create your router, you define all your routes with all your
handlers and then you call `Serve(l, router)` and everything's peachy.

But since URLs can encode a lot of important information to base your routing
decisions on, doing it this way requires a lot of extra features. The [stdlib
ServeMux](https://godoc.org/net/http#ServeMux) is an incredibly simple router
but even that contains a certain amount of magic in its routing decisions;
depending on whether a pattern contains a trailing slash or not it might either
be matched as a prefix or as a complete URL and longer patterns take precedence
over shorter ones and oh my. But the stdlib router isn't even powerful enough.
Many people need to match URLs like `"/articles/{category}/{id:[0-9]+}"` in
their router and while we're at it also extract those nifty arguments. So
they're using [gorilla/mux](https://godoc.org/github.com/gorilla/mux) instead.
An awful lot of code to route requests.

Now, without cheating (and actually knowing that package counts as cheating),
tell me for each of these requests:

* `GET /foo`
* `GET /foo/bar`
* `GET /foo/baz`
* `POST /foo`
* `PUT /foo`
* `PUT /foo/bar`
* `POST /foo/123`

What handler they map to and what status code do they return ("OK"? "Bad
Request"? "Not Found"? "Method not allowed"?) in this routing setup?

```go
r := mux.NewRouter()
r.PathPrefix("/foo").Methods("GET").HandlerFunc(Foo)
r.PathPrefix("/foo/bar").Methods("GET").HandlerFunc(FooBar)
r.PathPrefix("/foo/{user:[a-z]+}").Methods("GET").HandlerFunc(FooUser)
r.PathPrefix("/foo").Methods("POST").HandlerFunc(PostFoo)
```

What if you permute the lines in the routing-setup?

You might guess correctly. You might not. There are multiple sane routing
strategies that you could base your guess on. The routes might be tried in
source order. The routes might be tried in order of specificity. Or a
complicated mixture of all of them. The router might realize that it could
match a Route if the method were different and return a 405. Or it might not not. Or that
`/foo/123` is, technically, an illegal argument, not a missing page. I couldn't
really find a good answer to any of these questions in the documentation of
`gorilla/mux` for what it's worth. Which meant that when my web app suddenly
didn't route requests correctly, I was stumped and needed to dive into code.

You could say that people just have to learn how `gorilla/mux` decides it's
routing (I believe it's "as defined in source order", by the way). But there
are at least fifteen thousand routers for go and no newcomer to your
application will ever know all of them. When a request does the wrong thing, I
don't want to have to debug your router first to find out what handler it is
actually going to and then debug that handler. I want to be able to follow the
request through your code, even if I have next to zero familiarity with it.

Lastly, this kind of setup requires that all the routing decisions for your
application are done in a central place. That introduces edit-contention, it
introduces strong coupling (the router needs to be aware of all the paths and
packages needed in the whole application) and it becomes unmaintainable after a
while. You can alleviate that by delegating to subrouters though; which really
is the basis of how I prefer to do all of this these days.

## How to use the stdlib to route

Let's build the toy example from [this medium post](https://medium.com/@joeybloggs/gos-std-net-http-is-all-you-need-right-1c5555a9f2f6).
It's not terribly complicated but it serves nicely to illustrate the general
idea. The author intended to show that using the stdlib for routing would be
too complicated and wouldn't scale. But my thesis is that the issue is that
*they are effectively trying to write a router*. They are trying to
encapsulate all the routing decisions into one single component. Instead,
separate concerns and make small, easily understandable routing decisions
locally.

Remember how I told you that we're going to use only the stdlib for routing?

![Those where lies, plain and simple](https://i.giphy.com/l4FGmlJviGJcYM2sM.webp)

We are going to use this one helper function:

```go
// ShiftPath splits off the first component of p, which will be cleaned of
// relative components before processing. head will never contain a slash and
// tail will always be a rooted path without trailing slash.
func ShiftPath(p string) (head, tail string) {
	p = path.Clean("/" + p)
	i := strings.Index(p[1:], "/") + 1
	if i <= 0 {
		return p[1:], "/"
	}
	return p[1:i], p[i:]
}
```

Let's build our app. We start by defining a handler type. The premise of this
approach is that handlers are strictly separated in their concerns. They either
correctly handle a request with the correct status code or they delegate to
another handler which will do that. They only need to know about the immediate
handlers they delegate to and they only need to know about the sub-path they
are rooted at:

```go
type App struct {
	// We could use http.Handler as a type here; using the specific type has
	// the advantage that static analysis tools can link directly from
	// h.UserHandler.ServeHTTP to the correct definition. The disadvantage is
	// that we have slightly stronger coupling. Do the tradeoff yourself.
	UserHandler *UserHandler
}

func (h *App) ServeHTTP(res http.ResponseWriter, req *http.Request) {
	var head string
	head, req.URL.Path = ShiftPath(req.URL.Path)
	if head == "user" {
		h.UserHandler.ServeHTTP(res, req)
		return
	}
	http.Error(res, "Not Found", http.StatusNotFound)
}

type UserHandler struct {
}

func (h *UserHandler) ServeHTTP(res http.ResponseWriter, req *http.Request) {
	var head string
	head, req.URL.Path = ShiftPath(req.URL.Path)
	id, err := strconv.Atoi(head)
	if err != nil {
		http.Error(res, fmt.Sprintf("Invalid user id %q", head), http.StatusBadRequest)
		return
	}
	switch req.Method {
	case "GET":
		h.handleGet(id)
	case "PUT":
		h.handlePut(id)
	default:
		http.Error(res, "Only GET and PUT are allowed", http.StatusMethodNotAllowed)
	}
}

func main() {
	a := &App{
		UserHandler: new(UserHandler),
	}
	http.ListenAndServe(":8000", a)
}
```

This seems very simple to me (not necessarily in "lines of code" but
definitely in "understandability"). You don't need to know anything about any
routers. If you want to understand how the request is routed you start by
looking at `main`. You see that `(*App).ServeHTTP` is used to serve any
request so you `:GoDef` to its definition. You see that it decides to dispatch
to `UserHandler`, you go to its `ServeHTTP` method and you see directly how it
parses the URL and what the decisions are that it made on its base.

We still need to add some patterns to our application. Let's add a profile
handler:

```go
type UserHandler struct{
	ProfileHandler *ProfileHandler
}

func (h *UserHandler) ServeHTTP(res http.ResponseWriter, req *http.Request) {
	var head string
	head, req.URL.Path = ShiftPath(req.URL.Path)
	id, err := strconv.Atoi(head)
	if err != nil {
		http.Error(res, fmt.Sprintf("Invalid user id %q", head), http.StatusBadRequest)
		return
	}

	if req.URL.Path != "/" {
		head, tail := ShiftPath(req.URL.Path)
		switch head {
		case "profile":
			// We can't just make ProfileHandler an http.Handler; it needs the
			// user id. Let's instead…
			h.ProfileHandler.Handler(id).ServeHTTP(res, req)
		case "account":
			// Left as an exercise to the reader.
		default:
			http.Error(res, "Not Found", http.StatusNotFound)
		}
		return
	}
	// As before
	...
}

type ProfileHandler struct {
}

func (h *ProfileHandler) Handler(id int) http.Handler {
	return http.HandlerFunc(func(res http.ResponseWriter, req *http.Request) {
		// Do whatever
	})
}
```

This may, again, seem complicated but it has the cool advantage that the
dependencies of `ProfileHandler` are clear at compile time. It needs a user id
which needs to come from *somewhere*. Providing it via this kind of method
ensures this is the case. When you refactor your code, you won't accidentally
forget to provide it; it's impossible to miss!

There are two potential alternatives to this if you prefer them: You could put
the user-id into `req.Context()` or you could be super-hackish and add them to
`req.Form`. But I prefer it this way.

You might argue that `App` still needs to know all the transitive dependencies
(because they are members, transitively) so we haven't actually reduced
coupling. But that's not true. Its `UserHandler` could be created by a
`NewUserHandler` function which gets passed its dependencies via the mechanism
of your choice (flags, dependency injection,…) and gets wired up in `main`. All
`App` needs to know is the API of the handlers it's *directly* invoking.

## Conclusion

I hope I convinced you that routers *in and of itself* are harmful. Pulling
the routing into one component means that that component needs to encapsulate
an awful lot of complexity, making it hard to debug. And as no single existing
router will contain all the complicated cleverness you want to base your
routing decisions on, you are tempted to write your own. Which everyone does.

Instead, split your routing decisions into small, independent chunks and
express them in their own handlers. And wire the dependencies up at compile
time, using the type system of go, and reduce coupling.
