---
layout: post
date: 2016-11-27T13:49:16+08:00
title: Linux 内核系列－进程通信
category: 读书笔记
---

本系列文章为阅读《现代操作系统》和《Linux 内核设计与实现》所整理的读书笔记，源代码取自 Linux-kernel 2.6.34 版本并有做简化。

地址空间 
 

APUE 7.6

一个 C 程序包含以下部分：

文本段：