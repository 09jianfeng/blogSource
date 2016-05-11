title: dumpdecrypted砸壳教程
date: 2016-04-5 17:29:51
tags: [iOS,逆向]
---
### 准备步骤
* [dumpdecrypted下载地址](https://github.com/conradev/dumpdecrypted)
* 确认移动设备的的系统版本，比如我的设备是iOS8.0.2



### 修改dumpdecrypted的压缩包解压后的makefile的文件配置
  
* dumpdecrypted必须使用与iOS版本相同的SDK版本编译，才能正常工作。打开“终端（Terminal）”，输入 `xcrun --sdk iphoneos --show-sdk-path.` 我的输出是
 ` /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/SDKs/iPhoneOS9.2.sdk`
  设备与xcode的sdk不是同一个版本则需要修改makefile文件
  
  ```
  SDK=`xcrun --sdk iphoneos --show-sdk-path`
  为
  SDK=/Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/SDKs/iPhoneOS8.4.sdk
  
  （这个iPhoneOS8.X.sdk需要去网上下载旧版的xcode，然后提取出旧版的sdk,提取方法见下面解说，然后放到新版sdk相同的目录下,上面SDK=/Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/SDKs/iPhoneOS8.X.sdk   //8.4的sdk向下兼容，即8.4的兼容8.0 8.1 ....
的路径就是下载的sdk的路径）
  ```

### 下载旧版的xcode，提取旧版的iOS sdk
[旧版的xcode下载地址](https://developer.apple.com/downloads/index.action)
在这个目录下面，把sdk复制出来 Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/SDKs/iPhoneOS8.X.sdk

### 生成dumpdecrypted.dylib文件
* 配置框架，还要对makefile以及.c文件进行配置。ipod touch 5的架构是armv7
  
  1、`makefile`文件的`GCC_UNIVERSAL=$(GCC_BASE) -arch armv7 -arch armv7s -arch arm64` 这行要去掉`-arch armv7s -arch arm64`. 

  2、dumpdecrypted.c第76行的`if (lc->cmd ==LC_ENCRYPTION_INFO || lc->cmd == LC_ENCRYPTION_INFO_64)`更改为`if(lc->cmd == LC_ENCRYPTION_INFO)`


### 把dumpdecrypted.dylib传到设备上去
把dumpdecrypted.dylib文件传入到移动设备上去， /var/tmp目录

```
scp -P 2222 dumpdecrypted.dylib root@localhost:/var/tmp  //用usbmuxd链接的方式
scp dumpdecrypted.dylib root@192.168.3.24:/var/tmp   //无线远程链接的方式传文件
```

### 在设备上定位需要被砸壳的文件
可以用ifunbox或者直接用命令找
find -name discover
找到完整的文件目录为
`/private/var/mobile/Containers/Bundle/Application/480015C5-D3FD-46D0-A179-9B8FAA2F6CE4/BBGroover.app/BBGroover`

```
或者一开始不知道可执行文件名是什么，那么可以先关掉所有app，然后打开你要检查的app。然后运行`ps -e`命令查看所有在运行的进程。因为只开了一个store app，所以唯一含有/var/mobile/Containers/Bundle/的那个路径就是我们要找的可执行文件的路径
```


### 开始砸壳
cd到dumpdecrypted.dylib所在的目录，例如我的是/var/mobile。然后输入命令 `DYLD_INSERT_LIBRARIES=dumpdecrypted.dylib /private/var/mobile/Containers/Bundle/Application/480015C5-D3FD-46D0-A179-9B8FAA2F6CE4/BBGroover.app/BBGroover`

报错

```
testipod:/var/tmp root# DYLD_INSERT_LIBRARIES=dumpdecrypted.dylib /private/var/mobile/Containers/Bundle/Application/480015C5-D3FD-46D0-A179-9B8FAA2F6CE4/BBGroover.app/BBGroover
dyld: could not load inserted library 'dumpdecrypted.dylib' because no suitable image found.  Did find:
	dumpdecrypted.dylib: stat() failed with errno=1

Trace/BPT trap: 5
```

这是因为沙盒权限问题导致的，需要把dumpdecrypted.dylib文件放到对应app的沙盒Documet目录下。然后cd到这个目录。再执行上面的那个命令就可以了。

如果遇到`Abort trap: 6`的错误，可能是因为你要砸的应用在编译的时候加了restricted Mach-O 中加入了 restrict 字段，不允许动态库的插入.


### XXXX.decrypted就是砸壳完后的可执行文件了。就可以用classdump等工具去导出了




