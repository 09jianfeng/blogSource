---
title: RunLoop
date: 2016-07-13 10:57:56
tags:	[iOS,Objective-C]
---

## 什么是RunLoop
顾名思义，就是循环的意思。不断的循环

## event

命令式执行

```
int main(int argc, char * argv[]){
	NSLog(@"hello world");
	return 0;
}
```

Event驱动

```
int main(int argc, char * argv[]){
	while(AppIsRunning){
		id whoWakesMe = SleepForWakingUp();
		id event = GetEvent(whoWakesMe);
		HandleEvent(event);
	}
	return 0;
}
```

runloop就是 event驱动的。

## run Loop作用
* 使程序一直运行并接受用户输入
* 决定程序在何处应该处理哪些Event
* 调用解耦 (Message Queue)
* 节省CPU时间

## Run Loops在Cocoa
层次架构

Foundation: NSRunLoop -> 就是对CF层的一个封装

Core Foundation: CFRunLoop -> 开源的，大概是3000行左右

System：GCD、mach kernel、block、pthread....


跟RunLoop关系比较大的一些

* NSTimer
* UIEvent
* AutoreleaseNSObject(NSDelayedPerforming)
* NSObject(NSThreadPerformAddition)
* CADisplayLink、CATransition（设置controller之间的转场动画）
* CAAnimation
* dispatch_get_main_queue()
* NSURLConnection
* AFNetworking

## Runloop Callouts

主线程几乎所有函数都是从以下六个之一的函数调起来

```
__CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__
__CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__
__CFRUNLOOP_IS_CALLING_OUT_TO_A_BLOCK__
__CFRUNLOOP_IS_CALLING_OUT_TO_A_TIMER_CALLBACK_FUNCTION__
__CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE1_PERFORM_FUNCTION__
__CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__
```

## RunLoop 机制 

直接讲 Core Foundation层。

![runloop1](../img/runloop1.png)

CFRunLoopMode是个非常重要的概念， 起决定性的作用就是下面的三个。CFRunloopSource、CFRunLoopTimer、CFRunLoopObserver。   runloop可以嵌套，要么就同时只有一个mode在跑。

### CFRunLoopTimer

![runloop2](../img/runloop2.png)

### CFRunLoopSource
* Source是RunLoop的数据源抽象类
* RunLoop定义了两个Version的Source:

```
1、Source0: 处理App内部时间、App自己负责管理（触发），如UIEvent、CFSocket
2、Source1: 由Runloop和内核管理,Mach port驱动，如CFMackPort、CFMessagePort
```

![runloop3](../img/runloop3.png)


### CFRunLoopObserver

![runloop4](../img/runloop4.png)

CAAnimation应该是在beforWaiting或者afterWaiting后才调用，而不是调用了后就立即调用。会先收集一些animation的动作后，再一起执行。


## RunLoopObserver 与 Autorelease Pool
Observer被触发的时候，调用的 Autorelease。  UIKit通过RunLoopObserver在RunLoop两次Sleep间对AutoreleasePool进行Pod和Push。将这次Loop中产生的Autorelease对象释放。 

子线程中的自动释放池的释放：每个线程创建的时候就会创建一个autorelease pool，并且在线程退出的时候，清空autorelease pool。所以子线程的autorelease对象，要么在子线程中设置runloop清楚

[Autorelease原理](http://blog.sunnyxx.com/2014/10/15/behind-autorelease/) 关键点：一个以 AutoreleasePoolPage为节点的双向链表, 每个page是4096（虚拟内存一页的大小）。除了一些必要的字段占用的空间。剩下的空间用于存放需要释放的对象指针。放满了后再直线下一个链表

## CFRunLoopMode
RunLoop在同一段时间只能且必须在一种特定Mode下Run,更换Mode时,需要停止当前Loop,然后重启新Loop,Mode是iOS App滑动顺畅的关键 。（滑动的时候，会切掉mode，只跑滑动的mode）

* NSDefaultRunLoopMode  默认状态、空闲状态
* UITrackingRunLoopMode 滑动ScrollView时的mode
* UIInitializationRunLoopMode 私有，App启动时
* NSRunLoopCommonModes Mode集合


## UITrackingRunLoopMode 与 Timer

![runloop5](../img/runloop5.png)

注意：子线程默认是没有创建runloop的，除非调用了Runloop相关的接口，才会生成。  

## RunLoopMode的切换

![runloop6](../img/runloop6.png)

可以看出在滑动的时候是运行在UITrackingRunLoopMode下的，这个时候是在跑 timer。


## RunLoop 与 dispatch\_get\_main\_queue()

![runloop7](../img/runloop7.png)

queue如果是main queue的话，GCD就和runloop有关系。主要是因为大家获取的主线程是一样的。这里的关系，也就仅仅是用于runloop调起main queue的block。

## RunLoop的挂起与唤醒

![runloop8](../img/runloop8.png)

mach\_msg\_trap、mach\_msg 是mac 内核的东西.__CFRunLoopServiceMachPort 发送一个消息,代表等待别人唤起的状态，这个runloop就被挂起。

* 指定用于唤醒的mach_port端口
* 调用mach\_msg监听唤醒端口，被唤醒前，系统内核将这个线程挂起，停留在mach\_msg\_trap状态
* 由另一个线程 （或另一个进程中的某个线程）向内核发送这个端口的msg后，trap状态被唤醒，RunLoop继续开始干活。

## RunLoop迭代执行顺序

![runloop9](../img/runloop9.png)

确切说是主线程上的RunLoop的执行顺序。


## 实践
`[runloop addPort:[NSMachPort port] forMode:NSDefaultRunLoopMode]` 用于别让线程被销毁，一直等待这这个port。

![runloop10](../img/runloop10.png)

加载图片的时候避免在滑动的模式下加载，以免卡住滑动。闲时再加载

![runloop11](../img/runloop11.png)

Async Test Case

![runloop12](../img/runloop12.png)


## 参考
[iOS线下分享 孙源]()

深入理解RunLoop <http://blog.ibireme.com/2015/05/18/runloop/>











