---
date: "2015-04-13T02:49:53Z"
tags:
- programming
aliases:
- "/2015/04/13/difficulties-making-sql-based-au.html"
feedID: "https://blog.merovius.de//2015/04/13/difficulties-making-sql-based-au.html"
title: SQL authentication timing side-channels
summary: It is surprisingly hard to make SQL-based authentication immune to timing side-channels,
  if you want to prevent enumeration of users.
---
I've been thinking about how to do an authentication scheme, that uses some
kind of relational database (it doesn't matter specifically, that the database
is relational, the concerns should pretty much apply to all databases) as a
backing store, in a way that is resilient against timing side-channel attacks
and doesn't leak any data about which usernames exist in the system and which
don't.

The first obvious thing is, that you need to do a constant time comparison of
password-hashes. Luckily, most modern crypto libraries should include something
like that (at least go's bcrypt implementation comes with that).

But now the question is, how you prevent enumerating users (or checking for
existence). A naive query will return an empty result set if the user does not
exists, so again, obviously, you need to compare against *some* password, even
if the user isn't found. But just doing, for example

```go
if result.Empty {
	// Compare against a prepared hash of an empty password, to have constant
	// time check.
	bcrypt.CompareHashAndPassword(HashOfEmptyPassword, enteredPassword)
} else {
	bcrypt.CompareHashAndPassword(result.PasswordHash, enteredPassword)
}
```

won't get you very far. Because (for example) the CPU will predict either of
the two branches (and the compiler might or might not decide to "help" with
that), so again an attacker might be able to distinguish between the two cases.
The best way, to achieve resilience against timing side-channels is to make
sure, that your control flow does not depend on input data *at all*. Meaning no
branch or loop should ever take in any way into account, what is actually input
into your code (including the username and the result of the database query).

So my next thought was to modify the query to return the hash of an empty
password as a default, if no user is found. That way, your code is guaranteed
to always get a well-defined bcrypt-hash from the database and your control
flow does not depend on whether or not the user exists (and an empty password
can be safely excluded in advance, as returning early for that does not give
any new data to the attacker).

Which sounds well, but now the question is, if maybe the timing *of your
database query* tells the attacker something. And this is where I hit a
roadblock: If the attacker knows enough about your code (i.e. what database
engine you are using, what machine you are running on and what kind of indices
your database uses) they can potentially enumerate users by timing your
database queries. To illustrate: If you would use a simple linear list as an
index, a failed search has to traverse the whole list, whereas a successfull
search will abort early. The same issue exists with balanced trees. An attacker
could potentially hammer your application with unlikely usernames and measure
the mean time to answer. They can then test individual usernames and measure if
the time to answer is significantly below the mean for failures, thus
enumerating usernames.

Now, I haven't tested this for practicality yet (might be fun) and it is pretty
likely that this can't be exploited in reality. Also, the possibility of
enumerating users isn't particularly desirable, but it is also far from a
security meltdown of your authentication-system. Nevertheless, the idea that
this theoretical problem exists makes me uneasy.

An obvious fix would be to make sure, that every query always has to search
the complete table on every lookup. I don't know if that is possible, it might
be just trivial by not giving a limit and not marking the username column as
unique, but it might also be hard and database-dependent because there will
still be an index over this username column which might still create the same
kind of issues. There will also likely still be a variance, because we
basically just shifted the condition from our own code into the DBMS. I have
simply no idea.

So there you have it. I am happy to be corrected and pointed to some trivial
design. I will likely accept the possibity of being vulnerable here, as the
systems I am currently building aren't that critical. But I will probably still
have a look at how other projects are handling this. And maybe if there really
is a problem in practice.
