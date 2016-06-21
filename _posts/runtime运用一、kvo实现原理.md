---
title: runtime运用一、kvo实现原理
date: 2016-05-25 11:09:21
categories: [iOS,Objective-C,Runtime]
tags: [iOS,Objective-C,Runtime]
---

## 参考链接
[1、mikeash.com kvo实现原理](https://www.mikeash.com/pyblog/friday-qa-2009-01-23.html)

## KVO的基本用法
[苹果官方文档](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/KeyValueCoding/Articles/KeyValueCoding.html)
[参考objc.io的这篇文章](https://www.objc.io/issues/7-foundation/key-value-coding-and-observing/)
[作业部落这里有比较详细的用法](https://www.zybuluo.com/MicroCai/note/66738)

## 实现原理
当你为类A的对象设置属性观察的时候，编译器会利用Objc的runtime特性，生成一个类B继承自类A。并且重写所有类A被监测过的属性的set方法，插入检测代码。使得属性被修改的时候，可以检测到变量改变前与改变后的值。 同时会把那个A类对象的isa指针指向类B，编译器还会重写class类，以保证调用这个函数还是原来的那个类。 说的这些原理，跑一下下面的例子就都一目了然了。


看下这个例子：

```
#import <Foundation/Foundation.h>
#import <objc/runtime.h>


@interface TestClass : NSObject
{
    int x;
    int y;
    int z;
}
@property int x;
@property int y;
@property int z;
@end
@implementation TestClass
@synthesize x, y, z;
@end

static NSArray *ClassMethodNames(Class c)
{
    NSMutableArray *array = [NSMutableArray array];
    
    unsigned int methodCount = 0;
    Method *methodList = class_copyMethodList(c, &methodCount);
    unsigned int i;
    for(i = 0; i < methodCount; i++)
        [array addObject: NSStringFromSelector(method_getName(methodList[i]))];
    free(methodList);
    
    return array;
}

static void PrintDescription(NSString *name, id obj)
{
    NSString *str = [NSString stringWithFormat:
                     @"%@: %@\n\tNSObject class %s\n\tlibobjc class %s\n\timplements methods <%@>",
                     name,
                     obj,
                     class_getName([obj class]),
                     class_getName(object_getClass(obj)),
                     [ClassMethodNames(object_getClass(obj)) componentsJoinedByString:@", "]];
    printf("%s\n", [str UTF8String]);
}

int main(int argc, char **argv)
{
    [NSAutoreleasePool new];
    
    TestClass *x = [[TestClass alloc] init];
    TestClass *y = [[TestClass alloc] init];
    TestClass *xy = [[TestClass alloc] init];
    TestClass *control = [[TestClass alloc] init];
    
    [x addObserver:x forKeyPath:@"x" options:0 context:NULL];
    [xy addObserver:xy forKeyPath:@"x" options:0 context:NULL];
    [y addObserver:y forKeyPath:@"y" options:0 context:NULL];
    [xy addObserver:xy forKeyPath:@"y" options:0 context:NULL];
    
    PrintDescription(@"control", control);
    PrintDescription(@"x", x);
    PrintDescription(@"y", y);
    PrintDescription(@"xy", xy);
    
    printf("Using NSObject methods, normal setX: is %p, overridden setX: is %p\n",
           [control methodForSelector:@selector(setX:)],
           [x methodForSelector:@selector(setX:)]);
    printf("Using libobjc functions, normal setX: is %p, overridden setX: is %p\n",
           method_getImplementation(class_getInstanceMethod(object_getClass(control),
                                                            @selector(setX:))),
           method_getImplementation(class_getInstanceMethod(object_getClass(x),
                                                            @selector(setX:))));
    
    return 0;
}
```

输出：

```
control: <TestClass: 0x100106a30>
	NSObject class TestClass
	libobjc class TestClass
	implements methods <setZ:, setY:, z, setX:, x, y>
x: <TestClass: 0x1001068f0>
	NSObject class TestClass
	libobjc class NSKVONotifying_TestClass
	implements methods <setY:, setX:, class, dealloc, _isKVOA>
y: <TestClass: 0x1001069f0>
	NSObject class TestClass
	libobjc class NSKVONotifying_TestClass
	implements methods <setY:, setX:, class, dealloc, _isKVOA>
xy: <TestClass: 0x100106a10>
	NSObject class TestClass
	libobjc class NSKVONotifying_TestClass
	implements methods <setY:, setX:, class, dealloc, _isKVOA>
Using NSObject methods, normal setX: is 0x100001180, overridden setX: is 0x7fff8f77d0ad
Using libobjc functions, normal setX: is 0x100001180, overridden setX: is 0x7fff8f77d0ad
Program ended with exit code: 0
```

## 自己实现一个kvo

[参考这篇文章](http://tech.glowing.com/cn/implement-kvo/)