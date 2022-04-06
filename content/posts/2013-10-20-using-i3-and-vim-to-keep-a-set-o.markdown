---
date: "2013-10-20T03:45:52Z"
tags:
- linux
aliases:
- "/2013/10/20/using-i3-and-vim-to-keep-a-set-o.html"
omitFeed: true
title: Using i3 and vim to keep a set of notes at hand
summary: Put a terminal with a vim-instance in an i3-scratchpad, combine it with autosave-when-idle
  and you got the perfect note keeping workflow."
---

**tl;dr: Put a terminal with a vim-instance in an i3-scratchpad, combine it
with autosave-when-idle and you got the perfect note keeping workflow**

There are often occasions where I want to write something down, while not
wanting to disturb my thought-process too much or taking too much of an effort.
An example for the former would be a short TODO I suddenly remember while doing
something more important. As an example for the latter, I keep an "account" for
drinks at our local computer club, so that I don't always have to put single
coins into the register, but can just put 20€ or something in and don't have to
worry about it for a while. Combining the
[scratchpad-window](http://i3wm.org/docs/userguide.html#_scratchpad) feature of
i3 with a little vim-magic makes this effortless enough to be actually
preferable to just paying.

First of, you should map a key to `scratchpad show` in i3, for example I have
the following in my config:

```
bind Mod4+Shift+21 move scratchpad
bind Mod4+21 scratchpad show
```

I can then just use `Mod4+<backtic>` to access the scratchpad.

Now, just put a terminal in scratchpad-mode and open .notes in vim in this
terminal. By pressing the `scratchpad show` binding repeatedly, you can send it
to the background and bring it to the foreground again.

I have my current "balance" in this notes-file and during the meetings of the
computer club leave the cursor on this balance. If I take a drink, I press `^X`
decreasing my balance by one (every drink is one Euro). If I pay, say 10 Euros
into the register, I press `10^A` increasing my balance by 10.

This is already much better, but it still has one problem: I better save that
file every time I change my balance, else a crash would screw up my accounting.
Luckily, vim provides autocommands and has an event for "the user did not type
for a while". This means, that we can automatically save the file if we idled
for a few seconds, for example if we send the scratchpad window away. For this,
we put the following in our `.vimrc`:

```
" Automatically save the file notes when idle
autocmd CursorHold .notes :write
```

Now adjusting my balance is just a matter of a very short key sequence:
``<mod4>`<c-x><mod4>` ``
