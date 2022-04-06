---
date: "2013-10-11T02:09:38Z"
tags:
- C
- programming
aliases:
- "/2013/10/11/how-to-cgolf.html"
feedID: "https://blog.merovius.de//2013/10/11/how-to-cgolf.html"
title: How to C-Golf
summary: We had a codegolf challenge recently. My C-solution was 246 byte, the perl-winner
  was 191. I decided to give notes for C-golf beginners.
---

**tl;dr: We had a codegolf challenge recently. My C-solution was 246 byte, the
perl-winner was 191. I decided to give notes for C-golf beginners**

**Note:** Most of this blog post is incredibly boring. A better way than to
read through it is to just skip through the git-repository and refer to the
explanations here everytime you don't know why a change works or what it
does. To make this easier, I added ankers to every paragraph, named by the
commit of the change it explains. So if you want to know, how a specific change
works, you can add `#commitid` to the url of this post, with `commitid` being
the full id of that change. If you want to read the more interesting
stuff, [from about here](#4272ca2a181e8f50c1645b793c7a1338f9ff1502) it starts
to get non-obvious I think.

At the [rgb2rv10](http://rgb2r.noname-ev.de/) we again had a
[codegolf](https://en.wikipedia.org/wiki/Code_golf) event. C is generally not a
preferred language for golf, but I use it, because I know it best and my
experiences with it are not that bad. My C solutions are most of the times
longer then the shortest solutions in perl or similar languages, but they are
competitive. That's why I decided to make a blogpost explaining this years C
solution as an example to show some basic C-golfing techniques.

This years [challenge](https://www.noname-ev.de/w/Codegolf/RGB2Rv10) was to
implement an interpreter for esocalc, a two-dimensional language for arithmetic
expressions. Follow the link for a more detailed specification. This challenge
is primed to be solved in C. This is also reflected by the length of the
different solutions: The shortest C solution is 246 bytes, the shortest Python
solution is 227 and the shortest Perl solution is 191 bytes. For a
codegolf-challenge this is an impressively small gap between C and scripting
languages.

You can follow this post by checking out [the code](https://github.com/Merovius/cgolf)
on github. The oldest commit is the one we are starting with and we will refine
it until we reach the 246 byte solution in `master`. You can test it by
compiling it (`gcc -o golf golf.c` should suffice in most cases, the shortest
needs a longer commandline, which is put in the Makefile, so you should `make`
it). You can run it through the testsuite used in the contest by running
`GOLF_BIN="./golf prove -l"`.

<a id="e3dc46c7c88f740c6b4eb671cd3b987061797529"></a>
The first step is to implement an easily readable, working version. This is
done in the
[first commit](https://github.com/Merovius/cgolf/blob/e3dc46c7c88f740c6b4eb671cd3b987061797529/golf.c).
Though you yourself might have come up with a different implementation, this is
pretty straightforward I think. We just read the whole esocalc-sourcecode and
walk through it, executing every instruction as we go. The stack is statically
allocated and of a fixed size, but that's no problem because we only have a
limited testsuite anyway.

<a id="38e6ffceb633615f48d0a9d25a391abf5228c35c"></a>
The [next step](https://github.com/Merovius/cgolf/blob/38e6ffceb633615f48d0a9d25a391abf5228c35c/golf.c)
is obvious: We remove comments and move to one-letter variable names, thus
reducing readability, but also size considerably. We will leave most of the
whitespace for now, because else it is hard to follow the changes.

<a id="004b45da976b3d1aab23e1b5ed3b9ff87b002895"></a>
An important lesson for C-golfers is the following: *for is never longer then
while and most of the times shorter*. An endless loop with `while` takes one
character more then a `for`-loop. We will later see more instances when `for` will
be considerably shorter. Also, we see the `if`/`else`-constructs in the
control-flow instructions. It is considerably shorter to use a ternary operator
in most cases, because in C, most statements are also expressions, so we can
write them as cases in `?:` - or use the short-circuiting `&&` if there is no
`else`-part. We will see more of that later. Lastly we collapse multiple
variable declarations into one to save `int`-keywords. These three changes are
what happend in [the next version](https://github.com/Merovius/cgolf/blob/004b45da976b3d1aab23e1b5ed3b9ff87b002895/golf.c).

<a id="eb5227716869399d62f12dcfc07c7e42094782b7"></a>
We continue in our path and notice, that we every `char`-literal takes three
bytes, while the number it represents often only takes two in decimal.
[Let's fix that](https://github.com/Merovius/cgolf/blob/eb5227716869399d62f12dcfc07c7e42094782b7/golf.c).

<a id="75625a730875ded009a216887db5455b5105e7e6"></a>
We also have two temporary variables `a` and `b`, that we shouldn't need.
[We can get rid of them](https://github.com/Merovius/cgolf/blob/75625a730875ded009a216887db5455b5105e7e6/golf.c),
by thinking up a single statement for arithmetic operations.

<a id="f0af3799d6c5ee3c30a1f43dd5c89523f2619759"></a>
[The next step](https://github.com/Merovius/cgolf/blob/f0af3799d6c5ee3c30a1f43dd5c89523f2619759/golf.c)
uses a real detail of C: If you don't give a type for a global variable, a
parameter or the return type of a function, `int` is assumed. If a function is
not defined, a prototype of `int foo()` is assumed, meaning we can pass
arbitrary arguments and get an `int`. The libc is linked in by default. All
these facts means, we can drop all `include`s and put our variables in the
global scope to remove all `int`-keywords. This is a very basic, but very
usefull technique. It has one important caveat, you should look out for: If you
need the return value of a libc-function and it is *not* `int`, you should
think about wether it can be safely converted. For example on amd64 an `int`
has 32 bits, but a pointer has 64 bits, therefore pointers as return values get
truncated (even if you assign them to a pointer).

<a id="17f305a0091651c03bb9e86e6ee9332f72138c04"></a>
[We can save more](https://github.com/Merovius/cgolf/blob/17f305a0091651c03bb9e86e6ee9332f72138c04/golf.c)
by using a parameter to `main`. This is also a very basic and often seen trick
in C-golfing. You get up to 3 local variables for free this way. In our case
there is an additional benefit: The first parameter to `main` is the number of
arguments, which is 1 for a normal call (the first argument is the name with
which the programm was called). This means, we get the initialization to 1 for
free.

<a id="f3957253031431ec25f8d4f68c10ca1b4dcfd4ed"></a>
[A trivial optimization](https://github.com/Merovius/cgolf/blob/f3957253031431ec25f8d4f68c10ca1b4dcfd4ed/golf.c)
is using `gets` instead of `read`. `gets` always adds a terminating zero-byte,
so we need to grow our buffer a little bit.

<a id="https://github.com/Merovius/cgolf/blob/fed1a817b88072dc5d27d8ae4dc772da8518ee5d"></a>
If we now look at our code, all the `case`-keywords might annoy us. If we see
a lot of repititions in our code, the obvious tool to use in C are `define`s. So
[lets define](https://github.com/Merovius/cgolf/blob/fed1a817b88072dc5d27d8ae4dc772da8518ee5d/golf.c)
the structure of the cases and replace every case by a short 1-letter identifier.

<a id="9de0b6f05fc52e5c08829bcf6d60a83c6756fba2"></a>
The same goes for the arithmetic operations: Four times the same long code cries
for a [define](https://github.com/Merovius/cgolf/blob/9de0b6f05fc52e5c08829bcf6d60a83c6756fba2/golf.c).
A `define` is not always a good solution. You have to weigh the additional
overhead of the keyword and the needed newline against the savings and number
of repititions.

<a id="ec654b1a11012a7820807cd29fe65a6427f300d4"></a>
[Next](https://github.com/Merovius/cgolf/blob/ec654b1a11012a7820807cd29fe65a6427f300d4/golf.c)
we eliminate the variable `i`. Skilled C-coders use pointer-arithmetic quite
often (no matter how bad the reputation is). In this case it would be a bad
idea, if we were not explicitely allowed to assume that all programs are
correct and stay in the bounds given (because bound-checks are a lot harder
without indexing).

<a id="6a10cb1480e1ca6cdc61bd628d8cb2f4d365a699"></a>
Another example of savings by `for`-loops is
[the next change](https://github.com/Merovius/cgolf/blob/6a10cb1480e1ca6cdc61bd628d8cb2f4d365a699/golf.c).
Here we moved two statements into the `for`-loop, thus using the semicolons we
need there anyway and saving two bytes.

<a id="7d506e18324daf3d6d98e25682321c19c7bef781"></a>
So the next big thing that catches our eyes are the `switch`, `case` and
`break`-keywords. Everytime you see long identifiers or keywords you should
think about wether a different program-structure or a different libc-builtin
may help you save it. `switch`-construct can almost always be replaced by an
`if`-`else if` construct (which is why we learned to use `switch` anyway). This
is often shorter, but as we learned, the ternary operator is even shorter. So in
[the next step](https://github.com/Merovius/cgolf/blob/7d506e18324daf3d6d98e25682321c19c7bef781/golf.c)
we use a giant ternary expression instead of a `switch`-structure. This brings
one major problem: `return` is one of the few things that's a statement, but
not an expression. So we can't use it in `?:`-expressions (because the branches
have to be expressions). We use `exit()` instead, which is an expression, but a
`void`-expression, so again we run into problems using it in `?:`. We work
around that for now by using `(exit(0),1)` instead. If you connect expressions
by `,` they are evaluated in succession (contrary to using boolean operators
for example) and the value of the last one is becoming the value of the whole
expression - so our `exit`-expression evaluates to 1 in this case.

<a id="4272ca2a181e8f50c1645b793c7a1338f9ff1502"></a>
`exit` is still pretty long (especially with the added parens and
comma-expression), so we want to avoid it too. Here comes a notable quote of
the organisator of the competition into action: “The return value isn't
important, as long as the output is correct. So it doesn't matter if you
segfault or anything”. This is the key to
[the next change](https://github.com/Merovius/cgolf/blob/4272ca2a181e8f50c1645b793c7a1338f9ff1502/golf.c):
Instead of exiting orderly we just create the conditions for a segfault by
assigning zero to `p`, which is dereferenced shortly thereafter, thus creating
a segfault when we want to exit. This is one of my favourite optimizations.

<a id="bb1b73fdfd4be6a75ebc47046af7b9af06ff80fe"></a>
There still is some repitition in our code. We still assign to `d` more often
then not. But our big nested ternary operator doesn't return anything yet. So our
[next step](https://github.com/Merovius/cgolf/blob/bb1b73fdfd4be6a75ebc47046af7b9af06ff80fe/golf.c)
is to return the new value for `d` in all subexpressions (if need be by using a
comma). This does not save a lot, but still a few bytes.

<a id="309465a985f67a8326ab10347b568ef467362b1c"></a>
Now the sources of bytes to save are getting scarcer. What still is a pain is
the explicit stack of a fixed size. Here another deep mistery of C (or more
specifically the way modern computers work)  comes into play:
[The call stack](https://en.wikipedia.org/wiki/Call_stack). We can actually
[use this as our stack](https://github.com/Merovius/cgolf/blob/309465a985f67a8326ab10347b568ef467362b1c/golf.c).
The way this works is, that we use a pointer to an address in the memory area,
the operating system reserved for our call stack and grow down (contrary to the
illustration on wikipedia, the stack grows downwards. But this is a minor
detail). By writing to this pointer and decrementing, we can push to the stack.
By incrementing it and reading we can pop something from the top of the stack.
To get a valid stack-address we could use the address of a local variable (for
example `s` itself). Local variables are at the bottom of the stackframe, so we
do not overwrite anything important if growing down. There is however a
problem: We call `gets` and `printf` which push a few stackframes to the
callstack. Our stack would get smashed by these calls. Therefore we just
subtract a sufficiently high number from it to reserve space for the
stackframes of the function calls. 760 is the minimum amount needed in my
setup, everything up to 99999 should save at least one byte.

<a id="00afa97fb52ba275f638092118b49b4027261928"></a>
This still is unsatisfactory, so we will hack a little more and use the fact,
that the testsuite only uses quite small programms and a quite small stack is
needed. So we just
[use `s` unitialized](https://github.com/Merovius/cgolf/blob/00afa97fb52ba275f638092118b49b4027261928/golf.c),
which is absolutely crazy. I discovered (by accident), that you will always end
up with a pointer to your program-array, using around 200 bytes of the end
(most probably some earlier deeply nested call in the startup of the binary
will write an appropriate address here by accident). This of course is
borderline cheating, but it saves 6 bytes, so who cares. From now on it's
absolutely forbidden to compile with optimizations, because this will destroy
this coincidence. Oh well.

<a id="e2aafeb23a88abb731d0341610bc84acd285424d"></a>
So, if we are already doing unreliable horrific voodoo which will curl up the
fingernails of every honest C developer, we can also
[save two bytes](https://github.com/Merovius/cgolf/blob/e2aafeb23a88abb731d0341610bc84acd285424d/golf.c)
by not setting `p` to zero, but instead just doubling it. You will then end up
with *some* address, that is hard to predict, but in all cases I tried leads to
crashing just as reliable. This means, we exit our program in just one byte. Neato!

<a id="7b1803ce9fe52c0f57fb804067493bc975dfb3be"></a>
There is not a lot we can save left now. What might still annoy us and is a
very good tip in general are all this numbers. Even if most characters have
only 2 bytes as a decimal, they still only have one byte as a character (not a
`char`-literal!). We can
[fix this](https://github.com/Merovius/cgolf/blob/7b1803ce9fe52c0f57fb804067493bc975dfb3be/golf.c)
by passing a verbatim character as the first argument to the `c`-makro. To
interpret it as a `char`, we stringify it (with `#a`) and dereference it (with
`*#a`), getting the first `char`. This opens a problem: A space is a
significant character in the interpreted source code, so we need to use it as
an argument. But a space is not significant at that point in the C source code,
so we simply can not pass it to our makro. The solution to this is to move the
whole ASCII-table. So instead of comparing `*p` we compare `*p+n` with `n` to
be choosen. Thus we don't need to pass a space, but some other char, that is
`n` positions away and everyone is happy. Kind of. We also need to avoid single
quotes, double quotes (though we can avoid this by using emtpy string (think
about why this works), but too many bytes!!!), parenthesis and chars outside of
ASCII (because this will break our C-file). These constrictions make `n=3`
pretty much the only choice. This means, we have to include a DEL-character in
our source-code, but the compiler is quite happy about that (the wiki isn't,
github isn't, the editor isn't, but who cares). This is my second most favourite hack.

<a id="60a5912baccb94e3e31cc57fe09712b1e7cb0280"></a><a id="70da40d21ca8ff3a58e5d2a3a890ff0f44d2ee0c"></a>
Now there is not much left to do. We
[remove the last char-literal left](https://github.com/Merovius/cgolf/blob/60a5912baccb94e3e31cc57fe09712b1e7cb0280/golf.c) and
[remove all non-essential whitespace](https://github.com/Merovius/cgolf/blob/70da40d21ca8ff3a58e5d2a3a890ff0f44d2ee0c/golf.c).

<a id="09ff6c236827639aad31edec198e97748241c3ea"></a>
This leaves us with 253 bytes. To get below 250, we
[use buildflags](https://github.com/Merovius/cgolf/blob/09ff6c236827639aad31edec198e97748241c3ea/Makefile)
instead of
[defines](https://github.com/Merovius/cgolf/blob/09ff6c236827639aad31edec198e97748241c3ea/golf.c).
Usually such flags are counted by the difference they add to a minimal compiler
call needed. In this case, we have a 186 byte C-file (after removing the
trailing newline added by vim) and 60 bytes of compiler-flags, totalling 246
bytes.

I think there still is potential to remove some more characters. Other tools
not used here include
[dispatch tables](https://en.wikipedia.org/wiki/Dispatch_table)
(which are kind of hard in C, because it lacks an eval, but some variations of
the concept still apply) and magic formulas. If the testcases are very limited,
some people resort to hardcoding the wanted results and just golf a minimal way
to differentiate between what output is wanted. This might be surprising, but
in many cases (this included) this will end up being shorter (though I consider
it cheating and try to avoid it). We also didn't do a lot of
[bit banging](https://en.wikipedia.org/wiki/Bit_banging). For example using `^`
instead of `==` reverses the check but saves a byte. But I think it is a
usefull intro for people who are just learning C and want to dive deeper into
the language by golfing.
