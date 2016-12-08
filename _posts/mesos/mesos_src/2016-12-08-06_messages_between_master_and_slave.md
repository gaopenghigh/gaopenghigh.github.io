---
layout: post
title: "Mesos 源码学习(6) Master 和 Slave 之间的消息"
date: 2016-12-08 17:10:00 +0800
categories: Mesos
toc: true
---

Slave 初始化的最后，会做 Recovery ，而 Recovery 的最后，则调用 `detector->detect()` 方法找到
leader master ，找到后，就回调 `Slave:detected` 方法。

## Slave 向 Master 注册


`Slave::statusUpdate` 方法。