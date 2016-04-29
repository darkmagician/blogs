---
title: zookeeper小结
date: 2016-04-17 11:03:55
tags:
---

## zookeeper简介
就如当初单线程程序变成多线程程序一样，如今分布式系统越来越流行，分布式系统一般都支持local HA以及自由的伸缩性。当系统性能无法满足的时候，可以随时增加新实例来分担压力。另外在虚拟化以及云计算如此流行的今天，自由伸缩是一个很重要的功能。

如果自己开发一个分布式系统，考虑到分布式环境的特殊性，一般人估计没这个能力，就算做出来了，也难保没bug。而zookeeper的出现就是为了让这一切变的简单。

## 概念
zookeeper的数据保存在一个树状的结构中，就像文件系统的目录结构一个，每个数据节点就像一个目录，可以用path访问。
### 节点类型
1. 
## API
1. create。 创建新数据节点
1. 


## 部署
zookeeper支持单机以及多节点的部署。多节点部署保证服务的可用性，可以允许少部分节点失败。

比如部署n个节点的zookeeper，这些节点之间相互复制数据，只要大部分(大于n/2)的节点活着，zookeeper就可以保持服务正常。这样的设计可以防止脑裂的问题，集群的最新状态总是由大部分节点维护，所以节点数量最好是奇数个。

## zookeeper实现原理

### 数据读写
原则上，所有读取都是访问本地zookeeper节点，所以读取速度比较快。但是写入都是转发给leader，leader保证数据写入成功。leader拥有最新的数据。

### 选举leader
当集群启动或者当前的leader出现故障，就要做leader选举。选举的过程有点像Paxos算法。leader是由拥有最大zxid的节点(也就是说这个节点的状态是最新的)。

1. 每个candidate群发proposa。
1. 当其他节点收到proposal的时候，只接收最新的(zxid最大)请求，如果有的proposal已经接收，但是后来又收到更新的proposal，那就拒绝前面的proposal。
1. 而对于candidate而言，如果没有follower拒绝而且大部分节点接收了他的proposal，他就发commit请求到所有节点。
1. 当大部分follower接收了这个commit，这个candidate就成了leader。

### 数据写入
写入由leader负责，但是leader不仅要保证写入自己的内存，还要通知所有节点，只有当大部分节点都更新了，才算成功。整个过程有点像两次commitment。

1. leader 接收到写入请求。
1. leader 通知其他节点更新。
1. followers 回复ack。
1. leader 收到大部分ack后，发commit消息给followers。


## 使用场景

## 
