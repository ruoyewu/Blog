---
title: Android 自定义layout实操
date: 2018-01-08 00:00:00
tags: android
---

## 自定义 Layout

在 Android 的应用开发过程中，自定义 layout 是一个很重要的部分，比如说各个页面之间切换时候的动效，虽然 Android 系统本身随着版本的一代代更迭，提供给开发者的可能越来越大，如目前 Android 开发过程中经常会遇到的 [material design](https://developer.android.com/design/material/index.html) 设计，通过使用Android 官方推出的一系列控件，开发者可以很简单地将自己的 App 开发地很有美感，但是，毕竟这些也只是一些最基本的控件，虽然能够满足大多数人的需求，但对于一些定制化的需求，光有这些还是不够的，比如说很多手机里面都有的滑动页面返回的功能，就是需要自定义一个页面 Layout 来完成这个功能。

自定义此 Layout 需要使用到的知识点有：监听触摸事件