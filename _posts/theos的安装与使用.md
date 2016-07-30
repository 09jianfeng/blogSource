title: theos的安装以及使用
date: 2015-04-5 17:29:51
categories: [iOS,逆向]
tags: [iOS,逆向]
---
### theos的安装 《iOS逆向开发 第2版》这本书里面的介绍
* 1、从github上下载theos到/opt/theos目录

```
export THEOS=/opt/theos
sudo git clone git://github.com/DHowett/theos.git $THEOS
```
* 2、配置ldid， ` ldid是专门用来签名iOS可执行文件的工具，用以在越狱iOS中取代Xcode自带的codesign。从http://joedj.net/ldid下载ldid，把它放在“/opt/theos/bin/”下，然后用以下命令赋予它可执行权限：`

```
sudo chmod 777 /opt/theos/bin/ldid
```

* 3、配置CydiaSubstrate,首先运行Theos的自动化配置脚本，操作如下：

```
snakeninnysiMac:~ snakeninny$ sudo /opt/theos/bin/bootstrap.sh substrate
Password:
Bootstrapping CydiaSubstrate...
 Compiling iPhoneOS CydiaSubstrate stub... default target?
 failed, what?
 Compiling native CydiaSubstrate stub...
 Generating substrate.h header...
 ```
此处会遇到Theos的一个bug，它无法自动生成一个有效的libsubstrate.dylib文件，需要手动操作。解决方法很简单：首先在Cydia中搜索安装“CydiaSubstrate”.然后用iFunBox或scp等方式将iOS上的“/Library/Frameworks/CydiaSubstrate.framework/CydiaSubstrate”拷贝到OSX中，将其重命名为libsubstrate.dylib后放到“/opt/theos/lib/libsubstrate.dylib”中，替换掉无效的文件即可。

* 4、配置dpkg-deb,`deb是越狱开发安装包的标准格式，dpkg-deb是一个用于操作deb文件的工具，有了这个工具，Theos才能正确地把工程打包成为deb文件。`
从https://raw.githubusercontent.com/DHowett/dm.pl/master/dm.pl下载dm.pl，将其重命名为dpkg-deb后，放到“/opt/theos/bin/”目录下，然后用以下命令赋予其可执行权限：

```
snakeninnysiMac:~ snakeninny$ sudo chmod 777 /opt/theos/bin/dpkg-deb
```

* 5、配置Theos NIC templates

```
Theos NIC templates内置了5种Theos工程类型的模板，方便创建多样的Theos工程。除此以外，还可以从https://github.com/DHowett/theos-nic-templates/archive/master.zip获取额外的5种模板，下载后将解压得到的5个.tar文件复制到“/opt/theos/templates/iphone/”下即可。
```

### 开始工程
* 第一步，输入命令 `/opt/theos/bin/nic.pl`
* 我们一般是开发tweak。选择9
* 输入tweak的工程名
* 输入deb包的名字（bundle identifier）
* 输入tweak作者名称
* 输入“MobileSubstrate Bundle filter”，也就是tweak作用对象的bundle identifier
* 输入需要重启的进程名


### theos一些配置文件的作用
* makefile,`makefile 文件制定工程用到的文件、框架、库等信息，将整个过程自动化`

```
> include theos/makefiles/common.mk 
  固定写法【不要更改】
 
> TEWAK_NAME = iOSREProject 
  tewak的名字，即创建工程的时候你设置的名字,跟control文件中的name对应。【不要更改】

> iOSREProject_FILES=Tweak.xm 
  tweak包含的源文件（不包括头文件），多个文件间以空格分隔。如iOSREProject_FILES=Tweak.xm Hook.xm New.x ObjC.m ObjC++.mm  【按需更改】

> include $(THEOS_MAKE_PATH)/teak.mk 
   根据不同的Theos工程类型，通过include命令指定不同的.mk文件；在逆向工程初级阶段，我们开发的一般是Application、Tweak和Tool三种类型的程序，它们对应的.mk文件分别是application.mk、tweak.mk和tool.mk，【可以按需更改】。

> after-install::
   install.exec "killall -9 SpringBoard" 
   在tweak 安装之后杀掉SpringBoard进程，好让CydiaSubstrate在进程启动时加载对应的dylib。
   
> ARCHS=armv7 arm64 
   指定处理器架构

> TARGET=iphone:8.1:8.0 
   指定采用8.1版本的sdk，且发布对象为iOS8.0及以上版本。也可以把8.1替换为latest，指定以Xcode附带的最新版本的sdk编译。

> iOSREProject_FRAMEWORKS = UIKit CoreTelephony CoreAudio 
   导入framework

> iOSREProject_PRIVATE_FRAMEWORKS = AppSupport ChatKit IMCore 
   导入私有API，如果遇到有些私有api在不同的系统版本上不一样的情况。需要在代码中自己做特判

> iOSREProject_LDFLAGS=-lx 
   Theos采用GNU Linker来链接Mach-O对象，包括.dylib、.a和.O。-lx代表链接libx.a或libx.dylib，即给“x”加上“lib”的前缀，以及“.a”或“.dylib”的后缀；如果x是“y.o”的形式，则直接链接y.o，不加任何前缀或后缀 例如： iOSREProject_LDFLAGS=-lz --lsqlite.3.0 --dylib1.o   -lz表示libz.dylib。--lsqlite.3.0代表libsqlite3.0.dylib  --dylib1.o代表dylib1.o。
        
```

* Tweak.xm
Tweak.xm是默认的源文件。“xm”中的x代表这个文件支持logos语法，如果后缀名单独一个x说明支持logos和C/C++语法，与m和mm的区别类似。`logos语法`

```
> %hook
  指定需要hook的class，必须以%end结尾。如下:
  %hook SpringBoard
 //勾住SpringBoard类里的_menuButtonDown:函数，先执行NSLog然后再执行函数原来的操作
 - (void)_menuButtonDown:(id)down
{     
        NSLog(@"You've pressed home button.");
        %orig; // call the original _menuButtonDown:
}
%end

> %log
该指令在%hook内部使用，将函数的类名、参数等信息写入syslog，可以以%log([(<type>)<expr>,...])的格式追加其他打印信息，例如 %log((NSString*)@"hello world");

> %orig
 该指令在%hook内部使用，执行被勾住（hook）的函数的原始代码。还可以用%orig更改原始函数的参数。例如：
 %hook SBLockScreenDateViewController
- (void)setCustomSubtitleText:(id)arg1 withColor:(id)arg2
{
      %orig(@"iOS 8 App Reverse Engineering", arg2);
}
%end
 这样一来就更改了锁屏界面的文字显示

> %group 
  用来包住%hook的组，用%init初始化的组会生效，跟%end成对使用。未归并到组的一律归并到了 %group_ungrouped中

> %init 
  改指令用于初始化某个%group，必须在%hook或者%ctor内调用。如果带参数，则初始化指定的group，如果不带参数，则初始化_ungrouped。 只有调用了%init，对应的%group才能起作用。


> %ctor 
  tweak的构造体，完成初始化的工作；如果不显示定义，Theos会自动生成一个%ctor,并在其中调用 %init(_ungrouped)。自动生成的内容如下：
 %ctor
 {
 	%init; //相当于%init(_ungrouped);
 }
%ctor一般用来初始化%group，以及进行MSHookFunction等操作。

> %new
  在%hook内部使用，给一个现有的class添加新函数，功能与class_addMethod相同。他的用法如下：
%hook SpringBoard
%new
- (void)namespaceNewMethod
{
      NSLog(@"We've added a new method to SpringBoard.");
}
%end

> %c
  该指令的作用等同于objc_getClass 或者 NSClassFromString。在%hook或者%ctor内使用。

```
[更多的logos语法](http://iphonedevwiki.net/index.php/Logos)

* control文件
  control文件记录了deb包管理系统所需的基本信息，会打进包里，cydia的应用介绍那里看到的信息就是这些。
  
  ```
  这里重点关注一下Depends字段：depends字段用于描述这个deb包的依赖，就是程序运行需要的条件.例如：`Depends:mobilesubstrate,firmware(>=8.0)`。代表 必须运行在8.0系统以上，且必须安装CydiaSubstrate，才能正常运行这个tweak，可以按需更改
  ````
  
* xxx.plist
  
  这个plist文件的作用和App中的Info.plist类似，它记录了一些配置信息，描述了tewak的作用范围。
  
 ```
 Filter 下是一系列array，可以分为三类
> Bundles指定若干bundle为tweak的作用对象
> class 指定若干class为tweak的作用对象
> Executables，指定若干可执行文件为tweak的作用对象
当Filter下有不同类的array时，需要添加一个“Mode:Any”键值对。
 ```
 
### Theos 编译
 开发完后，在Theos工程目录下运行`make`命令来编译Theos工程。运行了make后目录下会多一个obj文件夹，里面有个.dylib文件。就是tweak的核心
 
### Theos 打包
 打包使用`make package`命令，其实是先执行make命令编译，然后再执行`dpkg-deb`命令。执行了`make package`命令后，会生成一个XXX.deb的文件，这就是可以最终发布的安装包了。
 [更多deb的信息](http://www.debian.org/doc/debian-policy)

### 清理

```
make clean
rm *.deb
```
 
### 安装
* 编辑makefile

```
在makefile的头加上这些代码
THEOS_DEVICE_IP = localhost -p 2222  //如果不是用usb连接调试，则把localhost -p 2222替换为移动设备的地址
ARCHS = armv7 arm64
TARGET = iphone:latest:8.0
```

* 在工程目录下敲命令

```
make package install
```

或者用dpkg -i来安装。把deb包放到移动设备系统目录，执行 dpky -i xx.deb。安装deb包。

### 如果 make package install出错
用下面的命令打印错误看看

```
make package messages=yes
```

## 免去远程链接的密码输入
* 1、找到 `~/.ssh/known_hosts` 文件，打开删掉 `localhost 2222 ssh-rsa` 那一行
* 2、生成authorized_keys

```
snakeninnysiMac:~ snakeninny$ ssh-keygen -t rsa
Generating public/private rsa key pair.
Enter file in which to save the key (/Users/snakeninny/.ssh/id_rsa): 
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /Users/snakeninny/.ssh/id_rsa.
Your public key has been saved in /Users/snakeninny/.ssh/id_rsa.pub.
……
snakeninnysiMac:~ snakeninny$ cp /Users/snakeninny/.ssh/id_rsa.pub ~/authorized_keys
```
* 3、配置iOS

```
FunMaker-5:~ root# ssh-keygen
Generating public/private rsa key pair.
Enter file in which to save the key (/var/root/.ssh/id_rsa): 
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /var/root/.ssh/id_rsa.
Your public key has been saved in /var/root/.ssh/id_rsa.pub.
……
FunMaker-5:~ root# logout
Connection to iOSIP closed.
snakeninnysiMac:iosreproject snakeninny$ scp ~/authorized_keys root@iOSIP:/var/root/.ssh
The authenticity of host 'iOSIP (iOSIP)' can't be established.
RSA key fingerprint is 75:98:9a:05:a3:27:2d:23:08:d3:ee:f4:d1:28:ba:1a.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added 'iOSIP' (RSA) to the list of known hosts.
root@iOSIP's password: 
authorized_keys                            100%  408    0.4KB/s   00:00
```
