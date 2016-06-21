---
title: '__bridge,__bridge_transfer,__bridge_retained'
date: 2016-05-09 12:59:30
categories: [iOS,Objective-C]
tags: [iOS,Objective-C]
---

## 什么时候用到__bridge关键字
在ARC模式中，如果需要进行Core Foundation与Objective-C类型之间的转换。需要用到__bridge桥接。

```
id obj = [[NSObject alloc] init];
 
void *p = (__bridge void *)obj;
 
id o = (__bridge id)p;
```
将Objective-C的对象类型用 \_\_bridge 转换为 void* 类型和使用 \_\_unsafe_unretained 关键字修饰的变量是一样的。

## \_\_bridge_retained  从  Objective-C -> Core Foundation

```
id obj = [[NSObject alloc] init];
 
void *p = (__bridge_retained void *)obj;
```
相当于下面的实现

```
id obj = [[NSObject alloc] init];
void *p = obj;
[(id)p retain];
```
所以用完了这个对象后，需要CFRelease一下对象


## \_\_bridge_transfer  从 Core Foundation -> Objective-C

```
id obj = (__bridge_transfer )p;
```
相当于

```
id obj = (id)p;
[obj retain];
[(id)p release];
```

ARC会管理内存