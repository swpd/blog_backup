---
layout: post
title: "GSoC Status: Week 9"
date: 2013-08-18 19:16
comments: true
categories: [GSoC, Monkey, Weekly Status]
---
{%img /images/blog/serious_elephant.jpg %}
This week I am happy to announce the basic PostgreSQL package can pass the build
process and integrate with Monkey. The asynchronous query handling have been added
to the package and serveral bugs got fixed. I wrote some tests to make sure it really
worked out. You may want to have a look at the [package](https://github.com/swpd/duda_postgresql).

Another stuff worth to mention about is I got a bug report for my MariaDB package
(thanks to `C1H1O3` on the irc), it is a good sign that someone is evaluating and
testing the package because that will make it more stable and robust.

I tried to reproduce the bug and traced the code with `gdb`, after several tests
I figured out the MariaDB package seemed to work properly. I wondered it might be
a bug from the Duda framework so I set up another tests without using MariaDB
package and could still reproduce the bug. It turned out to be a bug hidden in
Duda and I have reported it on the irc, `edsiper` will take care of it and I will
see what I can help.

C1H1O3 also advised me to use configuration file for connection pool parameters,
which seems like a good idea as we don't need to rebuild the application if all
we need is just to change the parameters.

Here are some tasks I will be hanlding for the next week:
* add connection pooling support to the PostgreSQL package
* as there are often more than one function to do the same thing in PostgreSQL
client libray `libpq`, I will add the sibling functions to the APIs of package to
make it more flexible.
<!-- more -->
