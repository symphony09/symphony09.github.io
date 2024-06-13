---
title: Coroutines for Go -1
date: 2024-06-13T20:00:00+08:00
categories:
  - 编程
tags:
  - golang
  - 翻译
---

> 这篇文章翻译自 Go 团队主要成员 [Russ Cox](https://swtch.com/~rsc/)的协程提案[research!rsc: Coroutines for Go (swtch.com)](https://research.swtch.com/coro)，此协程 coroutines 非彼协程 goroutine，由于全文较长，预计分为两篇，这篇主要说明是什么和为什么。本文采用AI辅助翻译，不妥之处，欢迎指正。

本文讲述了为什么我们需要一个 Go 协程包，以及它将成为什么样子。但首先，什么是协程？

今天的程序员都非常熟悉函数调用（子程序）：F 调用 G，F 暂停，G 运行。G 完成其工作，可能调用并等待其他函数，然 后返回。当 G 返回时，G 消失，F 继续运行。在这种模式下，只有一个函数在运行，而其调用者等待，整个调用栈都是如此。

与子程序不同，协程在不同的栈上并发运行，但仍然只有一个协程在运行，同时其调用者等待。F 开始 G，但 G 不立即运行。相反，F 必须显式地恢复 （resume）G，然后 G 才能运行。在任何时候，G 都可以反过来通过 yield 回到 F，这会暂停 G，然后继续 F 。最终 F 再次调用 resume，这会暂停 F，然后继续 G。他们继续反复运行，直到 G 返回，这会清理 G，并从其最近的恢复操作继续 F，并给 F 发出信号，表明 G 已完成，F 应不再尝试重新启动 G。在这种模式下，只有一个协程在运行，而其调用者在不同的栈上等待。他们以一种定义明确、协调的方式轮流运行。

这有点抽象。让我们看一些实际的程序。
## Lua中的协程

要使用一个著名的示例，我们可以考虑比较两个二叉树，看它们是否具有相同的值序列，即使它们的结构不同。例如，以下 是使用Lua 5编写的生成一些二叉树的代码：

```lua
function T(l, v, r)
    return {left = l, value = v, right = r}
end

e = nil
t1 = T(T(T(e, 1, e), 2, T(e, 3, e)), 4, T(e, 5, e))
t2 = T(e, 1, T(e, 2, T(e, 3, T(e, 4, T(e, 5, e)))))
t3 = T(e, 1, T(e, 2, T(e, 3, T(e, 4, T(e, 6, e)))))
```

t1和t2两个二叉树都包含值1，2，3，4，5；t3包含1，2，3，4，6。

 我们可以编写一个协程来遍历一棵树并返回每个值：

```lua
function visit(t)
    if t ~= nil then  -- note: ~= is "not equal"
        visit(t.left)
        coroutine.yield(t.value)
        visit(t.right)
    end
end
```

为了比较两棵树，我们可以创建两个访问协程，然后交替读取并比较相邻的值：

```lua
function cmp(t1, t2)
    co1 = coroutine.create(visit)
    co2 = coroutine.create(visit)
    while true
    do
        ok1, v1 = coroutine.resume(co1, t1)
        ok2, v2 = coroutine.resume(co2, t2)
        if ok1 ~= ok2 or v1 ~= v2 then
            return false
        end
        if not ok1 and not ok2 then
            return true
        end
    end
end
```

t1和t2参数仅在第一次迭代中用于协程 resume，作为访问参数。后续的resume返回值来自协程 yield，但代码会忽略该值。

【译者注】第一次迭代调用 `visit(t1)`，在 visit 函数中先尝试 `visit(t1.left)`，如果左子树为空，那么就执行`coroutine.yield(t.value)`，yield 是有返回值的，就是 cmp 函数中 `coroutine.resume` 传入的 t1，但是这个 t1 参数被忽略了。

一个更符合Lua语言习惯的版本是使用coroutine.wrap，它返回一个隐藏协程对象的函数：

```lua
function cmp(t1, t2)
    next1 = coroutine.wrap(function() visit(t1) end)
    next2 = coroutine.wrap(function() visit(t2) end)
    while true
    do
        v1 = next1()
        v2 = next2()
        if v1 ~= v2 then
            return false
        end
        if v1 == nil and v2 == nil then
            return true
        end
    end
end
```

 当协程完成时，next 函数返回nil。

## Python中的生成器（在CLU中为迭代器）

Python提供了类似于Lua的协程的生成器，但它们并不是协程，所以值得指出它们之间的差异。主要区别在于，“直白的”程序并不行得通。例如，这里是我们的Lua树和访问者的直接Python翻译：

```python
def T(l, v, r):
    return {'left': l, 'value': v, 'right': r}

def visit(t):
    if t is not None:
        visit(t['left'])
        yield t['value']
        visit(t['right'])
```

但是这个直白的翻译行不通：

```python
>>> e = None
>>> t1 = T(T(T(e, 1, e), 2, T(e, 3, e)), 4, T(e, 5, e))
>>> for x in visit(t1):
...     print(x)
...
4
>>>
```

我们失去了1，2，3和5。发生了什么？

在Python中，那个 visit 函数并没有定义为一个普通函数。因为 body 中包含一个 yield 语句，所以结果是一个生成器：

```python
>>> type(visit(t1))
<class 'generator'>
>>>
```

visit(t['left']) 的调用并没有运行 visit 函数中的任何代码。它只是创建并返回一个新的生成器，然后被丢弃。为了避 免丢弃这些结果，您必须遍历生成器并重新 yield 它们：

```python
def visit(t):
    if t is not None:
        for x in visit(t['left']):
            yield x
        yield t['value']
        for x in visit(t['right'])
            yield x
```

 Python 3.3 引入了 `yield` `from` 的语法，允许：

```python
def visit(t):
    if t is not None:
        yield from visit(t['left']):
        yield t['value']
        yield from visit(t['right'])
```

生成器对象包含单个 visit 调用的情况下的状态信息，这意味着局部变量值和当前执行的行号。该状态每次生成器重新开始时都会压入调用栈中，然后在每次 yield 时从调用栈中弹出到生成器对象，这只能发生在顶部的调用栈帧。这样，生成器使用与原始程序相同的栈，避免了需要完整的协程实现，但引入了这些令人困惑的限制。

【译者注】这里我理解是相当于把遍历操作换成了递归操作。

 Python的生成器似乎几乎完全是从CLU复制过来的，CLU创始人提供了这个抽象（以及其他许多东西）。虽然CLU将其称为迭代器，而不是生成器，但CLU树迭代器看起来像这样：

```
visit = iter (t: cvt) yields (int):
    tagcase t
        tag empty: ;
        tag non_empty(t: node):
            for x: int
                in tree$visit(t.left) do
                    yield(x);
                    end;
            yield(t.value);
            for x: int
                in tree$visit(t.right) do
                    yield(x);
                    end;
        end;
    end visit;
```

特别是对于检查树的标记联合表示使用的标签，语法有所不同，但是基本结构，包括嵌套的 for 循环，与最初的 Python 版本完全相同。此外，由于CLU是静态类型，因此visit显然被标记为迭代器（iter）而不是函数（proc在CLU中）。由于类型信息的帮助，编译器可以（我假设也）诊断出将visit作为普通函数调用的错误，就像我们那个有 bug 的Python示例一样。

关于CLU的实现，原始实现者写道：“迭代器是协程的一种形式；然而，它们的使用受到足够的限制，因此仅使用程序栈来实 现。”使用迭代器因此的开销比使用过程要稍微高一些。这听起来与我为Python生成器所提供的解释完全相同。此外，更多信息， 可以参考[Abstraction Mechanisms in CLU](https://dl.acm.org/doi/10.1145/359763.359789)，特别是第4.2节，第4.3节和第6节。
##  协程，线程和生成器

首先，协程，线程和生成器看起来都很相似。它们都提供某种形式的并发，但它们在一些重要方面有所不同。

-  协程通过无并行性提供并发：当一个协程运行时，恢复或让给它的那个协程或线程都没有并行执行。
	-  因为协程一个接一个地运行，只在程序的特定点进行切换，所以协程之间可以共享数据而不发生竞争。显式的切换（如第一 个Lua示例中的coroutine.resume或第二个Lua示例中的调用下一个函数）作为同步点，创建 happens-before 边。
	- 因为调度是显式的（没有任何抢占），并且完全由操作系统之外完成，协程切换最多只需要大约十纳秒，通常甚至更少。启动和退出比线程便宜得多。
-  线程比协程具有更多的能力，但成本更高。额外的能力是并行性，成本是调度器的开销，包括更昂贵的上下文切换和需要在 某种程度上实现预夺。通常操作系统提供线程，线程切换需要几微秒。
	- 对于这个分类法，Go的goroutines是便宜的线程：goroutine切换接近于几百纳秒，因为Go运行时承担了部分调度工作，但goroutines仍然提供了线程的全部并行性和预夺。 （Java的新轻量级线程基本上与goroutines相同。）
- 生成器提供的能力比协程少，因为只有协程的顶层帧允许 yield。这个帧在对象和调用堆栈之间来回移动，以暂停和恢复它 。

 协程对于编写希望使用程序结构而不是并行的程序是一个有用的构建块。有关详细示例，请参阅我之前的帖子，“[Storing Data in Control Flow](https://research.swtch.com/pcdata)”。其他示例，请参阅Ana Lúcia De Moura和Roberto Ierusalimschy于2009年发表的论文 “[Revisiting Coroutines](https://dl.acm.org/doi/pdf/10.1145/1462166.1462167)”。 对于原始示例，请参阅Melvin Conway于1963年发表的论文“[Design of a Separable Transition-Diagram Compiler](https://dl.acm.org/doi/pdf/10.1145/366663.366704)”。
## 为什么要在 Go 中实现协程

协程是一种并发模式，而不是现有的Go并发库直接提供的。goroutine通常是足够接近的，但我们看到了，它们并不相同，有时候这种差异很重要。

例如，Rob Pike的2011年的演讲“[Lexical Scanning in Go](https://go.dev/talks/2011/lex.slide)”介绍了text/template包的原始词法器和解析器。它们分别运行在不同的goroutine中，通过一个channel进行通信， imperfectly模拟了一对协程：词法器进程正向前看，而解析器处理最近的。生成器是不够好的——词法器产生许多不同的函数的结果——但full goroutines被证明是有点太多的。goroutines提供的并行性引起了竞争，最终导致放弃该 设计，转而使用词法器将状态存储在对象中，这更像是协程的真实模拟。适当的协程可以避免竞争，并且比goroutines更高效。

一个预期的未来 coroutines 使用案例在 Go 中的是遍历通用集合。我们讨论了为 Go 添加支持范围 over 函数，这将鼓励集合和其他抽象提供类似 CLU 样式的迭代器函数。迭代器如今可以使用函数值实现，而无需任何语言更改。例如，一个稍微简化 版的树迭代器可以在 Go 中如下所示：

```go
func (t *Tree[V]) All(yield func(v V)) {
    if t != nil {
        t.left.All(yield)
        yield(t.value)
        t.right.All(yield)
    }
}
```

那个迭代器现在可以如下所示进行调用：

```go
t.All(func(v V) {
    fmt.Println(v)
})
```

 也许在 Go 的一个未来版本中，可以以如下方式调用：

```go
for v := range t.All {
    fmt.Println(v)
}
```

有时，我们可能需要以一种不适合单层 for 循环的方式遍历集合。二叉树比较就是一个例子：两个迭代需要以某种方式交替进行。正如我们已经看到的，协程可以提供解决方案，让我们将像 Tree.All這樣的“推”迭代器变成返回一个值流，每个调用返回一个值 的“拉”迭代器。
