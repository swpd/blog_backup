---
layout: post
title: "GSoC Status: Week 4"
date: 2013-07-14 19:40
comments: true
categories: [GSoC, Monkey, Weekly Status]
---
{%img /images/blog/cool_monkey.png %}
This week my time was mostly occupied by completing the MariaDB package. For now,
the basic version of MariaDB package is available, it can happily communicate
with the MariaDB server on my Linux box. I wrote a trivial duda web service to
test some common tasks when dealing with a relational database. During the
implementation and test I did encounter certain bugs, some are very obscure but
thank godness I finally fixed them. The current version is still not the final one,
some APIs may be altered in the future and new features will be introduced into
it. You can retrieve the code [here](https://github.com/swpd/duda_mariadb) for
your own interest. :-D

During this week I also discussed some questions I was not sure about with the
guy who implemented the asynchronous APIs for MariaDB(known as knielsen on #maria).
He was nice and answered all my questions in detail. And my mentor, edsiper, who
was busy lately, gave me some advices on how to deal with the duda apis. I wanna
thanks both of them here!

Here are some tasks I will be working on for the next week:

* write documentation to explain the design and usage of the MariaDB package.
* do some code cleanup and fix some potential bugs.
* write a more comprehensive example for demonstrating How-To use MariaDB package.
* do some survey and try to add connection pool support for the package to reduce
cost of connection setup and release.
* add source code of `libmariadbclient` as a third party dependency because on some 
Linux distribution this library may still not come with the asynchronous APIs.
* add mutli-statement query support for the package.
<!-- more -->
