---
layout: post
title: "GSoC Status: Week 3"
date: 2013-07-06 11:21
comments: true
categories: [GSoC, Monkey, Weekly Status]
---
{%img /images/blog/running_monkey.jpg %}
This week I spent most of my time implementing the MariaDB package, it is still
incomplete but I've pushed it to Github, check [it](https://github.com/swpd/duda_mariadb)
out if you are interested. 

The problem I am currently stuck is for every continue part of the asynchronous
version MariaDB APIs, we need to pass the `ready_status`(bitmask of all events
occurred on a MariaDB connection socket) in able to issue the call. However,
the socket event callback functions are spilt into five sepreated functions,
and every function only knows the corresponding event it shall handle but no idea
about all the events available on the MariaDB socket. This makes it more tricky
to write the callback functions right, I need to make sure what events actually
occurs on every stage of a MariaDB's connection lifecycle.

Meanwhile, I am also discussing with Eduardo Silva to see what we can do with it.
He did offer me some useful advinces and will show me some examples later. And
as he recommended, we may add connection pool support to our MariaDB package to
avoid connect-on-demand mechanism.

For the next week, I will keep hard working on making the implementation complete
and integrate it with Duda.
<!-- more -->
