---
title: Cycript 注入测试函数
date: 2016-04-13 17:29:51
tags: [iOS,逆向]
---

`简介：`
安装了Cycript后，可以用Objective-javascript的方式编写代码。可以从MTerminal中执行Cycript，也可以ssh到设备，执行Cycript。支持Objective-C的语法，部分也需要JavaScript的语法来写。详细参考[官网地址](http://www.cycript.org)。
Cycript不是用来编写App，是用来测试函数的功能的工具。一般来说，选择注入哪个进程，要依测试的具体函数而定。

## 启动、关闭Cycript
ssh到设备，输入 `cycript`，出现`cy#`, 即启动cycript。按下`Control+D`退出cycript环境。

## 举例
- 1、 找到SpringBoard
```
toufangde-iPod:~ root# ps -e | grep SpringBoard
10608 ??         2:31.50 /System/Library/CoreServices/SpringBoard.app/SpringBoard
14362 ttys000    0:00.01 grep SpringBoard
toufangde-iPod:~ root#
```
SpringBoard的进程号是14362. 接下来输入
```
cycript -p 10608 或者输入 cycript -p SpringBoard
```
这样cycript就已经运行在SpringBoard里面了。

- 2、用UIAlertView来做个测试，在iOS开发中调用UIAlertView的代码一般是
```
UIAlertView *alertView = [[UIAlertView alloc] initWithTitle:@"iOSRE" message:@"snakeninny" delegate:nil cancelButtonTitle:@"OK" otherButtonTitles:nil];
[alertView show];
[alertView release];
```
转化成Cycript的写法就是
```
cy# alertView = [[UIAlertView alloc] initWithTitle:@"iOSRE" message:@"snakeninny" delegate:nil cancelButtonTitle:@"OK" otherButtonTitles:nil];
#"<UIAlertView: 0x1b4ce310; frame = (0 0; 0 0); layer = <CALayer: 0x170b00f0>>"
cy# [alertView show];
cy# [alerView release];
throw new ReferenceError("Can't find variable: alerView")
cy# [alertView release];
cy#
```
非常简单。

- 3、如果知道一个对象在内存中的地址，可以通过`#`符号来获取这个对象
例如：
```
cy# [[UIAlertView alloc] initWithTitle:@"iOSRE" message:@"snakeninny" delegate:nil cancelButtonTitle:@"OK" otherButtonTitles:nil];
#"<UIAlertView: 0x1b2931b0; frame = (0 0; 0 0); layer = <CALayer: 0x1b28b770>>"
cy# [#0x1b2931b0 show];
cy# [#0x1b2931b0 release];
cy#
```

- 4、如果知道一个对象在进程里面，却不知道他的地址，不能通过`#`来获取他。此时可以试试 `choose`
例如：
```
cy# choose(SBScreenShotter)
[#"<SBScreenShotter: 0x166e0e20>"]
cy# choose(SBUIController)
[#"<SBUIController: 0x16184bf0>"]
```
不过使用choose并不是百发百中的，当他不能返回一个对象给你的时候，就得自己手动寻找了。

- 5、登录iMessage实例
```
先拿到iMessage的登录管理器，命令如下：

FunMaker-5:~ root# cycript -p SpringBoard
cy# controller = [CNFRegController controllerForServiceType:1]
#"<CNFRegController: 0x166401e0>"


然后登录自己的iMessage，命令如下：

cy# [controller beginAccountSetupWithLogin:@"snakeninny@gmail.com" password:@"bbs.iosre.com" foundExisting:NO]
#"IMAccount: 0x166e7b30 [ID: 5A8E19BE-1BC9-476F-AD3B-729997FAA3BC Service: IMService[iMessage] Login: E:snakeninny@gmail.com Active: YES LoginStatus: Connected]"

函数返回了一个登录成功的IMAccount，也就是iMessage账号。接着选择用于收发iMessage的地址，返回值为1表示发送成功，命令如下：

cy# [controller setAliases:@[@"snakeninny@gmail.com"] onAccount:#0x166e7b30]
1


最后检查一下此账号是否完成了登录过程，返回值1表示成功，如下：

cy# [#0x166e7b30 CNFRegSignInComplete]
1

```
