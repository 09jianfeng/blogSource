layout: appstore
title: SearchADS
date: 2016-10-14 15:06:17
tags: [iOS,搜索广告]
---

[苹果搜索广告投放后台](https://searchads.apple.com)

## 一些首字母缩写（acronyms）定义

* Impressions

展示数，广告出现在AppStore搜索结果首页的次数

* Taps

点击数，广告被用户点击的次数

* Conversions

转化数，用户点击了广告后的下载数。包括第一次下载跟重复下载

* TTR

点击率， 点击数除于展示数

* CR

转化率， 下载数除于点击数

* Average CPA

总花费除于下载数

* Spend

用于投放广告消耗掉的钱

## 归因API
有了归因API，可以很方便的跟踪推广的广告的用户效果，可以看到不同用户群和不同的关键词带来的效果，可以通过这些数据来优化你的CPT，CPA。 只需要在App中嵌入一小段代码即可以。

### 嵌入代码

1. 添加 iAD.framework
2. `#import <iAd/iAd.h>`
3. 检查系统版本，然后嵌入代码

```
// Check for iOS 10 attribution implementation 
if ([[ADClient sharedClient] respondsToSelector:@selector(requestAttributionDetailsWithBlock:)]) { 
	NSLog(@”iOS 10 call exists”); 
	[[ADClient sharedClient] requestAttributionDetailsWithBlock:^(NSDictionary *attributionDetails, NSError *error) { 
		// Look inside of the returned dictionary for all attribution details 
		NSLog(@”Attribution Dictionary: %@”, attributionDetails); 
	}];
}
```

打印结果

```
{
"Version3.1" = { 
"iad-attribution" = true;
"iad-campaign-id" = 15292426; 
"iad-campaign-name" = “Light Bright Launch"; 
“iad-conversion-date" = “2016-06-14T17:18:07Z";
"iad-click-date" = “2016-06-14T17:17:00Z";
"iad-group-id" = 15307675; 
"iad-adgroup-name" = "LightRight Launch Group"; 
“iad-keyword” = “light right”; 
}; 
}
```





