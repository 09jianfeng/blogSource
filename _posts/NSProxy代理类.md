---
title: NSProxy代理类
date: 2014-12-11 22:22:27
tags: [iOS,Objective-C]
---


## NSProxy与NSObject的区别

代理类NSProxy，就是通过向外暴露Proxy类，来达到屏蔽内部使用的具体类。 NSProxy没有init方法，天生就是用来做代理的。

NSProxy与NSObject很相似，但是通过继承自NSObject的代理类是不会自动转发`respondsToSelector:`和`isKindOfClass:`这两个方法（就是调用这两个方法不会经过消息转发）, 而继承自NSProxy的代理类却是可以的。因为NSObject有定义respondsToSelector与isKindOfClass。而 NSProxy没有。 

NSProxy确实更适合实现做为消息转发的代理类, 因为作为一个抽象类, NSProxy自身能够处理的方法极小(仅<NSObject>接口中定义的部分方法), 所以其它方法都能够按照设计的预期被转发到被代理的对象中.

## 抽一个抽象协议接口，用来抽象某一类事物.

Animal.h 代表动物这一类

```

#import <Foundation/Foundation.h>

/**
 *  一个动物的所有行为的抽象
 */
@protocol Animal <NSObject>

/**
 *  动物的名字
 */
- (NSString *)name;

/**
 *  动物的类型
 */
- (NSString *)type;

/**
 *  叫
 */
- (void)call;

/**
 *  跑步
 */
- (void)run;

@end
```


## 接下来那么就是接口实现咯，简单两个实现类 Dog 和 Cat

* Dog

```
#import <Foundation/Foundation.h>
#import "Animal.h"

@interface Dog : NSObject <Animal>

@end


#import "Dog.h"

@implementation Dog

- (NSString *)name {
    return @"狗";
}

- (NSString *)type {
    return @"哺乳动物";
}

- (void)call {
    NSLog(@"狗在叫...");
}

- (void)run {
    NSLog(@"狗在跑...");
}

@end
```


* Cat

```
#import <Foundation/Foundation.h>
#import "Animal.h"

@interface Cat : NSObject <Animal>

@end
#import "Cat.h"

@implementation Cat

- (NSString *)name {
    return @"猫";
}

- (NSString *)type {
    return @"哺乳动物";
}

- (void)call {
    NSLog(@"猫在叫...");
}

- (void)run {
    NSLog(@"猫在跑...");
}

@end
```


## AnimalProxy类，作为所有Animal这一类对象的代理对象
AnimalProxy.h

```
#import <Foundation/Foundation.h>
#import "Animal.h"

/**
 *  专门用来做Animal这一类对象的代理
 *  代理也实现Animal协议
 */
@interface AnimalProxy : NSProxy <Animal>

+ (instancetype)animalWithType:(NSString *)type;

@end
```


AnimalProxy.m

```
#import "AnimalProxy.h"
#import <objc/runtime.h>

#import "Dog.h"
#import "Cat.h"

@interface AnimalProxy ()

@property (nonatomic,strong) NSMutableDictionary *selHandlerDict;

@end

@implementation AnimalProxy

+ (instancetype)animalWithType:(NSString *)type {
   //NSProxy没有init方法
    AnimalProxy *_sharedProxy = [AnimalProxy alloc];

    //创建SEL与接口实现对象的关联
    _sharedProxy.selHandlerDict = [NSMutableDictionary dictionary];

	Class animalClass = NSClassFromString(type);

    //绑定 协议 与 具体实现类对象
    [_sharedProxy _registerHandlerProtocol:@protocol(Animal) handler:[animalClass new]];
//        [_sharedProxy _registerHandlerProtocol:@protocol(Animal) handler:[Cat new]];

    return _sharedProxy;
}

- (void)_registerHandlerProtocol:(Protocol *)protocol handler:(id)handler {

    //记录protocol中的方法个数
    unsigned int numberOfMethods = 0;

    //获取protocol中定义的方法描述
    struct objc_method_description *methods = protocol_copyMethodDescriptionList(protocol,
                                                                                 YES,//是否必须实现
                                                                                 YES,//是否对象方法
                                                                                 &numberOfMethods);

    //遍历所有的方法描述，设置其Target对象
    for (unsigned int i = 0; i < numberOfMethods; i++) {

        //方法描述结构体实例
        struct objc_method_description method = methods[i];

        //方法的名字
        NSString *methodNameStr = NSStringFromSelector(method.name);

        //保存所有方法名对应的Target对象，最终接收处理消息
        [_selHandlerDict setValue:handler forKey:methodNameStr];
    }
}

- (void)forwardInvocation:(NSInvocation *)invocation {

    //获取当前触发消息的SEL
    SEL sel = invocation.selector;

    //SEL的字符串
    NSString *methodNameStr = NSStringFromSelector(sel);

    //SEL字符串查询字典得到保存的消息接收者Target
    id target = [_selHandlerDict objectForKey:methodNameStr];

    //是否找到了要转发SEL执行的Target
    if (target && [target respondsToSelector:sel])
    {
        //找到Target，就让Target处理消息
        [invocation invokeWithTarget:target];

    } else {

        //未找到Target 或 Target未实现方法，交给super去转发消息
        [super forwardInvocation:invocation];
    }
}

- (NSMethodSignature *)methodSignatureForSelector:(SEL)sel {

    //SEL的字符串
    NSString *methodNameStr = NSStringFromSelector(sel);

    //SEL字符串查询字典得到保存的消息接收者Target
    id target = [_selHandlerDict objectForKey:methodNameStr];

    //是否找到了要转发SEL执行的Target
    if (target && [target respondsToSelector:sel]) {

        //找到了Target，那么获取找到的Target的SEL对应的方法签名
        return [target methodSignatureForSelector:sel];

    } else {

        //未找到Target 或 Target未实现方法，交给super
        return [super methodSignatureForSelector:sel];
    }
}

@end
```


## 调用代理类

```
- (void)viewDidLoad {
    [super viewDidLoad];

	AnimalProxy *animalProxy = [AnimalProxy animalWithType:@"Dog"];
	
    NSLog(@"name = %@", [animalProxy name]);
    NSLog(@"type = %@", [animalProxy type]);

    [animalProxy call];
    [animalProxy run];

}
```

如果要换成 cat，则只需要调用 [AnimalProxy animalWithType:@"Cat"];


