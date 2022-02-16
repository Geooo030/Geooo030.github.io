---
title: "Redis进阶知识03-AOF和RDB实现原理"
subtitle: ""
layout: post
author: "Geooo"
header-img: ""
header-style: text
tags:
  - Redis
  
---

# Redis持久化 - AOF

## AOF（Append Only File）是如何实现的

AOF是 **"写后日志"** , 意思是先执行命令，把数据写入内存中，然后再记录日志，写进磁盘。（如下图）
![](https://static001.geekbang.org/resource/image/40/1f/407f2686083afc37351cfd9107319a1f.jpg "Redis AOF操作过程")

传统数据库的日志，例如 redo log（重做日志），记录的是修改后的数据，而 AOF 里记录的是 Redis 收到的每一条命令，这些命令是以文本形式保存的。



>> Redis 收到“set testkey testvalue”命令后记录的日志为例，看看 AOF 日志的内容。其中，“*3”表示当前命令有三个部分，每部分都是由“$+数字”开头，后面紧跟着具体的命令、键或值。这里，“数字”表示这部分中的命令、键或值一共有多少字节。例如，“$3 set”表示这部分有 3 个字节，也就是“set”命令。

![](https://static001.geekbang.org/resource/image/4d/9f/4d120bee623642e75fdf1c0700623a9f.jpg "Redis AOF日志内容")

