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

## xcode	的工程路径环境变量

*  工程主目录Pods目录
  
```
${PODS_ROOT}
```

* build 输出目录

```
$CONFIGURATION_BUILD_DIR

比如一个工厂里嵌套了 videolib动态库
那么build后product的videolib的路径就在
"$CONFIGURATION_BUILD_DIR/videolib.framework"

```

* 假设有个HelloWorld的工程,在 `~/Projects/Learn Objective-C `目录

```
$(SRCROOT) / $(PROJECT_DIR) 基本没啥区别，都是指向*.xcodeproj所在的路径

PROJECT = HelloWorld

PROJECT_DIR =~/Projects/Learn Objective-C/HelloWorld

PROJECT_FILE_PATH =${PROJECT_DIR}/HelloWorld.xcodeproj

PROJECT_NAME = HelloWorld
```