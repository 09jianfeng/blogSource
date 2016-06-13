---
title: LLDB与debugserver
date: 2016-04-5 17:29:51
tags: [iOS,逆向]
---

LLDB是Xcode内置的动态调试工具。debugserver运行在iOS上，他作为服务端实际执行LLDB传过来的命令，显示给用户，即所谓的"远程调试"。`注意`，设备只有连电脑真机测试过APP了，debugserver才会安装进设备的`/Developer/usr/bin`目录下。但是由于缺少 `task_for_pid`权限，通过Xcode安装的debugserver只能调试自己的app。想要调试别人的App要做下面的这些步骤配置：

## 配置debugserver
### 1、帮debugserver瘦身
找出你的设备支持arm，比如我的是ipod5，那么支持的是 armv7 。将未处理的debugserver从iOS拷贝出来，放到 OSX上的 `/Users/用户名/`目录下。
```
scp -P 2222 root@localhost:/Developer/usr/bin/debugserver ~/debugserver
```

然后帮他减肥：
```
lipo -thin armv7 ~/debugserver -output ~/debugserver 
```
注意把 armv7换成你的设备对应的ARM

### 2、给debugserver添加task_for_pid权限
下载 [ent.xml](http://iosre.com/ent.xml) 到 OSX的 `~`目录，然后运行一下命令：
```
/opt/theos/bin/ldid -Sent.xml debugserver
```
正常情况下5秒内就能执行完毕。如果不行，就下载 [ent.plist](http://iosre.com/ent.plist)。然后执行
```
codesign -s - --entitlements ent.plist -f debugserver
```

### 3、将经过处理的debugserver拷回iOS
```
scp -P 2222 ~/debugserver root@localhost:/usr/bin/debugserver

ssh到设备，然后执行下面的命令：
chmod +x /usr/bin/debugserver 
```

## 用debugserver启动或附加进程
### 启动进程 （一般用于从程序启动开始调试）
```
格式：
debugserver -x backboard IP:端口 executable文件地址
例子：
debugserver -x backboard *:1234 /Applications/MobileSMS.app/MobileSMS

ps： 用这个方法启动的进程，用image list -o -f列举加载的模块，会发现只有dylib这个模块，要不停的敲入 ni（下一步），直到出现 error: invalid thread这个错误。再去 image list -o -f.
```
上面的例子启动MobileSMS，并且开启1234端口，等待任意IP地址的LLDB接入。

### 附加到进程
```
debugserver *:1234 -a "SpringBoard"
```
同样，附加到了进程`SpringBoard`，等待LLDB的接入

## LLDB的使用说明
这里使用USBmuxd链接调试
```
先按照正常的USB链接流程 22：2222端口的那个链接着
$ tcprelay.py -t 22:222
$ ssh root@localhost -p 2222

ssh到设备，debugserver开启附加一下进程。如
# debugserver *:1234 -a "SpringBoard"

开启1234端口
$ tcprelay.py -t 1234:1234
$ /Applications/Xcode.app/Contents/Developer/usr/bin/lldb                             
(lldb) process connect connect://localhost:1234

Process 39 stopped
* thread #1: tid = 0x0256, 0x33d0f518 libsystem_kernel.dylib`mach_msg_trap + 20, queue = 'com.apple.main-thread', stop reason = signal SIGSTOP
    frame #0: 0x33d0f518 libsystem_kernel.dylib`mach_msg_trap + 20
libsystem_kernel.dylib`mach_msg_trap:
->  0x33d0f518 <+20>: pop    {r4, r5, r6, r8}
    0x33d0f51c <+24>: bx     lr

libsystem_kernel.dylib`mach_msg_overwrite_trap:
    0x33d0f520 <+0>:  mov    r12, sp
    0x33d0f524 <+4>:  push   {r4, r5, r6, r8}

```

## image list
用于列举当前进程中的所有模块。因为ALSR的关系，每次进程启动时，所有的模块在虚拟内存中的起始地址都会产生随机偏移。这个起始位置是接下来会频繁用到的一个关键数据。需要用到`image List`来需找。待LLDB链接上debugserver，输入下面命令
```
(lldb) image list -o -f | grep Foundation
[  0] 0x045d7000 /System/Library/Frameworks/Foundation.framework/Foundation(0x0000000026bca000)
```
第一列是：模块序号。 第二列是：模块在虚拟内存中的起始地址因ASLR而产生的随机偏移。第三列是：全路径。括号里面的是偏移后的地址。。 以Foundation这个模块举个例子：模块的偏移值是`0x045d7000`，偏移后的地址是`0x0000000026bca000`。那么它的偏移前的地址是 `0x0000000026bca000 - 0x045d7000 = 0x00000000225F3000`;

## 符号基地址
用`image List`分析出了模块的基地址，接下来分析一下`符号基地址 `。我们以`NSLog`为例（在foundation模块）。到IDA中去foundation中搜搜NSLog，然后双击跳转到他的实现。找出NSLog相对于Foundation的相对位置。方法如下：
![lldb1](/img/lldb1.png)
```
NSLog函数的第一条指令 "SUB SP,SP,#0xC"左边的那个数#0x226038A4,代表NSLog再Foundation中的位置

减去Foundation第一行 "HEADER:225F3000"中提取出来的225F3000,就是NSLog函数在Foundation中的相对位置，即 0x226038A4 - 0x225F3000 = 0x108A4。
```

因此NSLog的实际地址为 模块的基地址（偏移后的） + 符号相对于模块的地址。 为
`0x0000000026bca000 + 0x108A4 = 0x26BDA8A4`
所以无论是符号地址，还是指令地址，都是相对于模块的地址的。只要知道了相对于模块的地址，然后知道了模块的地址，就能计算出你要的指令地址。

## breakpoint
 用于设置断点
 

 //在函数的起
 **#b function**
```
(lldb) b NSLog
Breakpoint 2: where = Foundation`NSLog, address = 0x23c5fb94
```
 
 //在地址处设置断点
**#br s -a address**
**#br s -a 'ASLROffset+address'**

```
 (lldb) br s -a 0xCCCCC
 Breakpoint 5: where = SpringBoard`___lldb_unnamed_function303$$SpringBoard, address = 0x000ccccc
(lldb) br s -a '0x6+0x9'
Breakpoint 6: address = 0x0000000f
```
breakpoint 后面那个是断点的序号。逆向工程中大多数是给汇编指令打断点。

### 实战例子
这里掩饰一个 [SpringBoard _menuButtonDown:] 函数的第一条指令设置断点为例子，演示一下操作流程。

#### 1、用IDA查看偏移前基地址。
 查找 [SpringBoard _menuButtonDown:],的第一条指令的偏移前基地址为0x00017720。
![lldb2](/img/lldb2.png)

lldb到debugserver， image list -o -f。列出模块偏移地址如下：
![lldb3](/img/lldb3.png)
看到springBoard的偏移是 0x0009c000。 所以 _menuButtonDown:第一条指令的地址就是
```
0x0009c000 + 0x00017720 = 0xB3720
```
在llDB中输入命令给这个地址下断点：
```
(lldb) br s -a 0xB3720
Breakpoint 1: where = SpringBoard`___lldb_unnamed_function299$$SpringBoard, address = 0x000b3720
```
按下Home键触发断点，如图
![lldb4](/img/lldb4.png)
`注：`r1 寄存器放的是函数第二个参数的名称。第二个参数的名称是函数名。当程序停下来后，可以用
```
c
```
命令让程序继续运行.

#### 2、断点命令 
* 禁用断点：
```
//禁用所有断点
(lldb) br dis 
//禁用某个断点
(lldb) br dis 6
```

* 启动断点：
```
//启动所有断点
(lldb) br en
//启动某个断点
(lldb) br en 6
```

* 删除断点：
```
//删除所有断点
(lldb) br del
//删除某个断点
(lldb) br del 8
```

某个断点触发的时候，执行预先设置的指令，它的用法如下（假设1号断点位于某个objc_msgSend函数上）：
```
(lldb) br com add 1
```
执行这条命令后会要求我们设置一系列指令，以"Done"结束，如下：
```
Enter your debugger command(s).  Type 'DONE' to end.
> po [$r0 class]
> p (char *)$r1
> c
> DONE
```

#### 3、 print
打印某处的值。 仍然以[SpringBoard _menuButtonDown:]里的指令为例。
```
(lldb) po $r0
(lldb) p (char *)$r1
```

#### 4、命令 `ni`/ `si` 执行下一条指令。是 `nexti`/`stepi`的简写。 
如果下一步是个函数调用：
stepi 在执行下一步的时候会进入函数体。
nexti 在执行下一步的时候不会进入函数体。

#### 5、当进程停留在某一条"BL"指令上时，LLDB会自动解析这条指令，把指令对应的符号注释出来。但是，LLDB的解析有时会出错，注释出的符号不对。这种情况下，请以IDA静态分析出的符号为准。

#### 6、命令 `X` 打印一个地址处存放的值，如下：
```
(lldb) p/x $sp
(unsigned int) $4 = 0x006e838c
(lldb) x/10 $sp
0x006e838c: 0x00000000 0x22f2c975 0x00000000 0x00000000
0x006e839c: 0x26c6bf8c 0x0000000c 0x17a753c0 0x17a753c8
0x006e83ac: 0x000001c8 0x17a75200
(lldb) x/10 0x006e838c
0x006e838c: 0x00000000 0x22f2c975 0x00000000 0x00000000
0x006e839c: 0x26c6bf8c 0x0000000c 0x17a753c0 0x17a753c8
0x006e83ac: 0x000001c8 0x17a75200
```

#### 7、给指定的寄存器赋值，从而 "对程序进行改动，观察程序的执行过程有什么变化"
```
//写1 给寄存器r0
(lldb) register write r0 1
```

## 练习

### 记一次 register read $r1 与 p (char *)r1 的区别

```
1 __text:00009EA8                 PUSH            {R4,R7,LR}
2 __text:00009EAA                 ADD             R7, SP, #4
3 __text:00009EAC                 SUB             SP, SP, #0x1C
4 __text:00009EAE                 MOV             R1, #(selRef_navigationController - 0x9EBA) ; selRef_navigationController
5 __text:00009EB6                 ADD             R1, PC ; selRef_navigationController
6 __text:00009EB8                 MOV             R4, R0
7 __text:00009EBA                 LDR             R1, [R1] ; "navigationController"
8 __text:00009EBC                 BLX             _objc_msgSend
```

这段代码是从IDA上截图下来的，是UITableViewController的子类TableViewController的showSourcePressed:函数的前面8行代码。这段代码是Obective-C调用[tableViewController navigationController]转换后的汇编代码。

断点打在第一行（打断点用lldb配合debugServer），分析、查看内存以及寄存器的值：

* 触发断点后，控制台输出

```
-[TableViewController showSourcePressed:]:
->  0xd8ea8 <+0>: push   {r4, r7, lr}
    0xd8eaa <+2>: add    r7, sp, #0x4
    0xd8eac <+4>: sub    sp, #0x1c
    0xd8eae <+6>: movw   r1, #0xd1c6
```
r0 上放的是类指针， r1放的是函数名字符串指针。打印验证一下

```
(lldb) po $r0
<TableViewController: 0x16691d00>

(lldb) po (char *)$r1
"showSourcePressed:"
```

* ni 继续跟踪

```
->  0xd8eae <+6>:  movw   r1, #0xd1c6
    0xd8eb2 <+10>: movt   r1, #0x3
    0xd8eb6 <+14>: add    r1, pc
    0xd8eb8 <+16>: mov    r4, r0
```

[movt 指令](http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.dui0204ic/Cjagdjbf.html)是移动后面的立即数 #0x3 到 r1的高半字，不影响低半字。经过  movw   r1, #0xd1c6；movt   r1, #0x3；add    r1, pc；这三条指令的处理后打印一下。r1寄存器的值

```
->  0xd8eb8 <+16>: mov    r4, r0
    0xd8eba <+18>: ldr    r1, [r1]
    0xd8ebc <+20>: blx    0x10ee0c                  ; symbol stub for: objc_msgSend
    0xd8ec0 <+24>: movw   r1, #0xdef4
    
(lldb) me r -s1 -fc -c21 0x00116080
0x00116080: z\x12?'~\x06?'?\x9c\x02-?5\x10\0jL?'M
(lldb) me r -s4 -fx -c4 0x00116080
0x00116080: 0x27c9127a 0x27ca067e 0x2d029cd0 0x001035ed
(lldb) re r $r1
      r1 = 0x00116080  "navigationController"
(lldb) p (char *)$r1
(char *) $58 = 0x00116080 "z\x12\xffffffc9'~\x06\xffffffca'\xffffffd0\xffffff9c\x02-\xffffffed5\x10"

```
地址 0x00116080存放的是0x27c9127a。 其实0x27c9127a是一个指针，这里纠结了我很久，因为用p 打印不出来。但是用 re r读寄存器却能读出是字符串，令人好生怀疑。直到执行完  `ldr    r1, [r1]`后打印 r1的值，才恍然大悟。原来 register read 是可以识别 (char **) 然后输出指向的字符串。。

```
->  0xd8ebc <+20>: blx    0x10ee0c                  ; symbol stub for: objc_msgSend
    0xd8ec0 <+24>: movw   r1, #0xdef4
    0xd8ec4 <+28>: movt   r1, #0x3
    0xd8ec8 <+32>: add    r1, pc
(lldb) p (char *)$r1
(char *) $60 = 0x27c9127a "navigationController"
(lldb) re r $r1
      r1 = 0x27c9127a  "navigationController"
(lldb) me  r -s1 -fc -c21 0x27c9127a
0x27c9127a: navigationController\0
(lldb) 
```



# [lldb常用命令](http://www.cnblogs.com/wfwenchao/p/3991060.html) 

## lldb p命令
p打印变量的值， 打印出来默认是十进制的值。  p/t输出二进制，p/x输出16进制。

## lldb po命令
是打印对象的，就是调用对象的 description的函数

## lldb re命令
格式 `register read –format 寄存器`

`register read 寄存器` 简写 `re r/x 寄存器` 读取寄存器的值, r/x 16进制来表示。 r/t是二进制 。
`re r -a` 读取全部寄存器的内容

## 内存读取 memory
内存读取，从0xbffff3c0开始读，并且4字节 16进制 连续8个

```
(lldb) memory read -size 4 -format x -count 8 0x00116080
(lldb) me r -s4 -fx -c8 0x00116080
(lldb) x -s4 -fx -c8 0x00116080 
```