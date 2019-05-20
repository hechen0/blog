---
title: goroutine泄露
date: 2018-04-21T20:17:06+08:00
drafts: true
---

在go语言中，并发是通过goroutine实现。当goroutine永远阻塞或者进入无限循环状态而没有得到正确处理，但启动这些goroutine的函数就返回时，就产生了goroutine泄露。下面我举几个goroutine泄露的例子。

# 写一个没有接收者的channel

假设程序给后端发送了多个请求，取最先返回的请求，其它请求被丢弃的时候。

# 读一个没有发送者的channel

# nil channel

写nil channel会永远阻塞

读nil channel会永远阻塞

# 无限循环


goroutine泄露检测

# runtime.NumGoroutine

# net/http/pprof

# runtime/pprof

# gops

# leaktest

在程序开始和结束分别调用 runtime.Stack 来获取活跃的goroutine，比较前后是否有新的goroutine

