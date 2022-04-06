---
date: "2017-09-12T11:05:00Z"
tags:
- golang
- programming
aliases:
- "/2017/09/12/diminishing-returns-of-static-typing.html"
omitFeed: true
title: Diminishing returns of static typing
summary: When talking about static type systems, we often tend to focus on one side of
  the equation. I'm trying to make explicit how I view the question as a tradeoff
  and why I neither agree with “more is always better”, nor with “a little is enough”.
---

I often get into discussions with people, where the matter of strictness and
expressiveness of a static type system comes up. The most common one, by far,
is Go's lack of generics and the resulting necessity to use `interface{}` in
container types (the [container-subpackages](https://godoc.org/container) are
obvious cases, but also [context](https://godoc.org/context)). When I express
my view, that the lack of static type-safety for containers isn't a problem, I
am treated with condescending reactions ranging from disbelief to patronizing.

I also often take the *other* side of the argument. This happens commonly, when
talking to proponents of dynamically typed languages. In particular I got into
debates of whether Python would be suitable for a certain use-case. When the
lack of static type-safety is brought up, the proponents of Python defend it by
pointing out that it now features optional type hints. Which they say make it
possible, to reap the benefits of static typing even in a conventionally
dynamically typed language.

This is an attempt to write my thoughts on both of these (though they are not
in any way novel or creative) down more thoroughly. Discussions usually don't
provide the space for that. They are also often charged and parties are more
interested in “winning the argument”, than finding consensus.

---

I don't think it's particularly controversial, that static typing in general
has advantages, even though actual data about those seems to be [surprisingly
hard to come by](https://danluu.com/empirical-pl/). *I* certainly believe that,
it is why I use Go in the first place. There is a difference of opinion though,
in how large and important those benefits are and how much of the behavior of a
program must be statically checked to reap those benefits.

To understand this, we should first make explicit *what* the benefits of static
type checking are. The most commonly mentioned one is to catch bugs as early in
the development process as possible. If a piece of code I write already
contains a rigorous proof of correctness in the form of types, just writing it
down and compiling it gives me assurance that it will work as intended in all
circumstances. At the other end of the spectrum, in a fully dynamic language I
will need to write tests exercising all of my code to find bugs. Running tests
takes time. Writing *good* tests that actually cover all intended behavior is
hard. And as it's in general impossible to cover *all* possible execution
paths, there will always be the possibility of a rare edge-case that we didn't
think of testing to trigger a bug in production.

So, we can think of static typing as increasing the proportion of bug-free
lines of code deployed to production. This is of course a simplification. In
practice, we would still catch a lot of the bugs via more rigorous testing,
QA, canarying and other practices. To a degree we can still subsume these in
this simplification though. If we catch a buggy line of code in QA or the
canary phase, we are going to roll it back. So in a sense, the proportion of
code we wrote that makes it as bug-free into production will still go down.
Thus:

<img class="small" src="/assets/static_typing_v_good_code.png">

This is usually the understanding, that the “more static typing is always
better” argument is based on. Checking more behavior at compile time means less
bugs in production means more satisfied customers and less being woken up at
night by your pager. Everybody's happy.

Why then is it, that we don't all code in Idris, Agda or a similarly strict
language? Sure, the graph above is suggestively drawn to taper off, but it's
still monotonically increasing. You'd think that this implies more is better.
The answer, of course, is that static typing has a cost and that there is no
free lunch.

The costs of static typing again come in many forms. It requires more upfront
investment in thinking about the correct types. It increases compile times and
thus the change-compile-test-repeat cycle. It makes for a steeper learning
curve. And more often than we like to admit, the error messages a compiler will
give us will decline in usefulness as the power of a type system increases.
Again, we can oversimplify and subsume these effects in saying that it reduces
our speed:

<img class="small" src="/assets/static_typing_v_speed.png">

This is what we mean when we talk about dynamically typed languages being good
for rapid prototyping. In the end, however, what we are usually interested in,
is what I'd like to call *velocity*: The speed with which we can deploy new
features to our users. We can model that as the speed with which we can roll
out bug-free code.  Graphically, that is expressed as the product of the
previous two graphs:

<img class="small" src="/assets/static_typing_v_velocity.png">

In practice, the product of these two functions will have a maximum, a sweet
spot of maximum velocity. Designing a type system for a programming language
is, at least in part, about finding that sweet spot[^1].

Now if we are to accept all of this, that opens up a different question: If we
are indeed searching for that sweet spot, how do we explain the vast
differences in strength of type systems that we use in practice? The answer of
course is simple (and I'm sure many of you have already typed it up in an angry
response). The curves I drew above are completely made up. Given how hard it is
to do empirical research in this space and to actually quantify the measures I
used here, it stands to reason that their shape is very much up for
interpretation.

A Python developer might very reasonably believe that optional type-annotations
are more than enough to achieve most if not all the advantages of static
typing. While a Haskell developer might be much better adapted to static typing
and not be slowed down by it as much (or even at all). As a result, the
perceived sweet spot can vary widely:

<img src="/assets/static_typing_pythonista_v_haskeller.png">

What's more, the importance of these factors might vary a lot too. If you are
writing avionics code or are programming the control unit for a space craft,
you probably want to be pretty darn sure that the code you are deploying is
correct. On the other hand, if you are a Silicon Valley startup in your
growth-phase, user acquisition will be of a very high priority and you get
users by deploying features quicker than your competitors. We can model that,
by weighing the factors differently:

<img src="/assets/static_typing_startup_v_nasa.png">

Your use case will determine the sweet spot you are looking for and thus the
language you will choose. But a language is also designed with a set of use
cases in mind and will set its own sweet spot according to that.

I think when we talk about how strict a type system should be, we need to
acknowledge these subjective factors. And it is fine to believe that your
perception of one of those curves or how they should be weighted is closer to
a hypothetical objective reality than another persons. But you should make that
belief explicit and provide a justification of *why* your perception is more
realistic. Don't just assume that other people view them the same way and then
be confused that they do not come to the same conclusions as you.

---

Back to Go's type system. In my opinion, Go manages to hit a good sweet spot
(that is, its design agrees with my personal preferences on this). To me it
seems that Go reaps probably upwards of 90% of the benefits you can get from
static typing while still being not too impeding. And while I definitely agree
static typing is beneficial, the *marginal* benefit of making user-defined
containers type-safe simply seems pretty low (even if it's positive). In the
end, it would probably be less than 1% of Go code that would get this additional
type-checking and it is probably pretty obvious code. And meanwhile, I perceive
generics as a language feature pretty costly. So I find it hard to justify a
large perceived cost with a small perceived benefit.

Now, that is not to say I'm not open to be convinced. Just that simply saying
“but more type-safety!” is only looking at one side of the equation and isn't
enough. You need to acknowledge that there is no free lunch and that this is a
tradeoff. You need to accept that your perceptions of how big the benefit of
adding static typing is, how much it costs and how important it is are all
subjective. If you want to convince me that my perception of their benefit is
wrong, the best way would be to provide specific instances of bugs or
production crashes caused by a type-assertion on an `interface{}` taken out of
a container. Or a refactoring you couldn't make because of the lack of
type-safety with a specific container. Ideally, this takes the form of an
[experience report](https://github.com/golang/go/wiki/ExperienceReports), which
I consider an excellent way to talk about engineered tradeoffs.

Of course you can continue to roll your eyes whenever someone questions your
perception of the value-curve of static typing. Or pretend that when I say the
*marginal* benefit of type-safe containers is small, I am implying that the
*total* benefit of static typing is small. It's an effective debate-tactic, if
your goal is to shut up your opposition. But not if your goal is to convince
them and build consensus.

---

[^1]: There is a generous and broad exception for research
languages here. If the point of your design is to explore the possibility space
of type-systems, matters of practicality can of course often be ignored.
