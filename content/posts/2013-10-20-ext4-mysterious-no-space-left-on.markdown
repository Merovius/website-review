---
date: "2013-10-20T21:13:07Z"
tags:
- linux
aliases:
- "/2013/10/20/ext4-mysterious-no-space-left-on.html"
feedID: "https://blog.merovius.de//2013/10/20/ext4-mysterious-no-space-left-on.html"
title: 'ext4: Mysterious “No space left on device”-errors'
summary: ext4 has a feature called dir_index enabled by default, which is quite susceptible
  to hash-collisions
---

**tl;dr: ext4 has a feature called `dir_index` enabled by default, which is
quite susceptible to hash-collisions**

I am currently restructuring my mail-setup. Currently, I use offlineimap to
sync my separate accounts to a series of maildirs on my server. I then use sup
on the server as a MUA. I want to switch to a local setup with notmuch, so I
set up an dovecot imapd on my server and have all my accounts forward to my
primary address. I then want to use offlineimap to have my mails in a local
maildir, which I browse with notmuch.

I then stumbled about a curious problem: When trying to copy my mails from my
server to my local harddisk, it would fail after about 50K E-mails with the
message “could not create xyz: no space left on device” (actually, offlineimap
would just hog all my CPUs and freeze my whole machine in the process, but
that's a different story). But there actually was plenty of space left.

It took me and a few friends a whole while to discover the problem. So if you
ever get this error message (using ext4) you should probably check these four
things (my issue was the last one):

## Do you *actually* have enough space?

Use `df -h`. There is actually a very common pitfall with ext4. Let's have a look:

```
mero@rincewind ~$ df -h
Filesystem              Size  Used Avail Use% Mounted on
/dev/mapper/sda2_crypt  235G  164G   69G  71% /
...
```

If you add 164G and 69G, you get 233G, which is 2G short of the actual size.
This is about 1%, but on your system it will likely be more of 5% difference.
The reason is the distinction between "free" and "available" space. Per default
on ext4, there are about 5% of "reserved" blocks. This has two reasons: First
ext4's performance seems to take a small hit, when almost full. Secondly, it
leaves a little space for root to login and troubleshoot problems or delete
some files, when users filled their home-directory. If there was *no* space
left, it might well be, that no login is possible anymore (because of the
creation of temporary files, logfiles, history-files…). So use `tune2fs
<path_to_your_disk>` to see, if you have reserved blocks, and how many of them:

```
mero@rincewind ~$ sudo tune2fs -l /dev/mapper/sda2_crypt | grep "Reserved block"
Reserved block count:     2499541
Reserved blocks uid:      0 (user root)
Reserved blocks gid:      0 (group root)
```

## Do you have too many files?

Even though you might have enough space left, it might well be, that you have
too many files. ext4 allows an enormous amount of files on any file system, but
it is limited. Checking this is easy: Just use `df -i`:

```
Filesystem               Inodes  IUsed    IFree IUse% Mounted on
/dev/mapper/sda2_crypt 15622144 925993 14696151    6% /
...
```

So as you see, that wasn't the problem with me. But if you ever have the `IUse%`
column near 100, you probably want to delete some old files (and you should
*definitely* question, how so many files could be created to begin with).

## Do a file system check

At least some people on the internet say, that something like this has
happened to them after a crash (coincidentally my system crashed before the
problem arose. See above comments about offlineimap) and that a file system
check got rid of it. So you probably want to run `fsck -f <path_to_your_disk>`
to run such a check. You probably also want to do that from a live-system, if
you cannot unmount it (for example if it's mounted at the root-dir).

## Do you have `dir_index` enabled?

So this is the punch line: ext4 has the possibility to hash the filenames of
its contents. This enhances performance, but has a “small” problem: ext4 does
not grow its hashtable, when it starts to fill up. Instead it returns -ENOSPC
or “no space left on device”.

ext4 uses `half_md4` as a default hashing-mechanism. If I interpret my
google-results correctly, this uses the md4-hash algorithm, but strips it to 32
bits. This is a classical example of the
[birthday-paradox](http://en.wikipedia.org/wiki/Birthday_problem): A 32 bit
hash means, that there are 4294967296 different hash values available, so if we
are fair and assume a uniform distribution of hash values, that makes it highly
unlikely to hit one specific hash. But the probability of hitting two identical
hashes, given enough filenames, is much much higher. Using the
[formula](http://en.wikipedia.org/wiki/Birthday_problem#Cast_as_a_collision_problem)
from Wikipedia we get (with about 50K files) a probability of about 25% that a
newly added file has the same hash. This is a huge probability of failure. If
on the other hand we take a 64bit hash-function the probability becomes much
smaller, about 0.00000000007%.

So if you have a lot of files in the same directory, you probably want to switch
off `dir_index`, or at least change to a different hash-algorithm. You can
check if you have `dir_index` enabled and change the hash, like this:

```
mero@rincewind ~$ sudo tune2fs -l /dev/mapper/sda2_crypt | grep -o dir_index
dir_index

## Change the hash-algo to a bigger one
mero@rincewind ~$ sudo tune2fs -E "hash_alg=tea" /dev/mapper/sda2_crypt
## Disable it completely
mero@rincewind ~$ sudo tune2fs -O "^dir_index"
```

Note however, that `dir_index` and `half_md4` where choices made for
performance reasons. So you might experience a performance-hit after this.

**UPDATE:** After trying it out, I realized, that the problem actually also
persists with the tea-hash. I then had a look at [the
ext4-documentation](https://ext4.wiki.kernel.org/index.php/Ext4_Disk_Layout#Hash_Tree_Directories)
about the topic and it seems, that the hash is only stored as 32 bits, so it
actually does not matter what hash we choose, regarding this particular
problem. So if `half_md4` is chosen [because of its better performance and
collision-resistance](http://git.whamcloud.com/?p=tools/e2fsprogs.git;a=commitdiff_plain;h=d1070d91b4de8438dc78c034283baaa19b31d25e)
it actually makes sense to leave it as the default. You can by the way easily
test and reproduce the issue by using the following on an ext4 file system:

```
for a in `seq 100000`
do
        file=`head -c 51 /dev/urandom | base64 | tr '/' '_'`
        touch $file
done
```

Curiously, this only gives me about 160 collisions on 100K files (instead of
about 10K collisions on 60K files), which would suggest, that my original
sample (meaning my mailbox) exhibits some properties that make collisions more
likely both on `half_md4` *and* `tea`.
