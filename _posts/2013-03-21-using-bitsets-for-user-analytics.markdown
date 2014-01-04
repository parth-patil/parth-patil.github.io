---
layout: post
title: "Using BitSets for User Analytics"
date: 2013-03-21 23:55
comments: true
tags: [scala, analytics]
---
I read about a neat trick to do user analytics using Redis BitSets in [this article](http://blog.getspool.com/2011/11/29/fast-easy-realtime-metrics-using-redis-bitmaps/)
And I thought to hack up a simple finagle(finatra) based service that provides an API to do this kind of user analytics without needing Redis.

For BitSets I used the [javaewah library](https://code.google.com/p/javaewah/)

So I have started a small project called cardino-db and you can check the source code [here](https://github.com/parth-patil/cardino-db)
