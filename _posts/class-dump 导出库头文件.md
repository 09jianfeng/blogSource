---
title: class-dump 导出库头文件
date: 2015-04-5 17:29:51
categories: [iOS,逆向]
tags: [iOS,逆向]
---

## 1、[class-dump 下载地址](http://stevenygard.com/download/)
安装步骤:

　　　　a) 下载了 dmg 打开并把 class-dump copy 到 /usr/bin/ 目录下;

　　　　b) 修改 class-dump 的权限. `chemod 777`

## 2、[DumpFrameworks.pl 下载地址](https://github.com/shuhongwu/HackSpringDemo/edit/master/DumpFrameworks.pl)
a) 放到任意目录可行。

　　b) 根据自己 framework 的地址，修改路径。

　　c) cd 到  DumpFrameworks.pl 的路径，并执行

```
$ chmod 777 DumpFrameworks.pl
$ ./DumpFrameworks.pl
```