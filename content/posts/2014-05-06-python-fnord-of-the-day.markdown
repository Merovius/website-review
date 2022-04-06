---
date: "2014-05-06T01:07:28Z"
tags:
- programming
- python
aliases:
- "/2014/05/06/python-fnord-of-the-day.html"
feedID: "https://blog.merovius.de//2014/05/06/python-fnord-of-the-day.html"
title: Python-fnord of the day
summary: An unexpected behavior of python that surprised me today.
---

This is an argument of belief, but I think this is highly irregular and
unexpected behaviour of python:

```python
a = [1, 2, 3, 4]
b = a
a = a + [5]
print(b)
a = [1, 2, 3, 4]
b = a
a += [5]
print(b)
```

Output:

```
[1, 2, 3, 4]
[1, 2, 3, 4, 5]
```

Call me crazy, but in my world, `x += y` should behave exactly the same as `x =
x + y` and this is another example, why operator overloading can be abused in
absolutely horrible ways.

Never mind, that there is actually python [teaching
material](http://www.tutorialspoint.com/python/python_basic_operators.htm) [out
there](http://www.rafekettler.com/magicmethods.html#numeric) that teaches wrong
things. That is, there are actually people out there who think they know python
well enough to teach it, but don't know this. Though credit where credit is
due, the [official documentation](https://docs.python.org/2/reference/simple_stmts.html#augmented-assignment-statements)
mentions this behaviour.
