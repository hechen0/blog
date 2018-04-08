---
title: xorm对于表名的缓存
date: 2018-03-28T19:30:42+08:00
draft: true
---

最近碰到一个坑

golang中的orm库: xorm

如果不显示传入表名的话，xorm库可以根据你传入的struct主动调用定义的Tablename()方法去推断表名的。

但是这个表名是有缓存的，缓存的结构是map[reflect.Type]string，如果能读到缓存，不会调用Tablename()方法。

最近碰到的坑就是我们在业务层做了分表，导致不能依赖这种隐式推出表名的方法。

对于分表而言，传入的struct是相同的，所以reflect.Type也是相同的，所以只会在缓存不存在的时候调用Tablename()方法。
