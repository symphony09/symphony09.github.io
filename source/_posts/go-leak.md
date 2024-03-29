---
title: "Golang内存泄露"
date: 2021-06-30T19:38:41+08:00
categories:
- 编程
tags:
- golang
- 最佳实践
cover: https://w.wallhaven.cc/full/y8/wallhaven-y85ojk.png
---

## 防患未然

1. 避免引用大字符串和大切片的一小部分导致无用部分无法被回收，可以复制有用部分到新串和新切片以解除引用。对于切片，也可以将无用部分设nil。
2. 避免因为代码设计中的一些错误而导致一些协程处于永久阻塞状态。如：
    - 从一个永远不会有其它协程向其发送数据的通道接收数据；
    - 向一个永远不会有其它协程从中读取数据的通道发送数据；
    - 被自己死锁了；
    - 和其它协程相互死锁了；
    - 等等。
3. 对于不再使用使用的`time.Ticker`要记得调用`Stop`方法。
4. 避免将终结器设置到一个循环引用值组中的一个值上。
5. 避免无脑积压`defer`，可以包裹在一个匿名函数内，提前释放不再使用的资源。

## 亡羊补牢

使用 go pprof 分析定位内存泄露的地方





