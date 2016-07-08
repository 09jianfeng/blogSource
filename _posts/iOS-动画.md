---
title: iOS 动画
date: 2016-07-06 15:00:04
tags: [iOS,动画]
---

# 贝塞尔曲线

# CADisplayLink
简单的理解，CADisplayLink就是一个定时器。 每隔1/60刷新一次屏幕。使用的时候，我们要把它添加到一个runloop中，并且给他绑定selector、target，才能在屏幕以1/60秒刷新的时候实时调用绑定的方法。

```
self.displayLink = [CADisplayLink displayLinkWithTarget:self selector:@selector(displayLinkAction:)];
 
[self.displayLink addToRunLoop:[NSRunLoop mainRunLoop] forMode:NSDefaultRunLoopMode];”

```





