---
layout: post
title: "GSoC Status: Week 6"
date: 2013-07-28 00:22
comments: true
categories: [GSoC, Monkey, Weekly Status]
---
{%img /images/blog/seal_wins.jpg %}
This week the Duda MariaDB package was completed. I spent some time on reimplementing
the connection pooling due to my misunderstanding of it. The new version maintain
a pool of connections that are either connected or connecting. And a pool is
associated with a web service instead of a thread. The pool shall be defined as
global in the web service, but due to the lack of isolate namespace there can't
be same pool names in differenct web services.

The documentation of MariaDB package was also updated to explain the usage of it.
And for those who want to install this package, please refer to the [Monkey fork](https://github.com/swpd/monkey)
of my Github. Clone that repo and checkout to mariadb branch, environment needed
to build this package is set up for you. (Notice: you need `cmake` and `libaio`
to make things work, so don't forget to install them.) More information can be
found [here](https://github.com/swpd/duda_mariadb/blob/master/README.md).


Currently I am working on an example to demonstrate the usage of this package. It
will be finished very soon.

For next week I will make some potential improvemnts to MariaDB package, start
to review the asynchronous APIs of PostgreSQL and do some experiments.
