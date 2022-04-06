---
date: "2013-10-11T03:25:09Z"
tags:
- linux
aliases:
- "/2013/10/11/inject-environment-variables-int.html"
omitFeed: true
title: Inject Environment variables into running processes
summary: Using gdb to manipulate a running process is fun and just the right amount of
  danger to be exiting
---

**tl;dr: Using gdb to manipulate a running process is fun and just the right
amount of danger to be exiting**

Just to document this (a friend asked me): If you ever wanted for example to
globally change your `$PATH`, or add a global `$LD_PRELOAD` (for example to use
[insulterr](https://github.com/Merovius/insulterr) ;) ), without restarting
your session, gdb is your friend.

You can call arbitrary functions in the context of any process (that you are
priviledged to attach a debugger, it has to run under your uid or you have to
be root, see `ptrace(2)` for specifics), as long as they are linked. Almost
everything is linked to `libld`, so with enough effort this actually means
*every* function.

For example, suppose you are running [i3wm](http://i3wm.org) and want to add
`/home/user/insulterr/insulterr.so` to your `$LD_PRELOAD` in every process
started by i3:

```
$ gdb -p `pidof i3` `which i3`
<lots of output of gdb>
gdb $ call setenv("LD_PRELOAD", "/home/user/insulterr/insulterr.so")
gdb $ quit
A debugging session is active.

	Inferior 1 [process 2] will be detached.

Quit anyway? (y or n) y
Detaching from program: /usr/bin/i3, process 2
```

This is of course a terrible hack, by high standards. Things to look out for
are (off the top of my head):

* You call a function that manipulates `errno` or does some other non-reentrent
  things. If you are attaching the debugger right in the middle of a library
  call (or immediately after) this *might* make the program unhappy because it
  does not detect an error (or falsely thinks there is an error).
* You call a function that does not work in a multithreaded context and another
  thread modifies it at the same time. Bad.
* You interrupt a badly checked `read(2)`/`write(2)`/`whatever(…)` call and a
  badly written program doesn't realize it got less data then expected (and/or
  crashes).  Shouldn't happen in practice, if it does, file a bug.
* You try to use symbols that are not available. This is actually not very bad
  and can be worked around (a friend of mine had the problem of needing `false`
  and just substituted 0).
* You use a daemon (like `urxvtd(1)`) for your terminals and the environment
  does not get passed correctly. This is also not very bad, just confusing.
  Attach your debugger to the daemon and change the environment there too.
* You attach the debugger to some process vital to the interaction with your
  debugger. Your window manager is a mild example. The terminal daemon is
  slightly worse (because, well, you can't actually type in the terminal window
  that your debugger is running in, ergo you can't stop it…), but you can
  change to a virtual terminal. Something like getty or init might be killing
  it.

Have fun!
