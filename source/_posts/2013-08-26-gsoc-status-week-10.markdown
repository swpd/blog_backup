---
layout: post
title: "GSoC Status: Week 10"
date: 2013-08-26 12:04
comments: true
categories: [GSoC, Monkey, Weekly Status]
---
{%img /images/blog/war_elephant.jpg %}
This week the PostgreSQL package was completed, connection pooling support was
successfully added. Meanwhile, the sibling functions that are used for querying
and escaping string was also introduced into the package to provide more choice
for the user.

In order to demonstrate the usage of the PostgreSQL package, I have created
another duda web service [duda_postgresql_demo](https://github.com/swpd/duda_postgresql_demo)
on Github, if you ever used the mariadb_demo web service, you'll find it pretty
similar. Check it out for your interest.

From now on till the end of GSoC I will test the two packages I am in charge of
and fix some potential bugs to make them more stable. The documentation of
PostgreSQL package will be completed in no time. Last but not least, I shall
continue the [blog](http://swpd.github.io/blog/2013/05/18/monkey-http-daemon-internals/)
about Monkey internals that I haven't finished yet.
<!-- more -->
