
---
title: NSURLProtocol 配hosts（内含例子）
date: 2016-04-5 17:29:51
categories: [iOS,Objective-C,网络]
tags: [iOS,Objective-C,网络]
---
一些需要配hosts的情形：

* 产品开发，开发的过程中跟后台或者前端的同事联调。

* 一些内部使用的后台，出于某些原因，没有购买域名直接使用自定义的域名。

配hosts的方式：

* 把iOS设备越狱了，使用iFile等工具直接修改hosts文件。(hosts文件的目录为：/etc/hosts)

* 设备没有越狱，电脑开个代理给设备连接。电脑上修改hosts文件修改hosts

上面修改hosts的方式有些不方便。**都要修改hosts文件**。 下面介绍一种不需要修改hosts文件，在开发过程中可以自由切换线上线下环境的配置hosts的方法。 

## NSURLProtocol

详细的URLProtocol用法，这里就不赘述了。可以自己去google或者参考[这篇文章](https://www.raywenderlich.com/59982/nsurlprotocol-tutorial)。 大概说一下NSURLProtocol的作用以及用法

`NSURLProtocol的作用`

NSURLProtocol可以捕获App上所有的网络请求（包括UIWebView中JS产生的一些请求），进行一些中间的操作，比如可以进行缓存的优化、hosts的修改、请求重定向、自定义网络请求的返回结果

`NSURLProtocol的用法`

定义一子类MyURLProtocol继承NSURLProtocol。实现以下方法

* +canInitWithRequest:

URL Loading System当接收到一个请求，它会去搜索已经注册过的protocol去处理这个请求。每个自定义的protocol都会告诉系统，他是否会处理这个请求。就是通过这个函数的返回值来判断.如果没有protocol可以处理这个请求，系统会自己去处理这个请求

```
+ (BOOL)canInitWithRequest:(NSURLRequest *)request {
    //看看是否已经处理过了，防止无限循环
        if ([NSURLProtocol propertyForKey:@"URLProtocolHandledKey" inRequest:request]) {
            return NO;
        }
        
        return YES;
}
```


* +canonicalRequestForRequest:
一般是在这个函数修改捕获的request的header等。

```
+ (NSURLRequest *)canonicalRequestForRequest:(NSURLRequest *)request {
    return request;
}
```


* +requestIsCacheEquivalent:toRequest: 判断你定制的两个请求是否一样，如果一样应该使用同样的缓存. 一般是给父类调用

```
+ (BOOL)requestIsCacheEquivalent:(NSURLRequest *)a toRequest:(NSURLRequest *)b {
    return [super requestIsCacheEquivalent:a toRequest:b];
}
```

* -startLoading -stopLoading  这里是URL Loading System通知你的URLprotocol开始加载处理一个请求/停止一个请求。一般是再这里开始自己的NSURLConnection

```
- (void)startLoading
{
    NSMutableURLRequest *mutableReqeust = [[self request] mutableCopy];
    //标示改request已经处理过了，防止无限循环。因为这里用NSURLConnection去请求数据，又会走一遍Protocol的流程
    [NSURLProtocol setProperty:@YES forKey:@"URLProtocolHandledKey" inRequest:mutableReqeust];
    self.connection = [NSURLConnection connectionWithRequest:mutableReqeust delegate:self];
}

- (void)stopLoading
{
    [self.connection cancel];
}

```
* NSURLConnectionDataDelegate  URL

```
- (void) connection:(NSURLConnection *)connection didReceiveResponse:(NSURLResponse *)response {
    [self.client URLProtocol:self didReceiveResponse:response cacheStoragePolicy:NSURLCacheStorageNotAllowed];
}

- (void) connection:(NSURLConnection *)connection didReceiveData:(NSData *)data {
    [self.client URLProtocol:self didLoadData:data];
}

- (void) connectionDidFinishLoading:(NSURLConnection *)connection {
    [self.client URLProtocolDidFinishLoading:self];
}

- (void)connection:(NSURLConnection *)connection didFailWithError:(NSError *)error {
    [self.client URLProtocol:self didFailWithError:error];
    NSLog(@"error: %@",error);
}
```

`注册MyURLProtocol，使得自定义的URLProtocol生效`

注册了自定义的URLProtocol后，就能拦截系统的网络请求。包括UIWebView上的请求

```
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    [NSURLProtocol registerClass:[MyURLProtocol class]];
    return YES;
}
```

## 修改Host
替换URL中的host，并且设置header

* 在函数canonicalRequestForRequest捕获到URL请求后，mutableCopy一份request。替换URL中的host，设置header
 
```
+ (NSURLRequest *) canonicalRequestForRequest:(NSURLRequest *)request {
    if ([request.URL host].length == 0) {
        return request;
    }
    
    NSMutableURLRequest *Myrequest = [request mutableCopy];
    //设置UA，伪装成是safari打开。 因为我的网站有判断，不允许safari以外的浏览器打开
    [Myrequest setValue:@"Mozilla/5.0 (Macintosh; Intel Mac OS X 10_10_5) AppleWebKit/601.4.4 (KHTML, like Gecko) Version/9.0.3 Safari/601.4.4" forHTTPHeaderField:@"user-agent"];
    NSString *originUrlString = [request.URL absoluteString];
    
    NSArray *allHosts = [hostDict allKeys];
    for (NSString *host in allHosts) {
        NSString *ip = [hostDict objectForKey:host];
        //查找请求链接的host是否在host文件中配置了
        NSRange hostRange = [originUrlString rangeOfString:host];
        //查找请求链接的host（可能是ip）是否在host文件中配置了。 用于给httpHeader加Host字段。有时候整个网页重定向后，请求链接就会变成是ip组成的host，这个时候也要给加Host头
        NSRange ipRange = [originUrlString rangeOfString:ip];
        
        if (hostRange.location != NSNotFound) {
            NSString *urlString = [originUrlString stringByReplacingCharactersInRange:hostRange withString:ip];
            NSURL *url = [NSURL URLWithString:urlString];
            Myrequest.URL = url;
            [Myrequest setValue:host forHTTPHeaderField:@"Host"];
            break;
        }else if (ipRange.location != NSNotFound){
            [Myrequest setValue:host forHTTPHeaderField:@"Host"];
            break;
        }
    }
    
    return Myrequest;
}
```

* 如果有重定向（一般发生在UIWebView上）。也要对重定向的链接做处理

```
- (NSURLRequest *)connection:(NSURLConnection *)connection willSendRequest:(NSURLRequest *)request redirectResponse:(NSURLResponse *)response
{
    NSMutableURLRequest *mutableReqeust = [[self request] mutableCopy];
    [mutableReqeust setValue:@"Mozilla/5.0 (Macintosh; Intel Mac OS X 10_10_5) AppleWebKit/601.4.4 (KHTML, like Gecko) Version/9.0.3 Safari/601.4.4" forHTTPHeaderField:@"user-agent"];
    
    if (response) {
        if ([request.URL host].length == 0) {
            return request;
        }
        
        NSString *originUrlString = [request.URL absoluteString];
        NSString *originHostString = @"";
        NSArray *hosts = [hostDict allKeys];
        NSRange hostRange = NSMakeRange(NSNotFound, 0);
        for (int i = 0 ; i < hosts.count ; i++ ) {
            originHostString = hosts[i];
            hostRange = [originUrlString rangeOfString:originHostString];
            if (hostRange.location != NSNotFound) {
                break;
            }
        }
        
        NSString *ip = [hostDict objectForKey:originHostString];;
        // 替换域名
        NSString *urlString = originUrlString;
        if (hostRange.location != NSNotFound) {
            urlString = [originUrlString stringByReplacingCharactersInRange:hostRange withString:ip];
        }
        NSURL *url = [NSURL URLWithString:urlString];
        mutableReqeust.URL = url;
        if (hostRange.location != NSNotFound) {
            [mutableReqeust setValue:originHostString forHTTPHeaderField:@"Host"];
        }
        
        
        [self.client URLProtocol:self wasRedirectedToRequest:mutableReqeust redirectResponse:response];
        return mutableReqeust;
    }else{
        return mutableReqeust;
    }

}
```

## 另外
如果你使用这个例子，不能很好的运行你需要配Host的网站。可以打印每个请求的header跟url调试。看看哪里不对，然后做相应的处理

`绕过https检查`

有时候你的网站可能用了不被信任的证书，给你的URLConnectionDelegate 增加下面两个方法。绕过HTTPS检查

```
//-----https
- (BOOL)connectionShouldUseCredentialStorage:(NSURLConnection *)connection
{
    return YES;
}

- (void)connection:(NSURLConnection *)connection willSendRequestForAuthenticationChallenge:(NSURLAuthenticationChallenge *)challenge
{
    if ([challenge previousFailureCount]== 0) {
        //NSURLCredential 这个类是表示身份验证凭据不可变对象。凭证的实际类型声明的类的构造函数来确定。
        NSURLCredential* cre = [NSURLCredential credentialForTrust:challenge.protectionSpace.serverTrust];
        [challenge.sender useCredential:cre forAuthenticationChallenge:challenge];
        [challenge.sender continueWithoutCredentialForAuthenticationChallenge:challenge];
    }
    else
    {
        [challenge.sender cancelAuthenticationChallenge:challenge];
    }
    
}
```

[例子看这里](https://github.com/09jianfeng/HostWebView)

