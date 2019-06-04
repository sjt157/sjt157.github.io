---
title: Netty-CPU使用率高达100%
date: 2019-01-18 16:26:30
tags: Netty
categories: Netty
---

Netty做数据转发，但是CPU使用率太高了。

* 解决方案：
`System.setProperty("org.jboss.netty.epollBugWorkaround", "true");//避免CPU使用率达到100%`

Googling for sun.nio.ch.WindowsSelectorImpl$SubSelector high cpu brings up a few hits from as last at 2015. Are you running an older version of Netty?

Also see https://github.com/netty/netty/issues/3857 - you may want to try running with -Dorg.jboss.netty.epollBugWorkaround=true.

<https://stackoverflow.com/questions/37881109/netty-eats-100-of-cpu>
