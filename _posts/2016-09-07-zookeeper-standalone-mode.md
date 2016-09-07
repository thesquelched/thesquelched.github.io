---
layout: post
title: "Why Zookeeper starts up in standalone mode"
comments: true
description: "Why Zookeeper starts up in standalone mode"
keywords: "zookeeper standalone consul consul-template myid"
---

It's incredibly frustrating to see zookeeper 3.4.x servers come up in
standalone mode when you're trying to build a cluster.  Turns out that it only
happens when the server comes up before the
[myid](https://zookeeper.apache.org/doc/r3.3.2/zookeeperAdmin.html#sc_zkMulitServerSetup)
file exists, which is a problem when you're relying on
[consul-template](https://github.com/hashicorp/consul-template) to write it.
Go figure.
