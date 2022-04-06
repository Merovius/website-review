---
date: "2013-09-28T03:16:52Z"
tags:
- education
- programming
aliases:
- "/2013/09/28/firstyear-introductory-course-fo.html"
feedID: "https://blog.merovius.de//2013/09/28/firstyear-introductory-course-fo.html"
title: First-year introductory course for programming
summary: I gave a very introductory programming course and saw once again how the basic
  ideas underlying the modernization of teaching just work when implemented right.
---

**tl;dr: I gave a very introductory programming course and saw once again how
the basic ideas underlying the modernization of teaching *just work* when
implemented right.**

This last week I organized a (very basic) introductory course on programming for
our first-year students. I was set on C++ because it is the language used in the
introductory lecture and we wanted to give people with absolutely no background
in programming or proper use of a computer the necessary tools to start in this
lecture on mostly equal grounds to people who already took a basic computer
science course in school. We had five days, with 3 hours a day to try to reach
that goal, which is a very limited amount of time for such a course and we had
50 participants.

The whole concept of the course was very modern (at least for our universities
standards) - instead of just giving lectures, telling people about syntax and
stuff we divided up the whole course into 19 lessons, each of which was worked
at mostly independent. That had two big advantages (and was very positively
perceived): First, the amount of time, we needed to spend lecturing and doing
frontal presentations was minimized to about half an hour over all the course.
The saved time could be invested in individual tutoring. This enabled us to
react to every student needing help in a few seconds, using only about 3-4
senior students (with mostly pretty minimal background themselves actually) to
teach.

Second the students where able to just work in their own speed without external
pressure or a limit on the time spent on any lesson. Missing deadlines for
lessons meant more experimentation, less competition amongst the students, less
stress and less pressure to finish with all lessons in time. The course was not
designed to be finished, so even though many students didn't reach the last
lesson, I think the additional experimentation (combined with a less
content-driven curriculum) added much more value for the students.

The content also was rather different from what you usually read in tutorials or
get in lectures at the university. Instead of systematically developing syntax
and different language constructs, we used the language less then the object to
learn, but the mean to learn basic skills needed, when tackling a programming
lecture (basically: „How do I start“ and „what can I do, if it doesn't work?“).
We introduced every lesson with about a page of text, describing the key
constructs underlying the object of that lesson, gave some basic code-examples
and (without explaining the details of the syntax) then presented some basic
exercises, which could be mastered without much understanding of what was
happening, but which ensured the reproduction needed, to properly learn the
syntactic device or the idea. We then added some playfull, very open exercises,
where through experimentation and through their own mistakes the students where
supposed to discover themselves the more intricate details of the subject
matter. Thematically we restricted the syntax to the absolute minimum to get
some basic, but fun and usefull programms to work (for example, we introduced
only one kind of loop, and we introduced only the datatypes int, bool,
std::string and double, as well as arrays thereof)

Though this all might sound fairly „new-agey“, it worked remarkably well. We saw
a fair amount of experimentation, we saw very creative solutions to seemingly
easy and straightforward, we got very positive feedback and though we introduced
many special subjects (for example debuggers, online references and detailed
lectures and exercises on how to read manpages or error output of the compiler),
I think it is fair to say, that we reached at least the level of proficiency and
confidence as the more traditional courses we held the last years had.

So, the bottomline is: We took a very huge bite out of the ideas and thoughts
underlying the ongoing effort in europe to modernize teaching at universities
(The „Bologna Process“, as it's known at least here in germany) and though I
totally agree, that the implementation of these guidelines at the universities
is currently pretty misguided and plain *bad*, I once again feel confirmed in my
view, that if you put some effort into it and really use what the underlying
ideas of bologna are (instead of just picking up, what you hear from the media
about it), you can create a really kick-ass curriculum, that is both more fun
*and* more informative at the same time.

All used content is on
[github](https://github.com/FachschaftMathPhys/Infovorkurs), if you are
interested in what exactly we used in the course.
