---
title: GCD
date: 2016-06-15 15:45:12
categories: [iOS,Objective-C,线程]
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

上面提到的 定时器其实就是一个事件源,是一个内置事件源。事件源可以分为自定义事件源和内置事件源

<https://developer.apple.com/library/ios/documentation/General/Conceptual/ConcurrencyProgrammingGuide/GCDWorkQueues/GCDWorkQueues.html#//apple_ref/doc/uid/TP40008091-CH103-SW1>


**Reading Data from a Descriptor**

To read data from a file or socket, you must open the file or socket and create a dispatch source of type DISPATCH_SOURCE_TYPE_READ. The event handler you specify should be capable of reading and processing the contents of the file descriptor. In the case of a file, this amounts to reading the file data (or a subset of that data) and creating the appropriate data structures for your application. For a network socket, this involves processing newly received network data.

Whenever reading data, `you should always configure your descriptor to use non-blocking operations`. Although you can use the dispatch_source_get_data function to see how much data is available for reading, **the number returned by that function could change between the time you make the call and the time you actually read the data.** If the underlying file is truncated or a network error occurs, reading from a descriptor that blocks the current thread could stall your event handler in mid execution and prevent the dispatch queue from dispatching other tasks. For a serial queue, this could deadlock your queue, and even for a concurrent queue this reduces the number of new tasks that can be started.

Listing 4-2 shows an example that configures a dispatch source to read data from a file. In this example, the event handler reads the entire contents of the specified file into a buffer and calls a custom function (that you would define in your own code) to process the data. (The caller of this function would use the returned dispatch source to cancel it once the read operation was completed.) To ensure that the dispatch queue does not block unnecessarily when there is no data to read, this example uses the fcntl function to configure the file descriptor to perform nonblocking operations. The cancellation handler installed on the dispatch source ensures that the file descriptor is closed after the data is read.

```
dispatch_source_t ProcessContentsOfFile(const char* filename)
{
   // Prepare the file for reading.
   int fd = open(filename, O_RDONLY);
   if (fd == -1)
      return NULL;
   fcntl(fd, F_SETFL, O_NONBLOCK);  // Avoid blocking the read operation
 
   dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
   dispatch_source_t readSource = dispatch_source_create(DISPATCH_SOURCE_TYPE_READ,
                                   fd, 0, queue);
   if (!readSource)
   {
      close(fd);
      return NULL;
   }
 
   // Install the event handler
   dispatch_source_set_event_handler(readSource, ^{
      size_t estimated = dispatch_source_get_data(readSource) + 1;
      // Read the data into a text buffer.
      char* buffer = (char*)malloc(estimated);
      if (buffer)
      {
         ssize_t actual = read(fd, buffer, (estimated));
         Boolean done = MyProcessFileData(buffer, actual);  // Process the data.
 
         // Release the buffer when done.
         free(buffer);
 
         // If there is no more data, cancel the source.
         if (done)
            dispatch_source_cancel(readSource);
      }
    });
 
   // Install the cancellation handler
   dispatch_source_set_cancel_handler(readSource, ^{close(fd);});
 
   // Start reading the file.
   dispatch_resume(readSource);
   return readSource;
}
```


