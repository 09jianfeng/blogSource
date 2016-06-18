---
title: GCD
date: 2016-06-15 15:45:12
tags: [iOS,线程]
---

## dispatch_group

如果想在dispatch_queue中所有的任务执行完成后在做某种操作，在串行队列中，可以把该操作放到最后一个任务执行完成后继续，但是在并行队列中怎么做呢。这就有dispatch_group 成组操作。比如

```
dispatch_queue_t dispatchQueue = dispatch_queue_create("ted.queue.next", DISPATCH_QUEUE_CONCURRENT);
dispatch_group_t dispatchGroup = dispatch_group_create(); 
dispatch_group_async(dispatchGroup, dispatchQueue, ^(){ 
	NSLog(@"dispatch-1"); 
}); 
dispatch_group_async(dispatchGroup, dispatchQueue, ^(){
 	NSLog(@"dspatch-2"); 
 }); 
dispatch_group_notify(dispatchGroup, dispatch_get_main_queue(), ^(){ 
	NSLog(@"end"); 
});
```


## 定时器

```
	dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    dispatch_source_t timer = dispatch_source_create(DISPATCH_SOURCE_TYPE_TIMER, 0, 0, queue);
    dispatch_source_set_timer(timer, DISPATCH_TIME_NOW, 1.0 * NSEC_PER_SEC, 0.0 * NSEC_PER_SEC);
    dispatch_source_set_event_handler(timer, ^{
        NSLog(@"now is %@",[NSDate date]);
        dispatch_source_cancel(timer);
    });
    dispatch_resume(timer);
```

## 自定义队列

```
串行队列
dispatch_queue_t queue = dispatch_queue_create("hehehqueue", DISPATCH_QUEUE_SERIAL);
    dispatch_async(queue, ^{
        ....
    })
};

并行队列：
dispatch_queue_t queue = dispatch_queue_create("hehehqueue", DISPATCH_QUEUE_CONCURRENT);
    dispatch_async(queue, ^{
        ....
    })
};

```

## dispatch\_barrier\_async

是在前面的任务执行结束后它才执行，而且它后面的任务等它执行完成之后才会执行。
在并行队列中，有的时候我们需要让某个任务单独执行，也就是他执行的时候不允许其他任务执行。这时候dispatch_barrier就派上了用场。

dispatch_barrier最典型的使用场景是读写问题，NSMutableDictionary在多个线程中如果同时写入，或者一个线程写入一个线程读取，会发生无法预料的错误。但是他可以在多个线程中同时读取。如果多个线程同时使用同一个NSMutableDictionary。怎样才能保护NSMutableDictionary不发生意外呢？

```
- (void)setObject:(id)anObject forKey:(id
)aKey
{
    dispatch_barrier_async(self.concurrentQueue, ^{
        [self.mutableDictionary setObject:anObject forKey:aKey];
    });
}


- (id)objectForKey:(id)aKey
{
    __block id object = nil;    
    dispatch_sync(self.concurrentQueue, ^{
        object = [self.mutableDictionary objectForKey:aKey];
    });    return  object;
}
```

## 执行某个代码片段N次

```
dispatch_apply(5, globalQ, ^(size_t index) {
    // 执行5次
});
```

## set\_specific & get\_specific  给队列关联内容

iOS 6之后dispatch\_get\_current\_queue()被废弃，如果我们需要区分不同的queue，可以使用set_specific方法。根据对应的key是否有值来区分。或者有时候我们需要将某些东西关联到队列上，比如我们想在某个队列上存一个东西，或者我们想区分2个队列。GCD提供了dispatch\_queue\_set\_specific方法，通过key，将context关联到queue上

```
void dispatch_queue_set_specific(dispatch_queue_t queue, const void *key, void *context, dispatch_function_t destructor);

- queue：需要关联的queue，不允许传入NULL
- key：唯一的关键字
- context：要关联的内容，可以为NULL
- destructor：释放context的函数，当新的context被设置时，destructor会被调用



有存就有取，将context关联到queue上之后，可以通过dispatch_queue_get_specific或者dispatch_get_specific方法将值取出来。

void *dispatch_queue_get_specific(dispatch_queue_t queue, const void *key);

void *dispatch_get_specific(const void *key);

dispatch_queue_get_specific: 根据queue和key取出context，queue参数不能传入全局队列

dispatch_get_specific: 根据唯一的key取出当前queue的context。
如果当前queue没有key对应的context，则去queue的target queue取，取不着返回NULL，如果对全局队列取，也会返回NULL
```


## dispatch\_block\_t 定义不带参数的回调函数


之前不带参数的回调函数这样写的：

在.h文件里

定义类型

```
typedef void(^successBlockAction)();
```

在.m 文件中

```
-(void)onClick:(successBlockAction)successBlock{

successBlock();

}
```

现在使用这个更加方便的方式：

在.h文件里定义一个属性

```
@property(nonatomic,copy) dispatch_block_t succeedBlock;
```
在.m文件里

```
UIButton *btn = [[UIButton alloc]init];

btn.successBlockAction= ^() {

	NSLog(@" button clicked");

};
```



## dispatch source

上面提到的 定时器其实就是一个事件源,是一个内置事件源

<https://developer.apple.com/library/ios/documentation/General/Conceptual/ConcurrencyProgrammingGuide/GCDWorkQueues/GCDWorkQueues.html#//apple_ref/doc/uid/TP40008091-CH103-SW1>






