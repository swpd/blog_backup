---
layout: post
title: "GSoC Status: Week 5"
date: 2013-07-21 19:50
comments: true
categories: [GSoC, Monkey, Weekly Status]
---
{%img /images/blog/seal.png %}
For the past a few days of this week, I was integrating the source code that required
to build libmysqlclient.a into MariaDB package. Meanwhile I added multiple statements
query support and connection pooling support to MariaDB package. All the features
in my proposal has been implemented and it is happy to announce this package is 
coming near to its completion.

integrate libmysqlclient.a
--------------------------
It took a bit longer than I expected it would be as MariaDB is such a huge project
(totally 223MiB after extracting). I did spend some time to figure out the relation
of different subparts and picked up those that are really necessary for building
the static library.

The build tools used by MariaDB is `cmake`, I was not fimiliar with `cmake` so that
took me some other time to learn some basic usage. I have consulted from
[blog series](https://kb.askmonty.org/en/compiling-mariadb-from-source/) of MariaDB
but it did not offer enough information. I turned to more comprehensive
[documentation](http://dev.mysql.com/doc/refman/5.7/en/source-installation.html),
although not all the options are available on MariaDB(such as `-DWITH_DEBUG`), it
did teach me how to use the same build options used by official MySQL release. (
with option `-DCMAKE_CONFIG=mysql_release`)

As there are so many build targets, I seeked help on irc #maria and some guy told
me to use `make mysqlclient` if all I wanted is the client library.

After I got all the missing pieces of the puzzle together, it finally worked out.
I have tested on two other Linux distribution to make sure it was not a pure
accident. :-D (Note: remember to install `cmake` and `libaio`)
<!-- more -->

multiple statements query support
---------------------------------
Multiple statements query allows user to submit a query with serveral statements,
this may lead to SQL injection and should be used carefully.

Using multiple statements query implies after processing there may be several
result sets, and we need to take care of every result sets and make sure it is
freed after used.

I added a new stage to handle next result set if there are more than one.

connection pooling support
--------------------------
Connection pooling aims to reduce resource allocation and release overhead. It
was implemented based on the basic part of MariaDB package.

The strategy currently used is to spawn serveal connections for every worker
thread and put them into thread's pool. Once a use require a connection from the
pool, the connection is moved from free queue to busy queue. The connection will
become available again once the user disconnect the connection, it will be returned
to the pool.

There are something I am still discussing with my mentor, we want to make it more
dynamic scalable, thus the pool size may expand when all the connections in the
pool are in use, and shrink the pool size when the usage rate is low.

However, shall we expand the pool size unconditionally? Definitely not, that would
generate resource overhead which against the reason we are using connection pooling.
That's when `lower_limit` and `upper_limit` are introduced, we can expand the pool
size if we do not exceed the `upper_limit` and shrink the pool size if it is still
larger than `lower_limit`. And there will be a `pool_size` to configure the initial
size of the pool, all of these will make the pool strategy more flexible.

plans for next week
-------------------
For the next week, I will improve the pool support for MariaDB, write documentation
for this package and an example explaining how to use it.
