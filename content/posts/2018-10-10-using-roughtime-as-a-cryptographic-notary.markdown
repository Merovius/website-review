---
date: "2018-10-10T23:22:40Z"
tags:
- crypto
- golang
- programming
aliases:
- "/2018/10/10/using-roughtime-as-a-cryptographic-notary.html"
omitFeed: true
title: Using roughtime as a "cryptographic notary"
summary: Roughtime can be (ab)used for Trusted Timestamping. I wrote a simple tool as
  a PoC
---

**tl;dr: Roughtime can be (ab)used for [Trusted Timestamping][tsa]. I wrote a
[simple tool][notary] as a PoC**

Recently, [Cloudflare announced][cloudflare] that they are now running a
[roughtime][roughtime] server. Roughtime is a cryptographically secured
time-synchronization protocol - think NTP with signatures. For an actual
description of how it works, I recommend reading the Cloudflare blog post. But
at it's very core (oversimplification ahead), the user chooses an arbitrary
(usually randomly generated) nonce and the server signs it, plus the current
time.

One thing roughtime adds on top of this, is the ability to build a chain of
requests. This is achieved by taking a hash of a response, combining it with a
randomly generated "blind" and using the combined hash as a nonce to the next
request. The intended use-case of this is that if a server provides the wrong
time or otherwise misbehaves, you can obtain cryptographic proof of that fact
by getting a timestamped signature of its response from a different server. By
storing the initial nonce, generated blinds and responses, the entire chain can
be validated later.

When I saw Cloudflares announcement, my first thought was that it should be
possible to use a roughtime server as a [Time Stamping Authority][tsa]. The
goal is, to obtain a cryptographic proof, that you owned a particular document
at the current point in time - for example to ensure you can proof original
authorship without publishing the document itself.

The simplest way to achieve this using roughtime is to use the SHA512 hash of
the file as an initial nonce. That way, the roughtime server signs that hash
together with the current time with their private key. By using the roughtime
chain protocol, you can get that proof corroborated by multiple servers.

You can also think of extending this, to get stronger properties. Using the
hash of the file as a nonce only proves that the file existed *before* that
specific point in time. It also doesn't actually prove that you had the file,
but only the hash. This can be remediated though. If we run a regular roughtime
request, the resulting response is unpredictable (to us) and signs the current
time. Thus, if we use a hash of that response as a prefix "salt" of the file
itself, the resulting hash proofs that we knew the file *after* that chain
ran. We can then use that hash as a nonce for another roughtime chain and get a
proof that we had the file at a specific point (or rather, a small interval) in
time. Furthermore, we can opt to use the file-hash not as the nonce itself, but
as a blind. The advantage is, that the blind is never transmitted over the
network, so the actual proof is only available to us (if we use it as a nonce,
an eavesdropper could intercept the proof). I illustrated these options in a
[recent talk][slides] I gave on the subject.

These ideas are mostly academic. I'm not sure how useful these properties are
in practice. Nevertheless, the idea intriguiged me enough to [implement it][notary]
in a simple tool. It's in a pretty rough, proof-of-concept like shape and I
don't know if I will ever progress it from there. It also comes with a client
implementation of the roughtime protocol in Go - initially I was not aware that
there already was a Go implementation, but that also is not go-gettable. Either
way, it was fun to implement it myself :)

[cloudflare]: https://blog.cloudflare.com/roughtime/
[roughtime]: https://roughtime.googlesource.com/roughtime/
[tsa]: https://en.wikipedia.org/wiki/Trusted_timestamping#Trusted_(digital)_timestamping
[slides]: https://docs.google.com/presentation/d/1quTJfXHvBZCjKJgL6HjUFb_jhDF-PghBwm_lTFLAjdg/edit#slide=id.g43c753f2a5_1_542
[notary]: https://github.com/Merovius/notary
