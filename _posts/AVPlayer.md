---
title: AVPlayer
date: 2015-11-01 11:35:08
tags: [iOS,视频]
---


## 前言
---
需要实现播放器功能，需求比较复杂，所以使用AVPlayer来自定义。
研究方法：
1. 阅读AVFoundation Programming Guide，阅读相关类的头文件（AVPlayer，AVAssets，AVPlayItem）
2. 阅读相关播放器的开源代码（WMPlayer，ZFPlayer）
3. 写Demo
4. 实现项目需求

## AVFoundation
---

### Assets

* 创建Assets：

```
AVAsset *movieAsset  = [[AVURLAsset alloc]initWithURL:[NSURL fileURLWithPath:urlString] options:nil];
AVPlayerItem *playerItem = [AVPlayerItem playerItemWithAsset:movieAsset];

// 或者
//   AVURLAssetPreferPreciseDurationAndTimingKey，它的value是一个boolean类型（用NSValue包装的对象），这个值表示asset是否提供一个精确的duration。获取asset精确的duration需要很多处理时间，使用一个预估的duration效率比较高并且对播放来说足够
NSDictionary *options = @{ AVURLAssetPreferPreciseDurationAndTimingKey : @YES };
AVURLAsset *anAssetToUseInAComposition = [[AVURLAsset alloc] initWithURL:url options:options];
```

* 使用Assets
  * 使用AVAsynchronousKeyValueLoading协议获取诸如duration这些值，因为当一个assert第一次被加载，大多数属性的值是AVKeyValueStatusUnknown状态，为了获取一个或多个属性的值，你要调用loadValuesAsynchronouslyForKeys:completionHandler:，在comletiton handler里面，你可以根据属性的状态做任何合适的处理。

```
NSURL *url = <#A URL that identifies an audiovisual asset such as a movie file#>;
AVURLAsset *anAsset = [[AVURLAsset alloc] initWithURL:url options:nil];
NSArray *keys = @[@"duration"];
 
[asset loadValuesAsynchronouslyForKeys:keys completionHandler:^() {
 
    NSError *error = nil;
    AVKeyValueStatus tracksStatus = [asset statusOfValueForKey:@"duration" error:&error];
    switch (tracksStatus) {
        case AVKeyValueStatusLoaded:
            [self updateUserInterfaceForDuration];
            break;
        case AVKeyValueStatusFailed:
            [self reportError:error forAsset:asset];
            break;
        case AVKeyValueStatusCancelled:
            // Do whatever is appropriate for cancelation.
            break;
   }
}];
```

* 剪切视频、视频转码

### AVPlayItem
* 不能直接把asset传给AVPlayer对象，你应该提供AVPlayerItem对象给AVPlayer。一个player item管理着和它相关的asset。一个player item包括player item tracks-（AVPlayerItemTrack对象，表示asset中的tracks）
* 基于http live stream：用url初始化一个AVPlayerItem（**http live stream的情况下不能直接创建AVAsset对象**,当你关联一个player item到player的时候，这个播放器开始准备播放。当它可以播放的时候，player item会创建AVAsset和AVAssetTrack对象，这些对象可以用来检查live stream的内容。为了获取stream的时间，可以通过kvo的方式观察player item的duration的属性。当可以播放的时候，这个属性被设置为正确的值，这时就可以获取时间。）


```
NSURL *url = [NSURL URLWithString:@"http://devimages.apple.com/iphone/samples/bipbop/bipbopall.m3u8";
self.playerItem = [AVPlayerItem playerItemWithURL:url];
[playerItem addObserver:self forKeyPath:@"status" options:0 context:&ItemStatusContext];
self.player = [AVPlayer playerWithPlayerItem:playerItem];
```


### AVPlayer
* 正如assets和items一样，初始化一个player之后并不表明可以马上播放，你需要观察player的status属性，当status变为AVPlayerStatusReadyToPlay时表示可以播放了，你也需要观察curretItem属性来访问player item。

```
[self.playerItem addObserver:self forKeyPath:@"status" options:NSKeyValueObservingOptionNew context:nil];

- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary *)change context:(void *)context {
    AVPlayerItem *playerItem = (AVPlayerItem *)object;
    if ([keyPath isEqualToString:@"status"]) {
        if ([playerItem status] == AVPlayerStatusReadyToPlay) {
            NSLog(@"AVPlayerStatusReadyToPlay");
            
            // ...
            
        } else if ([playerItem status] == AVPlayerStatusFailed) {
            NSLog(@"AVPlayerStatusFailed");
            
            // ...
            
        }
    } else if ([keyPath isEqualToString:@"loadedTimeRanges"]) {
        // ...
    }
}
```

* 播放：`[player play]`
* rate：1.0表示正常播放，0.0表示暂停，负数为逆向播放（-0.0~-1.0）
* seekToTime:

```
//(CMTimeMake(a,b)    a当前第几帧, b每秒钟多少帧.当前播放时间a/b

CMTime fiveSecondsIn = CMTimeMake(5, 1);
[player seekToTime:fiveSecondsIn];
```
* 检测播放状态：（**在主线程注册和取消kvo**）
  * status:
  
```
- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object
                        change:(NSDictionary *)change context:(void *)context {
 
    if (context == <#Player status context#>) {
        AVPlayer *thePlayer = (AVPlayer *)object;
        if ([thePlayer status] == AVPlayerStatusFailed) {
            NSError *error = [<#The AVPlayer object#> error];
            // Respond to error: for example, display an alert sheet.
            return;
        }
        // Deal with other status change if appropriate.
    }
    // Deal with other change notifications if appropriate.
    [super observeValueForKeyPath:keyPath ofObject:object
           change:change context:context];
    return;
}
```

* 监听视频准备播放的状态:监听AVPlayerLayer的readyForDisplay属性，当layer有可显示的内容时，会发送一个notification。
* 时间:可以使用addPeriodicTimeObserverForInterval:queue:usingBlock: 或者 addBoundaryTimeObserverForTimes:queue:usingBlock:来跟踪播放的进度，根据这个进度更新UI，比如播放了多少时间，还剩多少时间，或者其他的UI状态。

```
// Assume a property: @property (strong) id playerObserver;
 
Float64 durationSeconds = CMTimeGetSeconds([<#An asset#> duration]);
CMTime firstThird = CMTimeMakeWithSeconds(durationSeconds/3.0, 1);
CMTime secondThird = CMTimeMakeWithSeconds(durationSeconds*2.0/3.0, 1);
NSArray *times = @[[NSValue valueWithCMTime:firstThird], [NSValue valueWithCMTime:secondThird]];
 
self.playerObserver = [<#A player#> addBoundaryTimeObserverForTimes:times queue:NULL usingBlock:^{
 
    NSString *timeDescription = (NSString *)
        CFBridgingRelease(CMTimeCopyDescription(NULL, [self.player currentTime]));
    NSLog(@"Passed a boundary at %@", timeDescription);
}];
```

* 播放结束的回调：向通知中心注册 AVPlayerItemDidPlayToEndTimeNotification 通知，当播放结束的时候可以收到一个结束的通知。	


## 用AVPlayer实现一个简单的播放器

### 大致步骤
1. 用AVPlayerLayer配置一个view 
2. 创建一个AVPlayer
3. 用video文件创建一个AVPlayerItem对象，并且用kvo观察他的status 
4. 当收到item的状态变成可播放的时候，播放按钮启用
5. 播放，结束之后把播放头设置到起始位置


### 根据需求来实现功能

* 添加手势操作（例如滑动调节音量、滑动调节亮度等），例如：

```
UITapGestureRecognizer* doubleTap = [[UITapGestureRecognizer alloc] initWithTarget:self action:@selector(handleDoubleTap)];
doubleTap.numberOfTapsRequired = 2; // 双击
[self addGestureRecognizer:doubleTap];


- (void)handleDoubleTap{
    self.playOrPauseBtn.selected = !self.playOrPauseBtn.selected;
    if (self.player.rate != 1.f) {
        if ([self currentTime] == self.duration)
            [self setCurrentTime:0.f];
        [self.player play];
    } else {
        [self.player pause];
    }
}
```

* 获取缓冲时间：

```
- (NSTimeInterval)availableDuration {
    NSArray *loadedTimeRanges = [[self.playerView.player currentItem] loadedTimeRanges];
    CMTimeRange timeRange = [loadedTimeRanges.firstObject CMTimeRangeValue];// 获取缓冲区域
    CGFloat startSeconds = CMTimeGetSeconds(timeRange.start);
    CGFloat durationSeconds = CMTimeGetSeconds(timeRange.duration);
    NSTimeInterval result = startSeconds + durationSeconds;// 计算缓冲总进度
    return result;
}
```

* 拖动滑动条来更新视频进度：

```
- (IBAction)videoSlierChangeValue:(id)sender {
    UISlider *slider = (UISlider *)sender;
    
    if (slider.value == 0.000000) {
        __weak typeof(self) weakSelf = self;
        [self.playerView.player seekToTime:kCMTimeZero completionHandler:^(BOOL finished) {
            __strong __typeof__(weakSelf) strongSelf = weakSelf;
            [strongSelf.playerView.player play];
        }];
    }
}

- (IBAction)videoSlierChangeValueEnd:(id)sender {
    UISlider *slider = (UISlider *)sender;
    CMTime changedTime = CMTimeMakeWithSeconds(slider.value, 1);
    
    __weak typeof(self) weakSelf = self;
    [self.playerView.player seekToTime:changedTime completionHandler:^(BOOL finished) {
        __strong __typeof__(weakSelf) strongSelf = weakSelf;
        [strongSelf.playerView.player play];
        [strongSelf.stateButton setTitle:@"Stop" forState:UIControlStateNormal];
    }];
}
```

* 利用第三方对AVPlayer的封装减少工作量，但是发现很多第三方库都有封装自己的界面逻辑，界面的可定制性不是很高。
* AVPlayer进行边下边播，使用AVAssetResourceLoader回调下载，通过实现下面的回调方法来控制逻辑

```
- (BOOL)resourceLoader:(AVAssetResourceLoader *)resourceLoader shouldWaitForLoadingOfRequestedResource:(AVAssetResourceLoadingRequest *)loadingRequest NS_AVAILABLE(10_9, 6_0);

- (void)resourceLoader:(AVAssetResourceLoader *)resourceLoader didCancelLoadingRequest:(AVAssetResourceLoadingRequest *)loadingRequest NS_AVAILABLE(10_9, 7_0);
```


# 下载M3U8视频文件
# M3U8视频下载

## MP4视频下载

* 与下载普通文件一样

```
- (void)downloadWithUrlString:(NSString *)urlStr {
    NSURL *url = [NSURL URLWithString:urlStr];
    NSURLRequest *request = [NSURLRequest requestWithURL:url];
    NSURLSessionDownloadTask *downloadTask = [self.manager downloadTaskWithRequest:request progress:^(NSProgress * _Nonnull downloadProgress) {
        
        NSLog(@"------ %f",downloadProgress.fractionCompleted);
        
    } destination:^NSURL * _Nonnull(NSURL * _Nonnull targetPath, NSURLResponse * _Nonnull response) {
        
        NSURL *documentsDirectoryURL = [[NSFileManager defaultManager] URLForDirectory:NSDocumentDirectory inDomain:NSUserDomainMask appropriateForURL:nil create:NO error:nil];
        return [documentsDirectoryURL URLByAppendingPathComponent:[response suggestedFilename]];
        
    } completionHandler:^(NSURLResponse * _Nonnull response, NSURL * _Nullable filePath, NSError * _Nullable error) {
        NSLog(@"File downloaded to: %@", filePath);
    }];
    [downloadTask resume];
}
```

## M3U8视频下载

### M3U8简介
事例:**http://cache.m.iqiyi.com/dc/dt/-0-f45bc84a7ea643209b29a72b0c1e385f/text/c0f2ab25x11111111x777a6171/20160511/8d/7c/5ec47e98afaf84573550eaadc2973d19.m3u8**

```
#EXTM3U
#EXT-X-TARGETDURATION:10
#EXTINF:9,
http://sf.video.qiyi.com/videos/vip/20160511/8d/7c/5cb69bf73d7831a0bae0b5cc694f1421.ts?start=0&end=1644168&hsize=13478&tag=0&v=&contentlength=1031932&qd_uid=0&qd_vip=0&qd_src=f45bc84a7ea643209b29a72b0c1e385f&qd_tm=1463971070454&qd_ip=58.62.17.190&qd_sc=3b82ce49bf1d65c849410de7670fcc08
#EXTINF:3,
http://sf.video.qiyi.com/videos/vip/20160511/8d/7c/5cb69bf73d7831a0bae0b5cc694f1421.ts?start=751894&end=2117577&hsize=13478&tag=1&v=&contentlength=682440&qd_uid=0&qd_vip=0&qd_src=f45bc84a7ea643209b29a72b0c1e385f&qd_tm=1463971070454&qd_ip=58.62.17.190&qd_sc=3b82ce49bf1d65c849410de7670fcc08
#EXTINF:3,
http://sf.video.qiyi.com/videos/vip/20160511/8d/7c/5cb69bf73d7831a0bae0b5cc694f1421.ts?start=984436&end=2556868&hsize=13478&tag=1&v=&contentlength=490304&qd_uid=0&qd_vip=0&qd_src=f45bc84a7ea643209b29a72b0c1e385f&qd_tm=1463971070454&qd_ip=58.62.17.190&qd_sc=3b82ce49bf1d65c849410de7670fcc08
#EXTINF:2,
http://sf.video.qiyi.com/videos/vip/20160511/8d/7c/5cb69bf73d7831a0bae0b5cc694f1421.ts?start=1644168&end=2606170&hsize=13478&tag=1&v=&contentlength=452892&qd_uid=0&qd_vip=0&qd_src=f45bc84a7ea643209b29a72b0c1e385f&qd_tm=1463971070454&qd_ip=58.62.17.190&qd_sc=3b82ce49bf1d65c849410de7670fcc08
#EXTINF:3,
http://sf.video.qiyi.com/videos/vip/20160511/8d/7c/5cb69bf73d7831a0bae0b5cc694f1421.ts?start=2117577&end=2896401&hsize=13478&tag=1&v=&contentlength=60536&qd_uid=0&qd_vip=0&qd_src=f45bc84a7ea643209b29a72b0c1e385f&qd_tm=1463971070454&qd_ip=58.62.17.190&qd_sc=3b82ce49bf1d65c849410de7670fcc08
#EXTINF:9,
http://sf.video.qiyi.com/videos/vip/20160511/8d/7c/5cb69bf73d7831a0bae0b5cc694f1421.ts?start=2556868&end=6047605&hsize=13478&tag=1&v=&contentlength=2553228&qd_uid=0&qd_vip=0&qd_src=f45bc84a7ea643209b29a72b0c1e385f&qd_tm=1463971070454&qd_ip=58.62.17.190&qd_sc=3b82ce49bf1d65c849410de7670fcc08
#EXTINF:9,
http://sf.video.qiyi.com/videos/vip/20160511/8d/7c/5cb69bf73d7831a0bae0b5cc694f1421.ts?start=4402162&end=6958244&hsize=13478&tag=1&v=&contentlength=1547052&qd_uid=0&qd_vip=0&qd_src=f45bc84a7ea643209b29a72b0c1e385f&qd_tm=1463971070454&qd_ip=58.62.17.190&qd_sc=3b82ce49bf1d65c849410de7670fcc08
#EXTINF:9,
http://sf.video.qiyi.com/videos/vip/20160511/8d/7c/5cb69bf73d7831a0bae0b5cc694f1421.ts?start=6411160&end=9534780&hsize=13478&tag=1&v=&contentlength=1637480&qd_uid=0&qd_vip=0&qd_src=f45bc84a7ea643209b29a72b0c1e385f&qd_tm=1463971070454&qd_ip=58.62.17.190&qd_sc=3b82ce49bf1d65c849410de7670fcc08
#EXTINF:9,
http://sf.video.qiyi.com/videos/vip/20160511/8d/7c/5cb69bf73d7831a0bae0b5cc694f1421.ts?start=7461096&end=13651259&hsize=13478&tag=1&v=&contentlength=3473300&qd_uid=0&qd_vip=0&qd_src=f45bc84a7ea643209b29a72b0c1e385f&qd_tm=1463971070454&qd_ip=58.62.17.190&qd_sc=3b82ce49bf1d65c849410de7670fcc08
```

* #EXTM3U作为文件的头部，必须是第一行
* #EXTINF指示多媒体文件的信息，包括播放时间和标题

### 处理M3U8文件
* 提取所有片段链接
* 按顺序一个一个地下载所有ts片段

```
- (void)downloadAllTsWithM3U8List:(M3U8SegmentInfoList *)m3u8 {
    self.manager.completionQueue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    dispatch_semaphore_t semaphore = dispatch_semaphore_create(1);
    for (NSInteger i = 0 ; i < m3u8.count; i++) {
        M3U8SegmentInfo *info = [m3u8 segmentInfoAtIndex:i];
        NSURLRequest *request = [NSURLRequest requestWithURL:info.mediaURL];
        dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
        NSURLSessionDownloadTask *downloadTask = [self.manager downloadTaskWithRequest:request progress:nil destination:^NSURL * _Nonnull(NSURL * _Nonnull targetPath, NSURLResponse * _Nonnull response) {
            
            NSURL *documentsDirectoryURL = [[NSFileManager defaultManager] URLForDirectory:NSDocumentDirectory inDomain:NSUserDomainMask appropriateForURL:nil create:NO error:nil];
            return [documentsDirectoryURL URLByAppendingPathComponent:[NSString stringWithFormat:@"%zd.ts",i]];
            
        } completionHandler:^(NSURLResponse * _Nonnull response, NSURL * _Nullable filePath, NSError * _Nullable error) {
            NSLog(@"File downloaded to: %@", filePath);
            dispatch_semaphore_signal(semaphore);
        }];
        [downloadTask resume];
    }
}
```

* 注意有可能需要做链接的拼接

### 创建本地M3U8文件
* 按照格式创建本地M3U8文件

```
- (void)createLocalM3U8:(M3U8SegmentInfoList *)m3u8 {
    NSString *pathPrefix = [NSSearchPathForDirectoriesInDomains(NSDocumentDirectory,NSUserDomainMask,YES) objectAtIndex:0];
    NSString *fullPath = [pathPrefix stringByAppendingPathComponent:@"movie.m3u8"];
    NSLog(@"full path : %@",fullPath);
    //创建文件头部
    NSString* head = @"#EXTM3U\n#EXT-X-TARGETDURATION:30\n#EXT-X-VERSION:2\n#EXT-X-DISCONTINUITY\n";
    NSMutableString *body = [NSMutableString string];
    for (NSInteger i = 0; i < m3u8.count; i++) {
        NSString *tsFileName = [NSString stringWithFormat:@"%zd.ts\n",i];
        NSString *durationStr = [NSString stringWithFormat:@"#EXTINF:%zd,\n",(NSInteger)[m3u8 segmentInfoAtIndex:i].duration];
        [body appendFormat:@"%@%@",durationStr,tsFileName];
    }
    NSString* end = @"#EXT-X-ENDLIST";
    NSString *fileStr = [NSString stringWithFormat:@"%@%@%@",head,body,end];
    [fileStr writeToFile:fullPath atomically:YES encoding:NSUTF8StringEncoding error:nil];
}

```

* 选择支持M3U8文件解析的播放器来播放
* 根据需求添加回调（下载速度，progress等等）


## 参考资料
* Demo:[AVPlayerDemo.zip](/uploads/1cba54cbf70d348e95035c967c2e0608/AVPlayerDemo.zip)
* [使用 AVPlayer 进行多视频播放](http://www.devtoutiao.com/t?url=http://t.cn/RGkOLfs)
* [AVFoundation Programming Guide](https://developer.apple.com/library/ios/documentation/AudioVideo/Conceptual/AVFoundationPG/Articles/00_Introduction.html#//apple_ref/doc/uid/TP40010188-CH1-SW3)
* [AVPlayer](https://developer.apple.com/library/ios/documentation/AVFoundation/Reference/AVPlayer_Class/index.html#//apple_ref/doc/c_ref/AVPlayer)
* [CMTime的概念](https://zwo28.wordpress.com/2015/03/06/%E8%A7%86%E9%A2%91%E5%90%88%E6%88%90%E4%B8%ADcmtime%E7%9A%84%E7%90%86%E8%A7%A3%EF%BC%8C%E4%BB%A5%E5%8F%8A%E5%88%A9%E7%94%A8cmtime%E5%AE%9E%E7%8E%B0%E8%BF%87%E6%B8%A1%E6%95%88%E6%9E%9C/)
* [ZFPlayer](https://github.com/renzifeng/ZFPlayer)
* [HTTP Live Streaming](https://zh.wikipedia.org/wiki/HTTP_Live_Streaming)
* <https://github.com/durfu/DFUProgressiveDownloadVideo>
* <https://github.com/devSC/WSY_XMHelper>
* <https://zh.wikipedia.org/wiki/M3U>