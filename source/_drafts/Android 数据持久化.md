---
title: Android 数据持久化
date: 2018-05-16 23:00
tags:
	- android
---

## 数据持久化

Android 应用开发中关于数据持久化，有以下几种方式：

1. SharedPreference
2. SQlite
3. ContentProvider
4. 上传到服务器
5. 本地文件存取

不过对于计算机上的数据持久化来说，归根结底还是以写文件的方式写到计算机的磁盘上，才能使得数据被保存而不会随着断电而消失，