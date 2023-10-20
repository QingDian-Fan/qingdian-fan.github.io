---
title: View的Touch事件分发
tags: 自定义View
permalink: android-source/dc-view-5
key: android-source-dc-view-5
sidebar:
  nav: android-source
---

Android中的View事件分发机制是指当用户触摸屏幕时，系统如何将触摸事件分发给各个View，并由它们来处理事件的过程。事件分发机
制主要包括三个部分：事件的产生、事件的分发和事件的处理。

事件的产生：当用户触摸屏幕时，系统会产生一个MotionEvent对象，该对象包含了触摸点的坐标、触摸的时间、触摸的压力等信息

事件的分发：事件分发是由ViewGroup来完成的，它会将事件分发给它的子View，并根据子View的返回值来决定是否继续分发事件。如果子View处理了事件，那么事件就不会再传递给其他View

事件的处理：事件的处理是由View来完成的，它会根据事件的类型来调用相应的回调方法，如onTouchEvent()、onClickListener()等。

总的来说，Android的View事件分发机制是一个复杂的过程，需要开发者深入理解和掌握，才能编写出高效、稳定的应用程序。

<!--more-->





![blog-logo](https://raw.githubusercontent.com/QingDian-Fan/ImageRepository/master/images/blog-logo-20231013.png)









