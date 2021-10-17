---
title: AndroidManifest.xml中launchMode的含义
date: 2021-10-17 23:41:47
tags:
- Android
---

本文将介绍`AndroidManifest.xml`文件中，`activity`标签的`launchMode`属性。

<!-- more -->

## 背景

Android中，Activity是基于返回栈管理的。这意思是说，当打开一个Activity时，通常会将该Activity入栈，因此这时该Activity将处于栈顶。而Android在设计上，UI始终显示的是位于栈顶的那个Activity。

## 介绍

背景所说的情况，是最简单的一种情况，实际上受到栈中是否有相同类型的实例、该实例是否位于栈顶，还会有不同的表现。



通过`launchMode`可以控制Activity的启动模式（基本等于英文直译了）。



安卓官网对该字段的介绍：https://developer.android.com/guide/topics/manifest/activity-element#lmode。



它有四个可选值：

* `standard`：默认/缺省值。当有新`Intent`启动Activity时，不考虑任何情况，总是**新**实例化一个Activity，并压入栈
* `singleTop`：与`standard`几乎一样，**除了**一种特殊情况：当检测到栈顶的Activity实例与将要创建的Activity**类型相同**时，不会创建新实例，直接使用栈顶实例，调用它的`onNewIntent`方法。
* `singleTask`：和前两者完全不同，栈中**只会存在一个该类型的实例对象**。即，当栈中不存在该Activity实例时，创建并压入栈（与调用者为同一个栈）；否则，弹出栈中位于这个实例前的所有实例，以确保使其位于栈顶
* `singleInstance`：与`singleTask`类似，区别是创建的这个Activity将会位于不同的栈内。即开辟一个新的栈来存放该Activity实例。



## 遗留问题

Q1：安卓系统中，只有一个任务栈吗？如果不是，有多少个？

当然不是。系统中会存在多个栈，每当一个App启动，就会为该App分配一个App的任务栈，然后根据App代码中Activity的launchMode，可能存在多个任务栈。

Q2：如何查看安卓的栈？

`adb shell dumpsys activity | grep -i <your-package-name>`，然后如下图：

![检查App的任务栈](/images/check-android-app-stack.jpg)
