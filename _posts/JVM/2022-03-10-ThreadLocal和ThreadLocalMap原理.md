---
title: "ThreadLocal和ThreadLocalMap原理"
subtitle: ""
layout: post
author: "Geooo"
header-img: ""
header-style: text
tags:
  - ThreadLocal
  - Java
  
---
https://zhuanlan.zhihu.com/p/158033837
先mark一下，后面再写笔记，最主要理解ThreadLocal产生内存泄露的原因 

- ThreadLocalMap 是线程私有的，生命周期与线程相同
- ThreadLocal对应的value是存在ThreadLocalMap的value中
- ThreadLocalMap的key是ThreadLocal本身，是弱引用(weak reference)，ThreadLocal在没有外部对象强引用的时候，会在下次GC的时候被回收（内存泄露的主要问题），此时因为value值是强引用的，key被回收之后剩下个value在ThreadLocalMap里面，无法通过调用remove方法进行删除，如果这个线程的生命周期很长，这些value值一直占用着内存，就会产生内存泄露。
- JDK1.8之后对ThreadLocal进行了优化，在每次 get()，set()，remove()的时候都会去检查ThreadLocalMap中是否存在key 为null的value，如果有则将其清理掉，让GC去回收。

![](https://pic3.zhimg.com/80/v2-a39ad53e2dff823223d5e25dab26ce96_720w.jpg)
