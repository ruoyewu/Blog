---
title: Android 数据持久化
date: 2018-05-16 23:00
tags:
	- android
---

对于计算机上的数据持久化而言，归根结底就是将数据保存在磁盘上，使得即使在断电的情况下数据仍然能够留存，不同的是使用不同存储方式时数据存储结构的不同，在 Android 开发中常用的数据持久存储方式大致有以下五个：

1.  SharedPreferences
2.  SQlite
3.  ContentProvider
4.  Socket
5.  本地文件存取

### SharedPreferences

SharedPreferences 是 Android 中一种比较轻量级的数据存储方式，其本质是通过 键-值 的方式将开发者要存储的数据保存在 xml 文件中，不过为了提高频繁存取的效率，

### SQLite

### ContentProvider

### Socket

### 文件直接存储