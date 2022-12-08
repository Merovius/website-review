---
date: "2022-12-08 12:05:00"
title: "Cleanly merge git repositories"
tags:
- "programming"
- "git"
- "advent-of-code"
summary: I describe how I did a clean, four-way merge of git repositories.
---

*Note: If you don't want to read the exposition and explanations and just want
to know the steps I did, scroll to the summary at the bottom.*

For a couple of years I have (with varying degrees of commitment) participated
in [Advent of Code](https://adventofcode.com/), a yearly programming
competition. It consists of fun little daily challenges. It is great to
exercise your coding muscles and can provide opportunity to learn new languages
and technologies.

So far I have created a separate repository for each year, with a directory
per day. But I decided that I'd prefer to have a single repository, containing
my solutions for *all* years. The main reason is that I tend to write little
helpers that I would like to re-use between years.

When merging the repositories it was important to me to preserve the history
of the individual years as well, though. I googled around for how to do this
and the solutions I found didn't *quite* work for me. So I thought I should
document my own solution, in case anyone finds it useful.

You can [see the result here](https://github.com/Merovius/AdventOfCode/network).
As you can see, there are four cleanly disjoint branches with separate
histories. They then merge into one single commit.

One neat effect of this is that the merged repository functions as a normal
remote for all the four old repositories. It involves no rewrites of history
and all the previous commits are preserved exactly as-is.  So you can just `git
pull` from this new repository and git will fast-forward the branch.

# Step 1: Prepare individual repositories

First I went through all repositories and prepared them. I wanted to have the
years in individual directories. In theory, it is possible to use
[git-filter-repo](https://git-scm.com/docs/git-filter-branch) and similar
tooling to automate this step. For larger projects this might be worth it.

I found it simpler to manually make the changes in the individual repositories
and commit them. In particular, I did not only need to move the files to the
sub directory, I also had to fix up Go module and import paths. Figuring out
how to automate that seemed like a chore. But doing it manually is a quick and
easy `sed` command.

You can see an example of that
[in this commit](https://github.com/Merovius/AdventOfCode/commit/8acd4ccd75c3e77cebacad98561bcf23a00e986e).
While that link points at the final, merged repository, I created the commit in
the old repository. You can see that a lot of files simply moved. But some also
had additional changes.

You can also see that I left the `go.mod` in the top-level directory. That was
intentional - I want the final repository to share a single module, so that's
where the `go.mod` belongs.

After this I was left with four repositories, each of which had all the
solutions in their own subdirectory, with a `go.mod`/`go.sum` file with the
shared module path. I tested that all solutions still compile and appeared to
work and moved on.

# Step 2: Prepare merged repository

The next step is to create a new repository which can reference commits and
objects in all the other repos. After all, it needs to contain the individual
histories. This is simple by setting the individual repositories as remotes:

```bash
$ mkdir ~/src/github.com/Merovius/AdventOfCode
$ cd ~/src/github.com/Merovius/AdventOfCode
$ git init
$ git remote add 2018 ~/src/github.com/Merovius/aoc18
$ git remote add 2020 ~/src/github.com/Merovius/aoc_2020
$ git remote add 2021 ~/src/github.com/Merovius/aoc_2021
$ git remote add 2022 ~/src/github.com/Merovius/aoc_2022
$ git fetch --multiple 2018 2020 2021 2022
$ git branch -a
remotes/2018/master
remotes/2020/main
remotes/2021/main
remotes/2022/main
```

One thing worth pointing out is that at this point, the merged `AdventOfCode`
repository *does not have any branches itself*. The only existing branches are
`remotes/` references. This is relevant because we don't want our resulting
histories to share any common ancestor. And because git behaves slightly
differently in an empty repository. A lot of commands operate on `HEAD` (the
“current branch”), so they have special handling if there is no `HEAD`.

# Step 3: Create merge commit

A git commit can have an arbitrary number of “parents”:

- If a commit has zero parents, it is the start of the history. This is what
  happens if you run `git commit` in a fresh repository.
- If a commit has exactly one parent, it is a regular commit. This is what
  happens when you run `git commit` normally.
- If a parent has more than one parent, it is a merge commit. This is what
  happens when you use `git merge` or merge a pull request in the web UI of a
  git hoster (like GitHub or Gitlab).

Normally merge commits have two parents - one that is the “main” branch and
one that is being “merged into”. However, git does not really distinguish
between “main” and “merged” branch. And it also allows a branch to have *more*
than two parents.

We want to create a new commit with four parents: The `HEAD`s of our four
individual repositories. I expected this to be simple, but:

```bash
$ git merge --allow-unrelated-histories remotes/2018/master remotes/2020/main remotes/2021/main remotes/2022/main
fatal: Can merge only exactly one commit into empty head
```

This command was supposed to create a merge commit with four parents. We have
to pass `--allow-unrelated-histories`, as git otherwise tries to find a common
ancestor between the parents and complains if it can't find any.

But the command is failing. It seems git is unhappy using `git merge` with
multiple parents if we do not have any branch yet.

I suspect the intended path at this point would be to check out one of the
branches and then merge the others into that. But that creates merge conflicts
and it also felt… asymmetric to me. I did not want to give any of the base
repositories preference. So instead I opted for a more brute-force approach:
Dropping down to the
[plumbing layer](https://git-scm.com/book/en/v2/Git-Internals-Plumbing-and-Porcelain).

First, I created the merged directory structure:

```bash
$ cp -r ~/src/github.com/Merovius/aoc18/* .
$ cp -r ~/src/github.com/Merovius/aoc_2020/* .
$ cp -r ~/src/github.com/Merovius/aoc_2021/* .
$ cp -r ~/src/github.com/Merovius/aoc_2022/* .
$ vim go.mod # fix up the merged list of dependencies
$ go mod tidy
$ git add .
```

*Note: The above does not copy hidden files (like `.gitignore`). If you do
copy hidden files, take care not to copy any `.git` directories.*

At this point the working directory contains the complete directory layout for
the merged commit and it is all in the staging area (or “index”). This is where
we normally run `git commit`. Instead we do the equivalent steps manually,
allowing us to override the exact contents:

```bash
$ TREE=$(git write-tree)
$ COMMIT=$(git commit-tree $TREE \
    -p remotes/2018/master \
    -p remotes/2020/main \
    -p remotes/2021/main \
    -p remotes/2022/main \
    -m "merge history of all years")
$ git branch main $COMMIT
```

The `write-tree` command takes the content of the index and writes it to a
[“Tree Object”](https://git-scm.com/book/en/v2/Git-Internals-Git-Objects#_tree_objects)
and then returns a reference to the Tree it has written.

A Tree is an immutable representation of a directory in git. It (essentially)
contains a list of file name and ID pairs, where each ID points either to a
“Blob” (an immutable file) or another Tree.

A Commit in git is just a Tree (describing the state of the files in the
repository at that commit), a list of parents, a commit message and some meta
data (like who created the commit and when).

The `commit-tree` command is a low-level command to create such a Commit
object. We give it the ID of the Tree the Commit should contain and a list of
parents (using `-p`) as well as a message (using `-m`). It then writes out that
Commit to storage and returns its ID.

At this point we have a well-formed Commit, but it is just loosely stored in
the repository. We still need a Branch to point at it, so it doesn't get lost
and we have a memorable handle.

You probably used the `git branch` command before. In the form above, it
creates a new branch `main` (remember: So far our repository had no branches)
pointing at the Commit we created.

And that's it. We can now treat the repository as a normal git repo. All that
is left is to publish it:

```bash
$ git remote add origin git@github.com:Merovius/AdventOfCode
$ git push --set-upstream origin main
```

# Executive Summary

To summarize the steps I did:

1. Create commits in each of the old repositories to move files around and
   fixing anticipated merge conflicts as needed.
2. Create a pristine new repository without any branches:
    ```bash
    $ git init merged
    $ cd merged
    ```
3. Add the old repositories as remotes for the merged repo:
    ```bash
    $ git remote add <repo1> /path/to/repo1
    $ git fetch repo1
    $ git remote add <repo2> /path/to/repo2
    $ git fetch repo2
    $ # …
    ```
4. Copy files from old repositories into merged repo:
    ```bash
    $ cp -r /path/to/repo1/* .
    $ cp -r /path/to/repo2/* .
    $ # …
    ```
5. Create commit using plumbing commands:
    ```bash
    $ git add .
    $ TREE=$(git write-tree)
    $ COMMIT=$(git commit-tree $TREE \
        -m "merge repositories" \
        -p remotes/repo1/main \
        -p remotes/repo2/main)
    $ git branch main $COMMIT
    ```
