---
title: java高并发详解-06-ThreadGroup详解
copyright: true
comments: true
related_posts: true
date: 2021-02-23 00:08:17
tags: ThreadGroup
categories: 高并发详解
---

- 创建线程的时候如果没有显示得指定ThreadGroup，那么新的线程会被加入到父线程相同的ThreadGroup中
- 其实这里主要就是看ThreadGroup的源码，以及Thread的源码，就可以知道关于Thread的一些规则，比如 Thread的优先级 的范围，以及不能超过ThreadGroup的MAX



# ThreadGroup 与Thread