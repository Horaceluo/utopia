---
title: "Linux Namespace"
date: 2022-03-19T09:25:05+08:00
draft: true
isCJKLanguage: true
categories:
- 容器原理
tags:
- 容器
- linux
---
# MOUNT_NAMESPACE

Mount namespace可以将进程的挂载分开，由此可以实现在不同命名空间里的进程都可以看到不一样的文件目录结构。

初始化的挂载列表会根据命名空间的不同创建方式有所不同。规则如下：

- 使用clone方式来创建的话，子命名空间的挂载列表会是父级命名空间的副本
- 使用unshare方式来创建的话，子命名空间的挂载列表是当前调用者的命名空间副本

# 共享树

从挂载命名空间的实际中，发现在某些情况下，隔离得太彻底了。例如，当挂载了一个新的磁盘之后，需要每个命名空间都需要重新挂载一次。为解决这个问题，从Linux2.6.15开始，增加了共享树得特性。通过这个特性，可以完成不同命名空间的挂载和卸载事件的自动传播、控制。

每个挂载都可以被标记为以下几种传播类型：

- MS_SHARED: 挂载的事件和卸载的事件会在同一个peer group内传播
- MS_PRIVATE：没有peer group，挂载的事件也不会传播，也不会接受到其他挂载、卸载事件
- MS_SLAVE： 共享peer group的挂载和卸载事件会传播到当前挂载中，但是当前的挂载和卸载不会传播到其他的peer
- MS_UNBINDABLE: 跟MS_PRIVATE差不多，但是多了不能绑定挂载的限制