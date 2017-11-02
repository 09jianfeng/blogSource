---
title: iphone摄像头采集
date: 2016-01-11 09:59:31
categories: [iOS,图像]
tags: [iOS,图像]
---

## 代码 tips

## 摄像头的一些关键词

### 曝光

### 白平衡

### 测光

### 调焦

### 防抖

### 采集帧率设置

先判断摄像头支持的帧率

```
- (BOOL)supportsVideoFrameRate:(NSInteger)videoFrameRate
{
    if (!_device) {
        return NO;
    }
    
    AVCaptureDeviceFormat* format = [_device activeFormat];
    NSArray *videoSupportedFrameRateRanges = [format videoSupportedFrameRateRanges];
    for (AVFrameRateRange *frameRateRange in videoSupportedFrameRateRanges) {
        if ( (frameRateRange.minFrameRate <= videoFrameRate) && (videoFrameRate <= frameRateRange.maxFrameRate) ) {
            return YES;
        }
    }
    
    return NO;
}
```

然后设置最大、最小的帧间间隔

```

- (void)doSetFrameRate:(int32_t)frameRate
{
    if (![self supportsVideoFrameRate:frameRate]) {
        //   NSLog(@"CameraCaptureFilter not support framerate %d", newFrameRate);
        return;
    }
    AVCaptureDevice *videoDevice = _device;
    
    if ([videoDevice lockForConfiguration:NULL]) {
        [videoDevice setActiveVideoMinFrameDuration:CMTimeMake(1, frameRate)];
        [videoDevice setActiveVideoMaxFrameDuration:CMTimeMake(1, frameRate)];
        
        [videoDevice unlockForConfiguration];
        _frameRate = frameRate;
        _cameraJustChanged = YES;
    }
}

```
