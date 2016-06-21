---
title: MVVM ReactiveCocoa.md
date: 2016-06-03 09:53:28
categories: [iOS,Objective-C,设计模式]
tags: [iOS,Objective-C,设计模式]
---

参考：

[ReactiveCocoaTutorial-part1](https://www.raywenderlich.com/62699/reactivecocoa-tutorial-pt1)

[ReactiveCocoaTutorial-part2](http://www.raywenderlich.com/62796/reactivecocoa-tutorial-pt2)

[南峰子的技术博客 MVVM一](http://southpeak.github.io/blog/2014/08/08/mvvmzhi-nan-yi-:flickrsou-suo-shi-li/)

[南峰子技术博客 MVVM 二](http://southpeak.github.io/blog/2014/08/12/mvvmzhi-nan-er-:flickrsou-suo-shen-ru/)

[雷纯锋博客 MVVM With ReactiveCocoa](http://blog.leichunfeng.com/blog/2016/02/27/mvvm-with-reactivecocoa/)

[ReactiveCocoa v2.5 源码解析之架构总览](http://blog.leichunfeng.com/blog/2015/12/25/reactivecocoa-v2-dot-5-yuan-ma-jie-xi-zhi-jia-gou-zong-lan/)

<http://www.jianshu.com/p/d262f2c55fbe#rd>

# MVVM


```
View引用了ViewModel，但反过来不行。
ViewModel引用了Model，但反过来不行。
如果我们破坏了这些规则，便无法正确地使用MVVM。 
```


MVVM是MVC模式的一个变种，它正逐渐流行起来， MVVM模式让View层代码变得更清晰，更易于测试， 严格遵守View=>ViewModel=>Model这样一个引用层次，然后通过绑定来将ViewModel的更新反映到View层上。ViewModel层决不应该维护View的引用， ViewModel层可以看作是视图的模型(model-of-the-view)，它暴露属性，以直接反映视图的状态，以及执行用户交互相关的命令。 Model层暴露服务。 针对MVVM程序的测试可以在没有UI的情况下运行。


* view ：由 MVC 中的 view 和 controller 组成，负责 UI 的展示，绑定 viewModel 中的属性，触发 viewModel 中的命令；

* viewModel ：从 MVC 的 controller 中抽取出来的展示逻辑，负责从 model 中获取 view 所需的数据，转换成 view 可以展示的数据，并暴露公开的属性和命令供 view 进行绑定；

* model ：与 MVC 中的 model 一致，包括数据模型、访问数据库的操作和网络请求等；

* binder ：在 MVVM 中，声明式的数据和命令绑定是一个隐含的约定，它可以让开发者非常方便地实现 view 和 viewModel 的同步，避免编写大量繁杂的样板化代码。


## ReactiveCocoa 

MVVM模式依赖于数据绑定，它是一个框架级别的特性，用于自动连接对象属性和UI控件。iOS没有数据绑定框架，幸运的是我们可以通过ReactiveCocoa来实现这一功能。我们从iOS开发的角度来看看MVVM模式，ViewController及其相关的UI(nib, stroyboard或纯代码的View)组成了View。

ViewModel暴露属性（`RAC(self.viewModel, searchText) = self.searchTextField.rac_textSignal;`）来表示UI状态，它同样暴露命令（`RACCommand`）来表示UI操作(通常是方法)。ViewModel负责管理基于用户交互的UI状态的改变。然而它不负责实际执行这些交互产生的的业务逻辑，那是Model的工作。

### 创建信号

**信号**

信号源代表的是随着时间而改变的值流，Streams of values over time。

**`RACObserve`**创建信号

用宏 RACObserve监测 NSString类型的 searchText。一旦 searchText的值发生变化，就发送信号。`distinctUntilChanged ` 确保信号的状态有改变，才继续下发。


```
//使用RACObserve宏来从ViewModel的searchText属性创建一个信号。一旦searchText有变化，就发出信号
    RACSignal *validSearchSignal =
    [[RACObserve(self, searchText)
      //map操作将文本转化为一个true或false值的流。
      map:^id(NSString *text) {
          return @(text.length > 3);
      }]
     
     //最后，distinctUntilChanges确保信号只有在状态改变时才发出
     distinctUntilChanged];
    
    [validSearchSignal subscribeNext:^(id x) {
        NSLog(@"search text is valid %@", x);
    }];
    
    //RACCommand是ReactiveCocoa中用于表示UI操作的一个类。它包含一个代表了UI操作的结果的信号以及标识操作当前是否被执行的一个状态。
    self.executeSearch = [[RACCommand alloc] initWithEnabled:validSearchSignal
                                                 signalBlock:^RACSignal *(id input) {
                                                     return [self executeSearchSignal];
                                                 }];
```

**`[RACSignal createSignal:subscribeOn:]`** 

```
/*
 1、定义了一个error，当用户拒绝访问时发送。
 2、和第一部分一样，类方法createSignal返回一个RACSignal实例。
 3、通过account store请求访问Twitter。此时用户会看到一个弹框来询问是否允许访问Twitter账户。
 4、在用户允许或拒绝访问之后，会发送signal事件。如果用户允许访问，会发送一个next事件，紧跟着再发送一个completed事件。如果用户拒绝访问，会发送一个error事件。
 
 这个block的返回值是一个RACDisposable对象，它允许你在一个订阅被取消时执行一些清理工作。当前的信号不需要执行清理操作，所以返回nil就可以了
 block的入参是一个subscriber实例，它遵循RACSubscriber协议，协议里有一些方法来产生事件，你可以发送任意数量的next事件，或者用error\complete事件来终止。本例中，信号发送了一个next事件来表示同意访问，随后是一个complete事件
 */
- (RACSignal *)requestAccessToTwitterSignal {
    // 1 - define an error
    NSError *accessError = [NSError errorWithDomain:RWTwitterInstantDomain
                                               code:RWTwitterInstantErrorAccessDenied
                                           userInfo:nil];
    
    // 2 - create the signal   @weakify宏让你创建一个弱引用的影子对象
    @weakify(self)
    return [RACSignal createSignal:^RACDisposable *(id subscriber) {
        // 3 - request access to twitter  @strongify让你创建一个对之前传入@weakify对象的强引用。
        @strongify(self)
        NSLog(@"request signal");
        [self.accountStore requestAccessToAccountsWithType:self.twitterAccountType
                                                   options:nil
                                                completion:^(BOOL granted, NSError *error) {
                                                    // 4 - handle the response
                                                    if (!granted) {
                                                        [subscriber sendError:accessError];
                                                    } else {
                                                        [subscriber sendNext:nil];
                                                        [subscriber sendCompleted];
                                                    }
                                                }]; 
        return nil; 
    }]; 
}
```


### 信号传递流程中的一些处理
`flattenMap`: 把block的信号转换替换为了源信号，同时还从内部信号发送事件到外部信号，使得信号继续传递下去。

`then`：等待 completed事件发射后，才订阅 block里面返回的signal。也就是阻断了 next的事件传递。只有complete，才能继续往下传信号。而且是信号还被转换了。 如果信号发出的是error，不会被then阻断，会直接调用订阅者的 error block。

`throttle`：只有当前一个next事件在指定的时间段内没有被接收到后，throttle操作才会发送next事件。 避免在一段时间内反复发送很多信号

`deliverOn`:  切换到主线程，用于更新UI。注意：如果你看一下RACScheduler类，就能发现还有很多选项，比如不同的线程优先级，或者在管道中添加延迟。

```
- (void)twitterPipe{
    /*
     一旦用户允许访问Twitter账号（希望如此），应用就应该一直监测search text filed的变化，以便搜索Twitter的内容。
     应用应该等待获取访问Twitter权限的signal发送completed事件，然后再订阅text field的signal。按顺序链接不同的signal是一个常见的问题，ReactiveCocoa处理的很好。
     
     requestAccessToTwitterSignal -> error -> subscribeNext:error 调用error。
     else
     requestAccessToTwitterSignal -> complete -> then 返回 searchText.rac_textSignal ，【转换成了 searchText.rac_textSignal的信号】,data 是NSString的text -> filter data是Bool 返回yes的话就 -> subscribeNext data是 UITextField的text。
     
     如果中间没有 then的对信号的偷换，是会调用 subscribeNext 跟 complete的。
     
     */
    @weakify(self)
    [[[[[[[self requestAccessToTwitterSignal]
    then:^RACSignal *{
            @strongify(self)
            return self.searchText.rac_textSignal;
    }]
       
    filter:^BOOL(NSString *text) {
           @strongify(self)
           return [self isValidSearchText:text];
    }]
      
    //只有当，前一个next事件在指定的时间段内没有被接收到后，throttle操作才会发送next事件。 避免在一段时间内反复发送很多信号
    throttle:0.5]
       
    flattenMap:^RACStream *(NSString *text) {
          @strongify(self)
          return [self signalForSearchWithText:text];
    }]
     
    //切换到主线程，用于更新UI。注意：如果你看一下RACScheduler类，就能发现还有很多选项，比如不同的线程优先级，或者在管道中添加延迟。
    deliverOn:[RACScheduler mainThreadScheduler]]
    subscribeNext:^(NSDictionary *jsonSearchResult) {
        NSArray *statuses = jsonSearchResult[@"statuses"];
        NSArray *tweets = [statuses linq_select:^id(id tweet) {
            return [RWTweet tweetWithStatus:tweet];
        }];
        [self.resultsViewController displayTweets:tweets];
    } error:^(NSError *error) {
         NSLog(@"An error occurred: %@", error);
    }];
}
```

###  切换线程

通过调用 deliverOn:[RACScheduler mainThreadScheduler]] 使得订阅者在主线程运行

```
    [[[[self signalForLoadingImage:tweet.profileImageUrl]
      
      takeUntil:cell.rac_prepareForReuseSignal]
     
     deliverOn:[RACScheduler mainThreadScheduler]]
     subscribeNext:^(UIImage *image) {
        cell.twitterAvatarView.image = image;
    }];
```

`[RACSignal createSignal:subscribeOn:scheduler]`在创建信号的时候，调用多一步 `scheduler`, 来指定信号源在哪个线程执行

```
-(RACSignal *)signalForLoadingImage:(NSString *)imageUrl {
    
    /*
     首先获取一个后台scheduler，来让signal不在主线程执行。然后，创建一个signal来下载图片数据，当有订阅者时创建一个UIImage。最后是subscribeOn：来确保signal在指定的scheduler上执行。
     */
    RACScheduler *scheduler = [RACScheduler
                               schedulerWithPriority:RACSchedulerPriorityBackground];
    
    return [[RACSignal createSignal:^RACDisposable *(id subscriber) {
        NSData *data = [NSData dataWithContentsOfURL:[NSURL URLWithString:imageUrl]];
        UIImage *image = [UIImage imageWithData:data];
        [subscriber sendNext:image];
        [subscriber sendCompleted];
        return nil;
    }] subscribeOn:scheduler];
    
}
```

### 避免block循环引用

@weakify宏让你创建一个弱引用的影子对象,@strongify(self)对这个影子对象的引用。

```
 
    // 2 - create the signal block
    @weakify(self)
    return [RACSignal createSignal:^RACDisposable *(id subscriber) {
        @strongify(self);
        ···
        }];
```

### 从代理创建一个信号以及其他异步行为

有代理 `@protocol OFFlickrAPIRequestDelegate <NSObject>`方法 `- (void)flickrAPIRequest:(OFFlickrAPIRequest *)inRequest didCompleteWithResponse:(NSDictionary *)inResponseDictionary;`。可以这样来通过代理创建信号,

```
// 3. 从代理方法中创建一个信号。rac_signalForSelector:fromProtocol: 方法创建了successSignal，同样也在代理方法的调用中创建了信号。
        RACSignal *successSignal = [self rac_signalForSelector:@selector(flickrAPIRequest:didCompleteWithResponse:)
                                                  fromProtocol:@protocol(OFFlickrAPIRequestDelegate)];
```
然后 代理中的参数都放在了RACTuple 这个类中。`tuple.second的意思是第二个参数的意思`

```
// 4. 处理响应
        [[[successSignal
           //代理方法每次调用时，发出的next事件会附带包含方法参数的RACTuple
           map:^id(RACTuple *tuple) {
               return tuple.second;
           }]
          map:block]
         subscribeNext:^(id x) {
             [subscriber sendNext:x];
             [subscriber sendCompleted];
         }];
```

**其他的异步行为**

```
// 代理方法
[[self
    rac_signalForSelector:@selector(webViewDidStartLoad:)
    fromProtocol:@protocol(UIWebViewDelegate)]
    subscribeNext:^(id x) {
        // 实现 webViewDidStartLoad: 代理方法
    }];

// target-action
[[self.avatarButton
    rac_signalForControlEvents:UIControlEventTouchUpInside]
    subscribeNext:^(UIButton *avatarButton) {
        // avatarButton 被点击了
    }];

// 通知
[[[NSNotificationCenter defaultCenter]
    rac_addObserverForName:kReachabilityChangedNotification object:nil]
    subscribeNext:^(NSNotification *notification) {
        // 收到 kReachabilityChangedNotification 通知
    }];

// KVO
[RACObserve(self, username) subscribeNext:^(NSString *username) {
    // 用户名发生了变化
}];
```

## 信号延迟，间隔时间内给机会反悔做决定是否发送

```
/* 不用这个方法为了节流
    RACSignal *fetchMetadata =
    [RACObserve(self, isVisible)
     filter:^BOOL(NSNumber *visible) {
         return [visible boolValue];
     }]; */
    
    // 1. 通过监听isVisible属性来创建信号。该信号发出的第一个next事件将包含这个属性的初始状态。
    // 因为我们只关心这个值的改变，所以在第一个事件上调用skip操作。
    RACSignal *visibleStateChanged = [RACObserve(self, isVisible) skip:1];
    
    // 2. 通过过滤visibleStateChanged信号来创建一个信号对，一个标识从可见到隐藏的转换，另一个标识从隐藏到可见的转换
    RACSignal *visibleSignal = [visibleStateChanged filter:^BOOL(NSNumber *value) {
        return [value boolValue];
    }];
    
    RACSignal *hiddenSignal = [visibleStateChanged filter:^BOOL(NSNumber *value) {
        return ![value boolValue];
    }];
    
    // 3. 这里是最神奇的地方。通过延迟visibleSignal信号1秒钟来创建fetchMetadata信号，在获取元数据之前暂停一会。
    // takeUntil操作确保如果cell在1秒的时间间隔内又一次隐藏时，来自visibleSignal的next事件被挂起且不获取元数据。
    RACSignal *fetchMetadata = [[visibleSignal delay:1.0f]
                                takeUntil:hiddenSignal];
```


## 多个信号合并成一个信号

```
- (RACSignal *)flickrImageMetadata:(NSString *)photoId {
    
    //请求收藏数
    RACSignal *favorites = [self signalFromAPIMethod:@"flickr.photos.getFavorites"
                                           arguments:@{@"photo_id": photoId}
                                           transform:^id(NSDictionary *response) {
                                               NSString *total = [response valueForKeyPath:@"photo.total"];
                                               return total;
                                           }];
    
    //请求评论数
    RACSignal *comments = [self signalFromAPIMethod:@"flickr.photos.getInfo"
                                          arguments:@{@"photo_id": photoId}
                                          transform:^id(NSDictionary *response) {
                                              NSString *total = [response valueForKeyPath:@"photo.comments._text"];
                                              return total;
                                          }];
    
    //一旦创建了两个信号，combineLatest:reduce:方法生成一个新的信号来组合两者。
    //这个方法等待源信号的一个next事件。reduce块使用它们的内容来调用，其结果变成联合信号的next事件。
    return [RACSignal combineLatest:@[favorites, comments] reduce:^id(NSString *favs, NSString *coms){
        RWTFlickrPhotoMetadata *meta = [RWTFlickrPhotoMetadata new];
        meta.comments = [coms integerValue];
        meta.favorites = [favs integerValue];
        return  meta;
    }];
}
```

