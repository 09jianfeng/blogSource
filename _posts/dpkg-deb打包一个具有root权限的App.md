---
title: iOSOpenDev与dpkg-deb打包一个具有root权限的App
date: 2016-01-11 09:59:31
categories: [iOS,逆向]
tags: [iOS,逆向]
---

参考：狗神的帖子 <http://bbs.iosre.com/t/run-an-app-as-root-on-ios/239>

# dpkg

## 安装Macports

 [下载对应系统的Macports](http://www.macports.org/install.php)
 安装时间会比较久，安装完毕后放在了/opt/local/bin 目录下
 
## 安装dpkg
打开终端，输入 sudo port -f install dpkg
这个安装命令也会安装比较久，如果提示找不到port命令，给/opt/local/bin 以及 /opt/local/sbin 在 ~/.bash_profile 中配置一下环境变量。

PS:按照安装 theos的教程，生成的那个dpkg-deb。打包不了下面的要安装在/Application的应用。要用Macports来安装dpkg-deb。 原因我还没有深究

## 生成必要的目录
* Applications

```
这个目录下放你要安装在/Application 目录下的App。比如  XXX.app (用开发appStore的流程开发的app)
```

* DEBIAN

```
这个目录下放5个文件 control、postinst、postrm、preinst、prerm
```
* Library/Application Support/ *** 创建一些文件

```
这个文件下的文件，安装的时候会放入相应的目录
```

#### DEBIAN目录文件解释
* control

工程配置文件

```
Package: control.packagename
Name: control.packagename
Version: 1.0
Description: 
Section:
Depends: firmware (>= 5.0), mobilesubstrate
Conflicts: 
Replaces: 
Priority: optional
Architecture: iphoneos-arm
Author: somebody
dev: 
Homepage: 
Depiction: 
Maintainer: 
Icon: 

```

* preinst

Debian软件包(".deb")解压前执行的脚本, 为正在被升级的包停止相关服务,直到升级或安装完成。 
(成功后执行 'postinst' 脚本)。

* postinst

主要完成软件包(".deb")安装完成后所需的配置工作的脚本.
通常, postinst 脚本要求用户输入, 和/或警告用户如果接受默认值, 应该记得按要求返回重新配置这个软件。
一个软件包安装或升级完成后，postinst 脚本驱动命令, 启动或重起相应的服务。

例如给itunesstored、keychain执行权限

```
#!/bin/sh
chmod +s /Applications/downloadipa.app/itunesstored
chmod +s /Applications/testxxx.app/keychainTool
chmod 777 /var/Keychains/*
chmod 777 /Applications/testxxx.app/testxxx
```

* prerm
停止一个软件包的相关进程, 要卸载软件包的相关文件前执行的脚本。


* postrm
修改相关文件或连接, 和/或卸载软件包所创建的文件。
当前的所有配置文件都可在 /var/lib/dpkg/info 目录下找到, 与 foo 软件包相关的命名以 "foo" 开头,以 "preinst", "postinst", 等为扩展。
这个目录下的 foo.list 文件列出了软件包安装的所有文件。
Debian里用apt-get安装或卸载软件时，会常发生前处理或后处理的错误，这时只要删除 对应的脚本文件，重新执行安装或卸载即可。

## 用dpkg-deb打包 .deb 并且安装

```
dpkg-deb -b ./ mydeb.deb
```
把 mydeb.deb拖进 iOS的根目录/ 。 执行 `dpkg -i mydeb.deb`即可安装。

安装完后运行`su mobile -c uicache` 刷新UI缓存

如果安装的过程中出现这个错误：

```
dpkg-deb: file `dazhong.deb' contains ununderstood data member data.tar.xz     , giving up
dpkg: error processing
```
打包deb的时候要用这个命令打包

```
dpkg-deb -Z gzip -b ./ mydeb.deb
```

# 提权
## 步骤
1、postinst文件配置：

```
chmod +s /Applications/aatext.app/aatext
chown  root:wheel /Applications/aatext.app/aatext
```

2、准备一个bash脚本。添加到工程

```
C=/${0}
C=${C%/*}
exec "${C:-.}"/aatext
```


3、修复info.plist文件

```
Executable file 值设置为 bash
```

然后再按照上面说的dpkg-deb打包成deb，安装。就是一个具有root权限的app了

## 原理
