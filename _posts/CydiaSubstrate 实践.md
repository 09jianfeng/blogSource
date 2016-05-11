title: CydiaSubstrate 实践
date: 2016-04-11 09:29:51
tags: [iOS,逆向]
---
CydiaSubstrate是绝大部分tweak正常工作的基础，它由MobileHooker、MobileLoader和Safe mode组成。

# MobileHooker
MobileHooker的作用是替换系统函数，也就是所谓的hook，它主要包含以下两个函数：
```
void MSHookMessageEx(Class class, SEL selector, IMP replacement, IMP *result);
void MSHookFunction(void* function, void* replacement, void** p_original);
```
其中MSHookMessageEx作用于Objective-C函数，通过调用`method_setImplementation`函数将[class selector]的实现改为replacement，达到hook的目的。

第3章提到的`Logos`语法主要是对此函数作了一层封装，让编写针对Objective-C函数的hook代码变得更简单直观了，但其底层实现仍完全基于MSHookMessageEx。对于Objective-C函数的hook，推荐使用更一目了然的Logos语法。

MSHookFunction作用于`C和C++`函数，通过编写汇编指令，在进程执行到function时转而执行replacement，同时保存function的指令及其返回地址，使得用户可以选择性地执行function，并保证进程能够在执行完replacement后继续正常运行.

`注意：`
MSHookMessageEX对函数有长度要求，function里的指令加起来的长度不能太短。如果想勾住短函数，一般是勾住短函数里面调用的其他函数来间接勾住短函数。

**例子：**
### 创建iOS工程 iOSRETargetApp，用来被hook
- 1、用Theos创建一个工程iOSRETargetApp.命令如下

```
snakeninnys-MacBook:Code snakeninny$ /opt/theos/bin/nic.pl
NIC 2.0 - New Instance Creator
-------------------------------
  [1.] iphone/application
  [2.] iphone/library
  [3.] iphone/preference_bundle
  [4.] iphone/tool
  [5.] iphone/tweak
Choose a Template (required): 1
Project Name (required): iOSRETargetApp
Package Name [com.yourcompany.iosretargetapp]: com.iosre.iosretargetapp
Author/Maintainer Name [snakeninny]: snakeninny
Instantiating iphone/application in iosretargetapp/...
Done.
```
- 2、修改RootViewController.mm 
```
#import "RootViewController.h"
class CPPClass
{
      public:
            void CPPFunction(const char *);
};
void CPPClass::CPPFunction(const char *arg0)
{
      for (int i = 0; i < 66; i++) // This for loop makes this function long enough to validate MSHookFunction
      {
            u_int32_t randomNumber;
            if (i % 3 == 0) randomNumber = arc4random_uniform(i);
            NSProcessInfo *processInfo = [NSProcessInfo processInfo];
            NSString *hostName = processInfo.hostName;
            int pid = processInfo.processIdentifier;
            NSString *globallyUniqueString = processInfo.globallyUniqueString;
            NSString *processName = processInfo.processName;
            NSArray *junks = @[hostName, globallyUniqueString, processName];
            NSString *junk = @"";
            for (int j = 0; j < pid; j++)
            {
                  if (pid % 6 == 0) junk = junks[j % 3];
            }
            if (i % 68 == 1) NSLog(@"Junk: %@", junk);
      }
      NSLog(@"iOSRE: CPPFunction: %s", arg0);
}
extern "C" void CFunction(const char *arg0)
{
      for (int i = 0; i < 66; i++) // This for loop makes this function long enough to validate MSHookFunction
      {
            u_int32_t randomNumber;
            if (i % 3 == 0) randomNumber = arc4random_uniform(i);
            NSProcessInfo *processInfo = [NSProcessInfo processInfo];
            NSString *hostName = processInfo.hostName;
            int pid = processInfo.processIdentifier;
            NSString *globallyUniqueString = processInfo.globallyUniqueString;
            NSString *processName = processInfo.processName;
            NSArray *junks = @[hostName, globallyUniqueString, processName];
            NSString *junk = @"";
            for (int j = 0; j < pid; j++)
            {
                  if (pid % 6 == 0) junk = junks[j % 3];
            }
            if (i % 68 == 1) NSLog(@"Junk: %@", junk);
      }
      NSLog(@"iOSRE: CFunction: %s", arg0);
}
extern "C" void ShortCFunction(const char *arg0) // ShortCFunction is too short to be hooked
{
      CPPClass cppClass;
      cppClass.CPPFunction(arg0);
}
@implementation RootViewController
- (void)loadView {
      self.view = [[[UIView alloc] initWithFrame:[[UIScreen mainScreen] applicationFrame]] autorelease];
      self.view.backgroundColor = [UIColor redColor];
}
- (void)viewDidLoad
{
      [super viewDidLoad];
      CPPClass cppClass;
      cppClass.CPPFunction("This is a C++ function!");
      CFunction("This is a C function!");
      ShortCFunction("This is a short C function!");
}
@end
```
- 3、修改makeFile，并安装
```
THEOS_DEVICE_IP = iOSIP
ARCHS = armv7 arm64
TARGET = iphone:latest:8.0
include theos/makefiles/common.mk
APPLICATION_NAME = iOSRETargetApp
iOSRETargetApp_FILES = main.m iOSRETargetAppApplication.mm RootViewController.mm
iOSRETargetApp_FRAMEWORKS = UIKit CoreGraphics
include $(THEOS_MAKE_PATH)/application.mk
after-install::
      install.exec "su mobile -c uicache"
```
make package install 后，应用安装到了设备上。然后在设备上执行 "su mobile -c uicache"，这句命令来刷新UI缓存，显示出图标来。点击运行app，变成红色背景后。在设备上运行命令查看log。看看是否跟预想的输出符合。
```
FunMaker-5:~ root# grep iOSRE: /var/log/syslog
Nov 18 11:13:34 FunMaker-5 iOSRETargetApp[5072]: iOSRE: CPPFunction: This is a C++ function!
Nov 18 11:13:34 FunMaker-5 iOSRETargetApp[5072]: iOSRE: CFunction: This is a C function!
Nov 18 11:13:35 FunMaker-5 iOSRETargetApp[5072]: iOSRE: CPPFunction: This is a short C function!
```
`提示`: /var/log/syslog 需要安装插件syslogd。

### 编写hook工程，用来勾住上面工程的函数
- 1、用theos来创建工程。命令如下：

```
snakeninnys-MacBook:Code snakeninny$ /opt/theos/bin/nic.pl
NIC 2.0 - New Instance Creator
------------------------------
  [1.] iphone/application
  [2.] iphone/library
  [3.] iphone/preference_bundle
  [4.] iphone/tool
  [5.] iphone/tweak
Choose a Template (required): 5
Project Name (required): iOSREHookerTweak
Package Name [com.yourcompany.iosrehookertweak]: com.iosre.iosrehookertweak
Author/Maintainer Name [snakeninny]: snakeninny
[iphone/tweak] MobileSubstrate Bundle filter [com.apple.springboard]: com.iosre.iosretargetapp
[iphone/tweak] List of applications to terminate upon installation (space-separated, '-' for none) [SpringBoard]: iOSRETargetApp
Instantiating iphone/tweak in iosrehookertweak/...
Done.
```

- 2、修改Tweak.xm
```
#import <substrate.h>
void (*old__ZN8CPPClass11CPPFunctionEPKc)(void *, const char *);
void new__ZN8CPPClass11CPPFunctionEPKc(void *hiddenThis, const char *arg0)
{
      if (strcmp(arg0, "This is a short C function!") == 0) old__ZN8CPPClass11CPPFunctionEPKc(hiddenThis, "This is a hijacked short C function from new__ZN8CPPClass11CPPFunctionEPKc!");
      else old__ZN8CPPClass11CPPFunctionEPKc(hiddenThis, "This is a hijacked C++ function!");
}
void (*old_CFunction)(const char *);
void new_CFunction(const char *arg0)
{
      old_CFunction("This is a hijacked C function!"); // Call the original CFunction
}
void (*old_ShortCFunction)(const char *);
void new_ShortCFunction(const char *arg0)
{
      old_CFunction("This is a hijacked short C function from new_ShortCFunction!"); // Call the original ShortCFunction
}
%ctor
{
      @autoreleasepool
      {
            MSImageRef image = MSGetImageByName("/Applications/iOSRETargetApp.app/iOSRETargetApp");
            void *__ZN8CPPClass11CPPFunctionEPKc = MSFindSymbol(image, "__ZN8CPPClass11CPPFunctionEPKc");
            if (__ZN8CPPClass11CPPFunctionEPKc) NSLog(@"iOSRE: Found CPPFunction!");
            MSHookFunction((void *)__ZN8CPPClass11CPPFunctionEPKc, (void *)&new__ZN8CPPClass11CPPFunctionEPKc, (void **)&old__ZN8CPPClass11CPPFunctionEPKc);
            void *_CFunction = MSFindSymbol(image, "_CFunction");
            if (_CFunction) NSLog(@"iOSRE: Found CFunction!");
            MSHookFunction((void *)_CFunction, (void *)&new_CFunction, (void **)&old_CFunction);
            void *_ShortCFunction = MSFindSymbol(image, "_ShortCFunction");
            if (_ShortCFunction) NSLog(@"iOSRE: Found ShortCFunction!");
            MSHookFunction((void *)_ShortCFunction, (void *)&new_ShortCFunction, (void **)&old_ShortCFunction); // This MSHookFuntion will fail because ShortCFunction is too short to be hooked
      }
}
```

`MSFindSymbol`的作用是用来找需要勾住的symbol。symbol其实就是函数名称，进程会根据这个symbol来寻找函数地址，然后跳过去执行。symbol分两类，public与private两种。private的symbol不是你想调用就能调用的。MSHookFunction只能调用public的symbol，如果想要调用private的symbol，就要借助MSFindSymbol。

举个例子，NSLog函数的实现位于Foundation库，所以对于NSLog这个symbol来说，MSGetImageByName的参数就应该是`/System/Library/Frameworks/Foundation.framework/Foundation`

**记住下面的写法：**
```
MSImageRef image = MSGetImageByName("/System/Library/Frameworks/Foundation.framework/Foundation");
void *symbol = MSFindSymbol(image, "NSLog");
```
在这个例子中，我们在iOSRETargetApp的RootViewController.mm中定义的3个函数名分别是`CPPClass::CPPFunction`、`CFunction`和`ShortCFunction`，怎么到了iOSREHookerTweak的tweak.xm里，它们却变成了`__ZN8CPPClass11CPPFunctionEPKc`、`_CFunction`和`_ShortCFunction`？简单地说，这是因为编译器对函数名做了进一步的处理。

这些symbol我们需要通过IDA来找。

- 3、用IDA来寻找symbol。
把iOSRETargetApp的二进制文件拉入IDA分析。
如下图
![CydiaSub1](/img/cydiasub1.jpeg)

可以看到，CPPClass::CPPFunction(char const*)、_CFunction和_ShortCFunction位列其中。双击“CPPClass::CPPFunction(char const*)",跳转到这个函数的实现上。如下图所示：

![CydiaSub2](/img/cydiasub2.jpeg)

可以看到export后面的那串字符串，__ZN8CPPClass11CPPFunctionEPKc。那就是symbol。其他两个方法的symbol查找以此类推。

- 4、MSHookFunction的写法
```
#import <substrate.h>
returnType (*old_symbol)(args);
returnType new_symbol(args)
{
      // Whatever
}
void InitializeMSHookFunction(void) // This function is often called in %ctor i.e. constructor
{
      MSImageRef image = MSGetImageByName("/path/to/binary/who/contains/the/implementation/of/symbol");
      void *symbol = MSFindSymbol(image, "symbol");
      if (symbol）MSHookFunction((void *)symbol, (void *)&new_ symbol, (void **)&old_ symbol);
      else NSLog(@"Symbol not found!");
}
```


### MobileLoader
MobileLoader的作用是加载第三方dylib。在iOS启动时，会由launchd将MobileLoader载入内存，然后MobileLoader会根据dylib的同名plist文件指定的作用范围，有选择地在不同进程里通过dlopen函数打开目录/Library/MobileSubstrate/DynamicLibraries/下的所有dylib。这个plist文件的格式已在Theos部分详细讲解，此处不再赘述。对于大多数初级iOS逆向工程师来说，MobileLoader的工作过程是完全透明的，此处仅作简单了解即可。

### 安全模式
应用的质量良莠不齐，程序崩溃在所难免。因为tweak的本质是dylib，寄生在别的进程里，一旦出错，可能会导致整个进程崩溃，而一旦崩溃的是SpringBoard等系统进程，则会造成iOS瘫痪，所以CydiaSubstrate引入了Safe mode，它会捕获SIGTRAP、SIGABRT、SIGILL、SIGBUS、SIGSEGV、SIGSYS这6种信号，然后进入安全模式。

在安全模式里，所有基于CydiaSubstrate的第三方dylib均会被禁用，便于查错与修复。但是，并不是拥有了Safe mode就能高枕无忧，在很多时候，设备还是会因为第三方dylib的原因而无法进入系统，症状主要有：开机时卡在白苹果上，或者进度圈不停地转。在出现这种情况时，可以同时按住home和lock键硬重启，然后按住音量“+”键来完全禁用CydiaSubstrate，待系统重启完毕后，再来查错与修复。当问题被成功修复后，再重启一次iOS，就能重新启用CydiaSubstrate了，非常方便。
 