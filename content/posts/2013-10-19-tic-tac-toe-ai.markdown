---
date: "2013-10-19T01:58:04Z"
tags:
- programming
aliases:
- "/2013/10/19/tic-tac-toe-ai.html"
omitFeed: true
title: Tic Tac Toe AI
summary: I wrote a simple Tic Tac Toe AI as an exercise. You can get it on github.
---

**tl;dr: I wrote a simple Tic Tac Toe AI as an exercise. You can get it on
[github](https://github.com/Merovius/tictactoe)**

I am currently considering writing a basic chess AI as an exercise in AI
development and to help me analyze my own games (and hopefully get a better
chess-player just by thinking about how a machine would do it). As a small
exercise and to get some familiarity with the algorithms involved, I started
with [Tic Tac Toe](https://en.wikipedia.org/wiki/Tic_tac_toe). Because of the
limited number of games (only [255168](http://www.se16.info/hgb/tictactoe.htm))
all positions can be bruteforced very fast, which makes it an excellent
exercise, because even with a very simple
[Minimax-Algorithm](https://en.wikipedia.org/wiki/Minimax#Minimax_algorithm_with_alternate_moves)
perfect play is possible.

[My AI](https://github.com/Merovius/tictactoe) uses exactly this algorithm (if
coded a little crude). It comes with a little TUI and a small testsuite, you
can try it like this:

```sh
$ git clone git://github.com/Merovius/tictactoe.git
$ cd tictactoe
$ make
$ make test
$ ./tictactoe
```

You will notice, that there already is no noticable delay (at least not on a
relatively modern machine), even though the AI is unoptimized and bruteforces
the whole tree of possible moves on every move.

Next I will first refactor the basic algorithm in use now, then I will probably
implement better techniques, such as limited search-depth,
[αβ-Pruning](https://en.wikipedia.org/wiki/Alpha-beta_pruning) or machine
learning. I will then think about moving on to a little more complex games (for
example Connect 4, Mill or Hex seem good choices). Then I will decide how big
the effort would be for chess and if it's worth a try.
