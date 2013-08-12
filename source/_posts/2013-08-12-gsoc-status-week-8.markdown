---
layout: post
title: "GSoC Status: Week 8"
date: 2013-08-12 08:59
comments: true
categories: [GSoC, Monkey, Weekly Status]
---
{%img /images/blog/fat_elephant.jpg %}
This week the draft APIs of PostgreSQL package has come out. You may check it out
on my [Github](https://github.com/swpd/duda_postgresql), feel free to leave me a
comment.

The APIs are modeled after the previous [MariaDB](https://github.com/swpd/duda_mariadb)
package. By doing so we can hide the differences between MariaDB and PostgreSQL
from our users and let the packages deal with the details.

The asynchronous APIs of PostgreSQL provide a more flexible way to issue non-blocking
connection and query, for example you can use a string array to specify the connection
options you are interested and leave others as default. Currently I am working on
integraing the APIs with the event handling module of Duda.

Next week I will mainly focus on the asynchronous handling implementation of the
PostgreSQL package.
<!-- more -->
