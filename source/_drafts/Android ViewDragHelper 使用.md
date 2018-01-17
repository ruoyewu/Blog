---
title: Android ViewDragHelper 使用
tags: android
---

## 1. 概述

官方介绍：

>ViewDragHelper is a utility class for writing custom ViewGroups. It offers a numberof useful operations and state tracking for allowing a user to drag and reposition views within their parent ViewGroup.

ViewDragHolper 是官方提供的一个用于自定义 ViewGroup 的实用类，在 v4 包里面。一般在自定义 ViewGroup 的时候，需要在 onIntercrptTouchEvent 和 onTouchEvent 中