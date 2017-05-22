---
title: NSURLSession
date: 2014-07-14 16:22:12
tags: [iOS,网络]
---

# 概览

和 NSURLConnection 一样，NSURLSession 指的也不仅是同名类 NSURLSession，还包括一系列相互关联的类。NSURLSession 包括了与之前相同的组件，NSURLRequest 与 NSURLCache，但是把 NSURLConnection 替换成了 NSURLSession、NSURLSessionConfiguration 以及 NSURLSessionTask 的 3 个子类：NSURLSessionDataTask，NSURLSessionUploadTask，NSURLSessionDownloadTask。 代理方法也有一些变化，在 `NSURLConnection` 中有两个 delegate 方法可以表明一个网络请求已经结束：`NSURLConnectionDataDelegate` 中的 `-connectionDidFinishLoading: `和 `NSURLConnectionDelegate` 中的` -connection:didFailWithError:`，而在 `NSURLSession` 中改为一个 delegate 方法：`NSURLSessionTaskDelegate` 的` -URLSession:task:didCompleteWithError:`

与 NSURLConnection 相比，NSURLsession 最直接的改进就是可以配置每个 session 的缓存，协议，cookie，以及证书策略（credential policy），甚至跨程序共享这些信息。这将允许程序和网络基础框架之间相互独立，不会发生干扰。每个 NSURLSession 对象都由一个 NSURLSessionConfiguration 对象来进行初始化，后者指定了刚才提到的那些策略以及一些用来增强移动设备上性能的新选项。

# NSURLSessionTask的三个子类——三种会话模式
## NSURLSessionDataTask
Data task 可以通过 NSURL 或 NSURLRequest 创建（使用前者相当于是使用一个对于该 URL 进行标准 GET 请求的 NSURLRequest，这是一种快捷方法）：

```
 NSURL *URL = [NSURL URLWithString:@"http://example.com"];
 NSURLRequest *request = [NSURLRequest requestWithURL:URL];

 NSURLSession *session = [NSURLSession sharedSession];
 NSURLSessionDataTask *task = [session dataTaskWithRequest:request
                                         completionHandler:
     ^(NSData *data, NSURLResponse *response, NSError *error) {
         // ...
     }];

 [task resume];
```

## NSURLSessionUploadTask

Upload task 的创建需要使用一个 request，另外加上一个要上传的 NSData 对象或者是一个本地文件的路径对应的 NSURL

```
NSURL *URL = [NSURL URLWithString:@"http://example.com/upload"];
 NSURLRequest *request = [NSURLRequest requestWithURL:URL];
 NSData *data = ...;

 NSURLSession *session = [NSURLSession sharedSession];
 NSURLSessionUploadTask *uploadTask = [session uploadTaskWithRequest:request
                                                            fromData:data
                                                   completionHandler:
     ^(NSData *data, NSURLResponse *response, NSError *error) {
         // ...
     }];

 [uploadTask resume];
```

## NSURLSessionDownloadTask
Download task 也需要一个 request，不同之处在于 completionHandler 这个 block。Data task 和 upload task 会在任务完成时一次性返回，但是 Download task 是将数据一点点地写入本地的临时文件。所以在 completionHandler 这个 block 里，我们需要把文件从一个临时地址移动到一个永久的地址保存起来：

```
NSURL *URL = [NSURL URLWithString:@"http://example.com/file.zip"];
 NSURLRequest *request = [NSURLRequest requestWithURL:URL];

 NSURLSession *session = [NSURLSession sharedSession];
 NSURLSessionDownloadTask *downloadTask = [session downloadTaskWithRequest:request
                                                         completionHandler:
    ^(NSURL *location, NSURLResponse *response, NSError *error) {
        NSString *documentsPath = [NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES) firstObject];
        NSURL *documentsDirectoryURL = [NSURL fileURLWithPath:documentsPath];
        NSURL *newFileLocation = [documentsDirectoryURL URLByAppendingPathComponent:[[response URL] lastPathComponent]];
        [[NSFileManager defaultManager] copyItemAtURL:location toURL:newFileLocation error:nil];
    }];

 [downloadTask resume];
```


# NSURLSessionConfiguration
NSURLSessionConfiguration 对象用于对 NSURLSession 对象进行初始化。NSURLSessionConfiguration 对以前 NSMutableURLRequest 所提供的网络请求层的设置选项进行了扩充，提供给我们相当大的灵活性和控制权。从指定可用网络，到 cookie，安全性，缓存策略，再到使用自定义协议，启动事件的设置，以及用于移动设备优化的几个新属性，使用 NSURLSessionConfiguration 可以找到几乎任何你想要进行配置的选项。

`注意：`NSURLSession 在初始化时会把配置它的 NSURLSessionConfiguration 对象进行一次 copy，并保存到自己的 configuration 属性中，而且这个属性是只读的。因此之后再修改最初配置 session 的那个 configuration 对象对于 session 是没有影响的。也就是说，configuration 只在初始化时被读取一次，之后都是不会变化的。

NSURLSessionConfiguration有三个工厂方法

* defaultSessionConfiguration

`+defaultSessionConfiguration` 返回一个标准的 configuration，这个配置实际上与 NSURLConnection 的网络堆栈（networking stack）是一样的，具有相同的共享 NSHTTPCookieStorage，共享 NSURLCache 和共享 NSURLCredentialStorage。

* ephemeralSessionConfiguration

`+ephemeralSessionConfiguration` 返回一个预设配置，这个配置中不会对缓存，Cookie 和证书进行持久性的存储。这对于实现像秘密浏览这种功能来说是很理想的。

* backgroundSessionConfiguration

+backgroundSessionConfiguration:(NSString *)identifier 的独特之处在于，它会创建一个后台 session。后台 session 不同于常规的，普通的 session，它甚至可以在应用程序挂起，退出或者崩溃的情况下运行上传和下载任务。初始化时指定的标识符，被用于向任何可能在进程外恢复后台传输的守护进程（daemon）提供上下文。  `另外`：NSURLSessionDownloadTask才支持后台下载，NSURLSessionDataTask不支持。

## 配置属性
NSURLSessionConfiguration 拥有 20 个配置属性。熟练掌握这些配置属性的用处，可以让应用程序充分地利用其网络环境。

HTTPAdditionalHeaders 指定了一组默认的可以设置出站请求（outbound request）的数据头。这对于跨 session 共享信息，如内容类型，语言，用户代理和身份认证，是很有用的。

```
NSString *userPasswordString = [NSString stringWithFormat:@"%@:%@", user, password];
NSData * userPasswordData = [userPasswordString dataUsingEncoding:NSUTF8StringEncoding];
NSString *base64EncodedCredential = [userPasswordData base64EncodedStringWithOptions:0];
NSString *authString = [NSString stringWithFormat:@"Basic %@", base64EncodedCredential];
NSString *userAgentString = @"AppName/com.example.app (iPhone 5s; iOS 7.0.2; Scale/2.0)";

configuration.HTTPAdditionalHeaders = @{@"Accept": @"application/json",
                                        @"Accept-Language": @"en",
                                        @"Authorization": authString,
                                        @"User-Agent": userAgentString};
```

* networkServiceType 

对标准的网络流量，网络电话，语音，视频，以及由一个后台进程使用的流量进行了区分。大多数应用程序都不需要设置这个。

* allowsCellularAccess 和 discretionary 

被用于节省通过蜂窝网络连接的带宽。对于后台传输的情况，推荐大家使用 discretionary 这个属性，而不是 allowsCellularAccess，因为前者会把 WiFi 和电源的可用性考虑在内。

* timeoutIntervalForRequest 和 timeoutIntervalForResource 

分别指定了对于请求和资源的超时间隔。许多开发人员试图使用 timeoutInterval 去限制发送请求的总时间，但其实它真正的含义是：分组（packet）之间的时间。实际上我们应该使用 timeoutIntervalForResource 来规定整体超时的总时间，但应该只将其用于后台传输，而不是用户实际上可能想要去等待的任何东西。

* HTTPMaximumConnectionsPerHost 

是 Foundation 框架中 URL 加载系统的一个新的配置选项。它曾经被 NSURLConnection 用于管理私有的连接池。现在有了 NSURLSession，开发者可以在需要时限制连接到特定主机的数量。

* HTTPShouldUsePipelining 

这个属性在 NSMutableURLRequest 下也有，它可以被用于开启 HTTP 管线化（HTTP pipelining），这可以显着降低请求的加载时间，但是由于没有被服务器广泛支持，默认是禁用的。

* sessionSendsLaunchEvents 

是另一个新的属性，该属性指定该 session 是否应该从后台启动。

* connectionProxyDictionary 

指定了 session 连接中的代理服务器。同样地，大多数面向消费者的应用程序都不需要代理，所以基本上不需要配置这个属性。


## Cookie策略
* HTTPCookieStorage 

存储了 session 所使用的 cookie。默认情况下会使用 NSHTTPCookieShorage 的 +sharedHTTPCookieStorage 这个单例对象，这与 NSURLConnection 是相同的。

* HTTPCookieAcceptPolicy 

决定了什么情况下 session 应该接受从服务器发出的 cookie。

* HTTPShouldSetCookies 

指定了请求是否应该使用 session 存储的 cookie，即 HTTPCookieSorage 属性的值。

## 安全策略
URLCredentialStorage 存储了 session 所使用的证书。默认情况下会使用 NSURLCredentialStorage 的 +sharedCredentialStorage 这个单例对象，这与 NSURLConnection 是相同的。

TLSMaximumSupportedProtocol 和 TLSMinimumSupportedProtocol 确定 session 是否支持 SSL 协议。

## 缓存策略
URLCache 是 session 使用的缓存。默认情况下会使用 NSURLCache 的 +sharedURLCache 这个单例对象，这与 NSURLConnection 是相同的。
requestCachePolicy 指定了一个请求的缓存响应应该在什么时候返回。这相当于 NSURLRequest 的 -cachePolicy 方法。


# 例子
下载图片的例子

1、首先创建session：

```
NSString *imageUrl =
@"http://xxx.xxx.com/img";

NSURLSessionConfiguration *sessionConfig =[NSURLSessionConfiguration defaultSessionConfiguration];

NSURLSession *session =[NSURLSession sessionWithConfiguration:sessionConfig
                            delegate:self
                       delegateQueue:nil];
```

2、然后创建 urlSessionTask的子类，downloadTask来处理下载

```
NSURLSessionDownloadTask *getImageTask = [session 
						downloadTaskWithURL:[NSURL URLWithString:imageUrl]
						  completionHandler:^(NSURL *location,NSURLResponse *response,NSError *error) {
									UIImage *downloadedImage = [UIImage imageWithData:[NSData dataWithContentsOfURL:location]];
									dispatch_async(dispatch_get_main_queue(), ^{
									    // do stuff with image
									    _imageWithBlock.image = downloadedImage;
									});
						  }];

[getImageTask resume];
```

3、代理

以NSURLSessionDownloadTask为例，如果要用delegate监控下载，创建NSURLSessionDownloadTask的时候不能带completionHandler这个block,跟踪下载进度

```
-(void)URLSession:(NSURLSession *)session
     downloadTask:(NSURLSessionDownloadTask *)downloadTask
     didWriteData:(int64_t)bytesWritten
totalBytesWritten:(int64_t)totalBytesWritten
totalBytesExpectedToWrite:(int64_t)totalBytesExpectedToWrite
{
  NSLog(@"%f / %f", (double)totalBytesWritten,(double)totalBytesExpectedToWrite);
}
```

监控下载完成通知

```
 -(void)URLSession:(NSURLSession *)session downloadTask:(NSURLSessionDownloadTask *)downloadTask didFinishDownloadingToURL:(NSURL *)location { 
 // use code above from completion handler 
 }
```






