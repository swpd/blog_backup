---
layout: post
title: "GSoC Status: Week 2"
date: 2013-06-30 23:54
comments: true
categories: [GSoC, Monkey, Weekly Status]
---

{%img /images/blog/jump_monkey.jpg %}
This week I came out with the API draft for Duda MariaDB package, it has been
pushed to Github but is still under development and may be altered later. For your
interest, you can refer to the repository and leave me some comments:
[Duda MariaDB package](https://github.com/swpd/duda_mariadb).

Roughly speaking, the APIs were influenced by the `mariasql` Node.js package and
the asynchronous access code of `redis`. Currently there are three header files,
`mariadb.h` contains event handling functions declaration for MariaDB connection
sockets and some global definitions; `connection.h` contains code related to
MariaDB client connections, it is responsible for connection management; `query.h`
takes care of stuffs related to a MariaDB query, every connection can have several
queries, the queries arer linked as a list and will be executed one by one.

[Monkey Internals](http://swpd.github.io/blog/2013/05/18/monkey-http-daemon-internals/) 
was updated this week (composed part of the scheduler section; the command line
utility section was finished), the remaining sections are still under writing. 

This week I also moved to my new apartment, everything got settled down and it
is a nice place :D.

For the next week, I will be implementing the APIs and try to put the package into
working.
<!-- more -->
