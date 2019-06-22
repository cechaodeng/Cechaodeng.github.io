---
layout:     post
title:      Spring Boot读取配置文件过程解析
subtitle:   源码解读
date:       2019-06-22
author:     Kent
header-img: img/post-bg-spring-boot.jpg
catalog: true
tags:
    - Spring Boot
    - 源码
---

# Spring Boot读取配置文件过程解析

## 背景

Spring Boot的配置文件读取文章有很多，但是每一篇文章总结起来都是在将一件事，就是优先级。
你可能会看到如下片段：

![](https://i.loli.net/2019/06/22/5d0de5d34691784115.png)

这个总结非常有用，我们只需要记住常用的几个的优先级，就能把配置文件用好。
但是，没有任何一篇文章能告诉我为什么是这样，所以我打算自己通过阅读源码找到答案。

## 问题

阅读源码是枯燥无味的，所以我一般会选择带着问题去找答案。
为了快速弄清原理，我们不需要把场景设置得太复杂，就用最简单的配置文件结构，如图所示。

![](https://i.loli.net/2019/06/22/5d0de83e3606688492.png)

我们在application.yml里面配置了spring.profiles.active:staging
所以，根据对Spring Boot的常规里面，会先去读取application.yml，然后根据spring.profiles.active去读取application-staging.yml。
于是就有了一下几个问题：

+ 一、Spring Boot为什么会先去读取application.yml,application-staging.yml不是优先级更高吗？
如果是先读application-staging.yml的，那Spring Boot怎么知道是staging,而不是uat？
+ 二、假如application-staging.yml里面有spring.profiles.active:uat配置项会生效吗？
+ 三、到底哪些位置可以放profiles。
+ 四、假如application.yml和application-staging.yml里面有重复的配置项会怎样处理，为什么？

## 过程

整个过程可以概括为两步：

+ 初始化配置文件名称
+ 找到配置文件位置并读取配置文件

this.profiles就是存放已经读取的配置文件的，第一次读取的时候是size()=0的，
所以先add(null),然后再判断size()==1，如果是的话，就说明是第一次读取文件，就取this.environment.getDefaultProfiles()。
这个值就决定了Spring Boot会先去读取application.yml。

等第一次的application.yml已经读取完了，就会第二次走到这里来，首先会getProfilesActivatedViaProperty()，
然后addActiveProfiles(activatedViaProperty)，就把application-staging.yml也add到this.profiles中了，
并且把this.activatedPro设置为true。

所以，到这里，我们的问题一就有了答案。Spring Boot会先去读取application.yml，因为是默认配置好的，也可以人工干预这个配置。
并且在applicaton.yml中的spring.profiles.active配置项中得到下一个配置文件是staging。

![](https://i.loli.net/2019/06/22/5d0deee47d93247695.png)
![](https://i.loli.net/2019/06/22/5d0def7a4293758950.png)

读取application-staging.yml的时候，也会读取到里面的spring.profiles.active这个配置，
但是这个时候会在这里会对当前profile的active状态进行判断，如果当前已经是active，就不会将这里读到的配置放到this.profiles中，
通过这里可以看出来，Spring Boot只能在默认的配置文件中配置spring.profiles.active这个参数。
我这里是application.yml。

![](https://i.loli.net/2019/06/22/5d0df3ca8197393887.png)

所以，我们第二个问题也有了答案。application-staging.yml中的spring.profiles.active配置项不会生效，
Spring Boot 只支持在默认配置文件中配置这一项。

继续我们的第三个问题，我们对load()方法进行调试，得到结果如图：

![](https://i.loli.net/2019/06/22/5d0df71166c1837279.png)
![](https://i.loli.net/2019/06/22/5d0dff9e0bfd457164.png)

所以，我们第三个问题也有了答案，默认情况下是会读取图中的四个位置的配置文件。

关于第四个问题,我们可以看到前面读取的配置文件都会放在一个集合里面，当需要配置的时候，则根据优先级读取。

## 总结

这里代码并不复杂，逻辑也很简单，但是出于好奇，正好这段时间在做工作交接， 又有一些时间，就把这块的逻辑总结一下吧。