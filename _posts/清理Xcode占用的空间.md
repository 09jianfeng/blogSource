---
title: 清理Xcode占用的空间
date: 2016-08-10 16:22:18
tags: [Xcode]
---

## 大小从大到小排序开始清理

首先进入 Xcode的用户文件目录 `~/Library/Developer/Xcode/`

1、`iOS DeviceSupport` 占用巨大(通常会有几十G),这里会缓存你编译过的所有设备的 Symbols。 每个设备占用差不多1G。 可以全删

2、 `Installs` 存放的是安装过的模拟器，适当删减

3、`DerivedData` 编译过的应用缓存。 可以全删

4、`Archives` 可以全删，缓存的是你打包ipa

