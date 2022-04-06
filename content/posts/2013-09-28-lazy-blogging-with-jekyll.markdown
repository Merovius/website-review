---
date: "2013-09-28T15:11:10Z"
tags:
- meta
- programming
aliases:
- "/2013/09/28/lazy-blogging-with-jekyll.html"
feedID: "https://blog.merovius.de//2013/09/28/lazy-blogging-with-jekyll.html"
title: Lazy blogging with jekyll
summary: I put up a small script to automate creating blog-posts in jekyll
---

**tl;dr: I put up a [small script](https://gist.github.com/Merovius/6736709) to
automate creating blog-posts in jekyll**

If you think about setting up your own blog, jekyll seems to be an appropriate
choice. This short guide should put you through the process of having an easy
setup for writing and deploying your blog via your favourite editor (vim) and
your favourite version control system (git) to a publicly available server via
ssh.

First thing you will need, is to have jekyll installed on both machines (the
ones where you will write your posts and the one where you will deploy them to).
Because the debian-version appears to be horribly outdated, I installed it via
gem. As far as I understood, this has the advantage of making an installation
without root-privileges possible. You should also have git available on both
machines.

Initializing your blog is pretty easy, `jekyll new mynewblog` (on your local
machine) should suffice.  You still want to do some configuration and
customization, most of which should be straight-forward. Edit the `index.html`,
the `_config.yml` and the `_layouts/default.html`. You might also want to have
an Atom-template, so people can subscribe to your blog in their favourite
RSS-reader. My good friend Stefan [helped with
that](https://git.yrden.de/?p=blog.git/.git;a=blob;f=atom.xml;hb=HEAD) just put
that file into the root of your blog-directory, edit your blogtitle and
everything into it and add the line
{{< highlight html >}}
<link rel="alternate" type="application/atom+xml" href="/atom.xml" title="Atom feed">
{{< / highlight >}}
in the `<head>` section of `_layouts/default.html`.

Next thing is setting up deployment. Just `git init` a blog, `git add` every
configuration file, page, template and whatnot and `git commit` it. ssh onto
your deployment-machine and do a `git init --bare blog.git`. Save the following
file to `blog.git/hooks/post-update` and change the path to point to a
directory, that is served by your http-server:
{% gist 6736709 post-update %}
Everytime you push into `blog.git` you will then have jekyll automatically
rebuild your blog. You now only have to do the following on your local machine
to deploy your blog:

```sh
git remote add origin username@example.com:blog.git
git push --set-upstream origin master
```

Now to the really fancy stuff. Jekyll expects your blogposts to live under the
`_posts`-directory under a special filename-format and to have a YAML-preamble,
containing some configuration. It can be quite cumbersome to manage this
yourself, so I wrote a [shellscript](https://gist.github.com/Merovius/6736709)
to ease the process. Put it anywhere in your path (i chose the name `newpost`)
and make it executable.

When you run the script, it will look into the current directory for a
jekyll-blog and create a draft from a small template given in the script. It
will then optionally run a jekyll-development server, so that you can preview
your blog-post in your browser (by saving the draft) and open the draft in your
favourite editor. After you close your editor, the jekyll server will be
stopped and the draft will be saved under `_posts/YYYY-MM-DD-abbrev-title.fmt`,
where `YYYY-MM-DD` is the current date (date and time will also be automatically
added to the YAML-preamble), `fmt` is a configurable format (markdown is default)
and `abbrev-title` is a short string derived from the title you put in.

There will also (optionally) be a git-commit created with a default
commit-message. You can edit the message in an editor and abort the commit, by
deleting everything and saving an empty commit-message. If you really want
(though I would not advise it) you can also automatically push it, after you're
done.

After this setup, to create a new blogpost, you just have to `cd` to your
blog-repository, run `newpost`, type your blogpost (and add a title), preview it
in your browsers, exit your editor and you have everything ready to push. It
can't get much easier.
