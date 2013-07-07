---
layout: post
title: "GSoC Status: Week 1"
date: 2013-06-24 11:36
comments: true
categories: [GSoC, Monkey, Weekly Status]
---

{% img /images/blog/music_monkey.jpg %}
During the first week of GSoC I was investigating the APIs of MariaDB to get myself
familiar with its usages. I had done several experiments with a simple MariaDB
client program which includes connecting database, running queries, fetching results
and preparing SQL statements.

After getting a basic understanding of the MariaDB client library, I moved on to
the asynchronous APIs. There's a [post](https://kb.askmonty.org/en/using-the-non-blocking-library/)
explaining how to use non-blocking interfaces with a trivial example. The non-blocking
APIs are modelled after the normal blocking ones, for example the non-blocking
version of `mysql_real_query` is a couple of functions named `mysql_real_query_start`
and `mysql_real_query_cont`.

Besides the official introduction of non-blocking APIs, there's a Node.js package
`mariasql` that makes use of the non-blocking APIs and serves as a complete reference
of writing a MariaDB driver. I am studying the source code of `mariasql` and the
`redis` package of Monkey to see how I can integrate it with the Monkey's event
loop.

For the next week I will focus on designing the basic structures and functions for
Monkey MariaDB package. And the unfinished [blog entry](http://swpd.github.io/blog/2013/05/18/monkey-http-daemon-internals/)
discussing about Monkey intervals will continue to be updated from next week
(got a lot of stuffs related to graduation recently).
<!-- more -->
