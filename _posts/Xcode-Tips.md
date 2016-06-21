---
title: Xcode Tips
date: 2016-06-06 15:49:17
categories: [Xcode,工具]
tags: [Xcode,工具]
---

## Xcode的一些小技巧

* 展开某个.m文件的所有宏

```
在Xcode中，选择Product -> Perform Action -> Preprocess "xxxxxx.m"
```

* block 循环引用提示
在block中使用实例变量时请小心谨慎。这也许会导致block捕获一个self的强引用。你可以打开一个编译警告，当发生这个问题时能提醒你。

```
在项目的build settings中搜索“retain”，找到 Implicit retain of 'self' within blocks 设置成 YES
```

* 描述文件 .mobileprovision 

provision profile（描述文件）包含了certificate, app id, device list, entitlements等，这些条件综合起来，限制了一个指定app id的app在有效期内如何运行在指定的设备以及一些设备能力。

```
打开一个.mobileprovision文件，执行：
security cms -D -i xxx.mobileprovison
或
openssl smime -in /path/to/your.mobileprovision -inform der -verify
可以看到包含在文件内的entitlements信息
```