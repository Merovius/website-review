---
date: "2014-01-23T04:04:25Z"
tags:
- crypto
- meta
aliases:
- "/2014/01/23/signed-blog-posts.html"
feedID: "https://blog.merovius.de//2014/01/23/signed-blog-posts.html"
title: Signed blog posts
summary: I sign my blog posts.
---

**tl;dr: I sign my blog posts. curl
http://blog.merovius.de/2014/01/23/signed-blog-posts.html | gpg**

**Update: I no longer do this. I haven't for a while**

I might have to update my TLS server certificate soon, because the last change
seems to have broken the verification of https://merovius.de/. This is nothing
too exciting, but it occured to me that I should actually provide some warning
or notice in that case, so that people can be sure, that there is nothing
wrong. The easiest way to accomplish this would be a blogpost and the easiest
way to verify that the statements in that blogpost are correct would be, to
provide a signed version. So because of this (and, well, because I can) I
decided to sign all my blogposts with my gpg-key. People who know me should
have my gpg key so they can verify that I really have written everything I
claim.

I could have used
[jekyll-gpg_clearsign](https://github.com/kormoc/jekyll-gpg_clearsign), but it
does not really do the right thing in my opinion. It wraps all the HTML in a
GPG SIGNED MESSAGE block and attaches a signature. This has the advantage of
minimum overhead - you only add the signature itself plus some constant
comments of overhead. However, it makes really verifying the contents of a
blogpost pretty tedious: You would have to either manually parse the HTML in
your mind, or you would have to save it to disk and view it in your browser,
because you cannot be sure, that the HTML you get when verifying it via curl on
the commandline is the same you get in your browser. You could write a
browser-extension or something similar that looks for these blocks, but still,
the content could be tempered with (for example: Add the correctly signed page
as a comment in a tampered with page. Or try to somehow include some javascript
that changes the text after verifying…). Also, the generated HTML is not really
what I want to sign; after all I can not really attest that the HTML-generation
is really solid and trustworthy, I never read the jekyll source-code and I
don't want to, at every update. What I really want to sign is the stuff I wrote
myself, the markdown (or whatever) I put into the post. This has the additional
advantage, that most markdown is easily parseable by humans, so you can
actually have your gpg output the signed text and immediately read everything I
wrote.

So this is, what happens now. In every blogpost there is a HTML-comment
embedded, containing the original markdown I wrote for this post in compressed,
signed and ASCII-armored form. You can try it via

	curl http://blog.merovius.de/2014/01/23/signed-blog-posts.html | gpg

This should output some markdown to stdout and a synopsis of gpg about a valid
(possibly untrusted, if you don't have my gpg-key) signature on stderr. Neat!

The [changes](http://git.merovius.de/posts/commit/?id=dd005159f9fb25ebc8ef789608a609bcb65fc62c)
needed in the blog-code itself where pretty minimal. I had however (since I
don't want my gpg secret key to be on the server) to change the deployment a
little bit. Where before a git push would trigger a hook on the remote
repository on my server that ran jekyll, now I have a local script, that wraps
a jekyll build, an rsync to the webserver-directory and a git push. gpg-agent
ensures, that I am not asked for a passphrase too often.

So, yeah. Crypto is cool. And the procrastinator prevailed again!
