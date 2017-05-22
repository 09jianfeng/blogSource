---
title: xcode工程嵌入工程
date: 2017-02-22 15:19:49
tags: [xcode,iOS]
---

注意点：

1、先创建静态工程 .a工程 比如说目录是
 ~/testlib
 
2、在.a工程目录下创建demo路径并且创建demo工程。目录是：
~/testlib/demo/testlibdemo

3、打开demo工程，把静态库工程拖进demo工程。设置

```
1、testlibdemo的工程要依赖于testlib。(Build Phases -> Target Dependencies -> add testlib)。
2、设置库引用到testlib, Build Phases -> Link Binary With Libraries -> add libtestlib.
3、header应用。比如说testlib静态库在目录~/testlib/include目录下放公开的头文件，想要在demo工程中引用到这写头文件。则必须设置 buildSetting -> Header Search Paths -> add -> $(SRCROOT)/../../include  链接到include文件夹就可以连接到头文件了。
```