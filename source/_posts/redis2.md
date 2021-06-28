---
title: "Redis（二）：链表"
date: 2021-03-09T20:13:45+08:00
categories:
- 学习笔记
- Redis
tags:
- Redis数据结构与对象
keywords:
- redis
- 链表
comments: false
#thumbnailImage: images/host/redis/redis-icon-logo.png
summary: 链表被广泛用于实现Redis的各种功能，比如列表键、发布与订阅、慢查询、监视器等。本文介绍了Redis链表的结构和特征。
---

<!--more-->

**链表**被广泛用于实现Redis的各种功能，比如列表键、发布与订阅、慢查询、监视器等。直接看结构示意图：

![链表结构示意图](/images/host/redis/redis-list.png)

##### 特征总结如下：

- **双端**：链表节点带有prev和next指针，获取某个节点的前置节点和后置节点的复杂度都是O（1）。
- **无环**：表头节点的prev指针和表尾节点的next指针都指向NULL，对链表的访问以NULL为终点。
- **带表头指针和表尾指针**：通过list结构的head指针和tail指针，程序获取链表的表头节点和表尾节点的复杂度为O（1）。
- **带链表长度计数器**：程序使用list结构的len属性来对list持有的链表节点进行计数，程序获取链表中节点数量的复杂度为O（1）。
- **多态**：链表节点使用void*指针来保存节点值，并且可以通过list结构的dup、free、match三个属性为节点值设置类型特定函数，所以链表可以用于保存各种不同类型的值。
  - dup函数用于复制链表节点所保存的值；
  - free函数用于释放链表节点所保存的值；
  - match函数则用于对比链表节点所保存的值和另一个输入值是否相等。

##### 总结

链表在数据结构中相当经典实用。

