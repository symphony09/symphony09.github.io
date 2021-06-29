---
title: "Golang最佳实践：Functional Option"
date: 2021-06-29T19:38:41+08:00
categories:
- 编程
tags:
- golang
- 最佳实践
cover: https://w.wallhaven.cc/full/y8/wallhaven-y85ojk.png
---

## 使用场景

在初始化结构体时常常会碰到这样的情况，一些字段是不能为空且没有默认值，一些字段是不能为空但有默认值，还有的的则可以为空。

举例来说，有这么一个结构体

```go
type Pool struct {
	Capacity       int32
	ExpiryDuration time.Duration
	Logger         Logger
}
```

- `Capacity`表示池容量，不能为空且没有默认值（可以有，这里只是假设）

- `ExpiryDuration`表示过期时间间隔，不能为空但有默认值

- `Logger`表示日志，可以为空

在这种情况下，由于需要使用默认值，所以直接用字面量初始化是不合适的。一般做法是提供初始化函数，比如如下：

```go
func NewDefaultPool(cap int32) (*Pool,error) {...}

func NewPoolWithLogger(cap int32, logger Logger) (*Pool, error) {...}
```

### 缺点

比较繁琐，并且因为go不支持重载，所以还得使用不同的函数名。

## 解决方案一：Config

### 做法

把可选配置（有默认值或可以为空）的字段提到单独的结构体里

```go
type Config struct {
	ExpiryDuration time.Duration
	Logger         Logger
}
```

然后提供一个初始化函数

```go
func NewPool(cap int32, conf *Config) (*Pool, error) {...}
```

### 缺点

这是比较常见的做法，但是需要额外的结构体，并且要判断`Config`是否为`nil`

## 解决方案二：Builder

### 做法

把结构体包装为一个`Builder`

```go
type PoolBuilder struct {
    Pool *Pool
    Err  error
}

func (pb *PoolBuilder) Create(cap int32) *PoolBUilder {...}

func (pb *PoolBuilder) WithLogger(logger Logger) *PoolBUilder {...}

func (pb *PoolBuilder) Build() (*Pool, error)
```

然后通过链式调用构造结构体

```go
pb := &PoolBuilder{}
p, err := pb.Create(100).WithLogger(logger).Build()
```

### 缺点

这样还是需要额外的结构体，如果直接为`Pool`实现方法，无法很好进行错误处理

## 最佳实践：Functional Options

### 做法

首先声明一个函数类型

```go
type Option func(*Pool)
```

然后声明一组函数

```go
func WithExpiryDuration(t time.Duration) Option {
    return func(p *Pool) {
        p.ExpiryDuration = t
    }
}

func WithLogger(logger Logger) Option {
    return func(p *Pool) {
        p.Logger = logger
    }
}

func NewPool(cap int32, options ...func(*Pool)) (*Pool, error) {
    p := &Pool{
        Capacity: 		cap
        ExpiryDuration: time.Hour //默认值
    }
    
    for _, option := range options {
        option(p)
    }
    
    //...
    
    return p, nil
}
```

最后构造方式如下

```go
p, err := NewPool(100, WithLogger(Logger))
```

### 优点

- 直觉式的编程
- 高度的可配置化
- 很容易维护和扩展
- 自文档
- 对于新来的人很容易上手
- 没有什么令人困惑的事（是nil 还是空）

OK，吹完了，收工
