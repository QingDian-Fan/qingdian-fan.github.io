---
title: Material Design自定义Behavior
tags: 自定义View
permalink: android-source/dc-view-4
key: android-source-dc-view-4
sidebar:
  nav: android-source
---

CoordinatorLayout简介
CoordinatorLayout中文翻译为协调布局，它可以协调子布局之间交互，当触摸改变孩子View的时候会影响布局从而产生联动效果，产生的根源在于Behavior类
CoordinatorLayout自己并不控制View，所有的控制权都在Behavior这个类
系统里Behavior有FloatingActionButton.Behavior和AppBarLayout.Behavior等等；我们平时用户那些联动都是靠系统定义的Behavior来实现
系统里面Behavoir有些实现了复杂的控制功能。但是毕竟有限，我们可以通过自定义的方式来实现自己的Behavior

<!--more-->













