---
title: "Golang标准库：sync"
date: 2021-07-04T21:35:41+08:00
categories:
- 编程
tags:
- golang
- 标准库
cover: https://w.wallhaven.cc/full/y8/wallhaven-y85ojk.png
---

## sync 包



​	虽然go主要通过协程和通道来完成同步，但是在某些情况下使用sync包提供的同步更加简洁和巧妙。下面列举sync包的应用场景及相应的用法和坑。

## 一：单例



单例是经常要用到的一个编程模式，go最简单的单例实现就是使用 `sync.One`

### 示例

```go
package main

import (
	"fmt"
	"math/rand"
	"sync"
	"time"
)

type Instance struct {
	ID int
}

var (
	once     sync.Once
	instance Instance
)

func GetInstance() Instance {
	isNew := false
	once.Do(func() {
		rand.Seed(time.Now().UnixNano())
		instance = Instance{
			ID: rand.Intn(100),
		}
		isNew = true
	})
	if isNew {
		fmt.Printf("return new instance \n")
	} else {
		fmt.Printf("return instance that already exist. \n")
	}
	return instance
}

func main() {
	for i := 1; i <= 3; i++ {
		fmt.Printf("got instance: %d \n", GetInstance().ID)
	}
}

```

输出

```
return new instance 
got instance: 13 
return instance that already exist. 
got instance: 13 
return instance that already exist. 
got instance: 13
```



### 说明

从结果可以看到虽然调用了多次` GetInstance`函数，得到的却始终是同一个示例。这是因为`once.Do`只会执行一次。

### 注意点

不要在套娃使用同一个`Once`，会造成死锁，如：

```go
func ErrOnce() {
	var o sync.Once
	o.Do(func() {
		o.Do(func() {
			fmt.Println("never print")
		})
	})
}
```

`Once`内有一个互斥锁，这样使用会内层无法加锁

## 二：并发安全

`sync`提供的互斥锁可以保证数据（比如一个map）的并发安全

### 示例

```go
type RWMap struct {
    sync.RWMutex
    m map[int]int
}

func (m *RWMap) Set(k int, v int) {
    m.Lock()
    defer m.Unlock()
    m.m[k] = v
}

//...

```

### 说明

在对数据进行读写操作前加锁，操作后解锁即可。

对于`map`而言，这样实现并发安全性能不高，可以使用`sync.Map`。

## 三：一等多

如果有一个协程需要等待多个协程结束的场景，可以考虑`sync.WaitGroup`

### 示例

```go
package main

import (
	"fmt"
	"math/rand"
	"sync"
	"time"
)

var wg sync.WaitGroup

func Work(no int) {
	rand.Seed(time.Now().UnixNano())
	fmt.Printf("work - %d start.\n", no)
	time.Sleep(time.Duration(rand.Intn(5)) * time.Second)
	fmt.Printf("work - %d done\n", no)
    wg.Done() // 减少一个等待个数，等同于Add(-1)
}

func main() {
	for i := 0; i < 3; i++ {
		wg.Add(1) // 增加一个等待个数
		go Work(i)
	}
	wg.Wait() // 阻塞至等待个数归零，不能小于0
}
```

结果

```
work - 2 start.
work - 0 start.
work - 0 done.
work - 1 start.
work - 1 done.
work - 2 done.
```

### 说明

见注释

### 注意点

不要复制`WaitGroup`的值，而应使用指针，因为要保证所有操作在同一个`WaitGroup`上。

## 四：多等一

如果有多个协程需要等待一个协程的场景，可以考虑`sync.Cond`

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

var (
	done = false // 等待条件，由被等待的协程控制
	c    = sync.NewCond(&sync.Mutex{})
)

func consume(i int) {
	c.L.Lock() // 读条件前加锁
	for !done {
		fmt.Printf("c - %d waiting\n", i)
		c.Wait() // 先解锁，后挂起等待被唤醒，唤醒后再加锁
	}
	fmt.Printf("c - %d exit\n", i)
	c.L.Unlock() // 读条件后解锁
}

func main() {
	for i := 0; i < 3; i++ {
		go consume(i)
	}
	fmt.Println("do something")
	time.Sleep(3 * time.Second)
	fmt.Println("done")
	c.L.Lock() // 写条件前加锁
	done = true
	c.L.Unlock()  // 写条件后解锁
	c.Broadcast() // 广播通知所有协程，协程将会抢锁读条件
	time.Sleep(time.Second) // 等待协程结束
}
```

结果

```
c - 0 waiting
c - 2 waiting
c - 1 waiting
do something
done
c - 1 exit
c - 2 exit
c - 0 exit

```

### 说明

见注释，除了广播通知还可以用`Signal()`单个通知。

此外通过关闭通道实现一对多通知也是非常不错的方法。

### 注意点

1. 调用`Wait`前后要加解锁，如果不加锁`Wait`内部无法解锁，如果不解锁其他协程无法加锁
2. 同样不可复制值
3. 遵循FIFO规则，即先到先出
