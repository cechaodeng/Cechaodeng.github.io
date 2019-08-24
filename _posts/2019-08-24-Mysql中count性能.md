---
layout:     post
title:      Mysql中count的性能解读
subtitle:   源码解读
date:       2019-08-24
author:     Kent
header-img: img/post-bg-mysql.jpg
catalog: true
tags:
    - MySQL
    - 源码
---

# Mysql中count的性能解读

## 背景

MySQL中count函数大家都会用，但是看到有些同事写count(id)，有些同事写count(*)，还有一些写count(1),
那到底哪种才是最好的呢，我们一起来看看他们的原理。

## 原则

首先，我们要知道mysql的一个原则：server层要什么，引擎层就返回什么。

## count(id)

基于上面那个原则，server层要的是id，所以这里我们要返回id。
具体的做法是，引擎层遍历最小的的索引树，将符合条件的的行的id返回给server层，server层判断不为空的，则按行累加。

`这里你可能会说id肯定不是空的，为什么还要判断，然鹅mysql的代码就是这么写的`

## count(1)

还是基于server层要什么，引擎层就给什么的原则，具体做法是，
遍历整张表（具体走什么索引有待考证），将符合条件的行不取值直接返回给server，
可以理解为返回了一个空的对象，然后server层按行累加。

这种方式由于不用取值，所以比count(id)是要快的。

## count(字段)

如果该字段有索引，就遍历该字段的索引，如果没有，就遍历主键索引，
取出符合条件的行中的这个字段，返回给server层，server判断不为null，则累加。

## count(*)

具体走什么索引有待考证，MySQL对count(*)做了优化，count(*)不会返回所有字段，也不会取值，
跟count(1)一样，可以理解为返回了一个空对象，server层直接累加即可。

## 总结

count(字段)<count(id)<count(1)≈count(*)

所以，现在你知道该如何使用count函数了吧！

## 附
[xMind地址](https://github.com/cechaodeng/cechaodeng.github.io/blob/master/xmind/mysql-count.xmind)
