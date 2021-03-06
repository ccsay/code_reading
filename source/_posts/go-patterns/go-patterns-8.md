---
title: Go编程模式 - 8-装饰、管道和访问者模式
date: 2021-02-20 18:32:09
categories: 
- 经典品读
tags:
- Go-Programming-Patterns
---

>  注：本文的灵感来源于GOPHER 2020年大会陈皓的分享，原PPT的[链接](https://www2.slideshare.net/haoel/go-programming-patterns?from_action=save)可能并不方便获取，所以我下载了一份[PDF](https://github.com/Junedayday/code_reading/tree/master/doc/Go_Programming_Patterns.pdf)到git仓，方便大家阅读。我将结合自己的实际项目经历，与大家一起细品这份文档。



## 目录

- [装饰模式](#Decoration)
- [管道模式](#Pipeline)
- [访问者模式](#Visitorl)



今天，我会抛开官方的定义，简单介绍一下三种设计模式。

>  后续会有介绍Go语言设计模式Design Patterns的系列，会更具理论性。

## Decoration

代码实例

```go
func decorator(f func(s string)) func(s string) {
	return func(s string) {
		fmt.Println("Started")
		f(s)
		fmt.Println("Done")
	}
}
```

一句话解释：**在函数f前后，添加装饰性的功能函数，但不改变函数本身的行为**。

这种设计模式，对一些被高频率调用的代码非常有用：

1. HTTP Server被调用的handler
2. HTTP Client发送请求
3. 对MySQL的操作

而装饰性的功能，常见的有：

1. 打印相关的日志信息（Debug中非常有用！）
2. 耗时相关的计算
3. 监控埋点



## Pipeline

代码示例

```go
type HttpHandlerDecorator func(http.HandlerFunc) http.HandlerFunc
func Handler(h http.HandlerFunc, decors ...HttpHandlerDecorator) http.HandlerFunc {
    for i := range decors {
        d := decors[len(decors)-1-i] // iterate in reverse
        h = d(h)
    }
    return h
}
```

一句话解释：**用不定参数的特性，将入参中的函数，逐个应用到对象上**

> 看到这里，如果你能想起之前 `Functional Option` 那篇，会发现有这块的影子。

主要应用于： 有多种可选择的配置（对应Field）或处理（对应方法）的复杂对象。

耗子叔在后面又增加了一些用Goroutine+Channel的方式，其实就是讲Channel作为一个管道的承载体。



## Visitor

关于访问者设计者模式，我之前在Kubernetes源码分析中专门分析了源码。今天，我们也简单地过一下。

```go
// 定义访问的函数类型
type VisitorFunc func(*Info, error) error

// Visitor接口设计
type Visitor interface {
	Visit(VisitorFunc) error
}

// 资源对象
type Info struct {
	Namespace   string
	Name        string
	OtherThings string
}

// 将Visitor函数应用到资源对象上
func (info *Info) Visit(fn VisitorFunc) error {
	return fn(info, nil)
}
```

然后看其中一个实现：NameVisitor，其余的也类似，这样就能注入对应的Visitor

```go
type NameVisitor struct {
	visitor Visitor
}

func (v NameVisitor) Visit(fn VisitorFunc) error {
	return v.visitor.Visit(func(info *Info, err error) error {
		// 这里是运行入参中的VisitorFunc，这一块的逻辑有点像pipeline
		err = fn(info, err)
		// NameVisitor自己实现的Visit逻辑
		return err
	})
}
```

当然，Kubernetes中的Visitor还有进一步的封装，包括遇到错误时的处理，这里不细讲，有兴趣的朋友可以看看我对那一篇的分析。

Visitor模式最大的优点就是 `解耦了数据和程序`。回头看Kubernetes的Visitor应用场景，主要是从各种输入源中解析出资源`Info`。这个过程中Info是数据，各类解析方法是资源。

所以，我认为Visitor模式比较适合的是：**目标数据明确，但获取数据的方法多样且复杂**。但由于多层Visitor调用复杂，建议大家可以在外面再简单地封一层，提供常用的几种Visitor组合后的接口，供使用方调用。



> Github: https://github.com/Junedayday/code_reading
>
> Blog: http://junes.tech/
>
> Bilibili：https://space.bilibili.com/293775192
>
> 公众号：golangcoding

