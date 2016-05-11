---
title: dyld_decache 导出系统库的可执行文件
date: 2016-04-5 17:29:51
tags: [iOS,逆向]
---

使用dyld_decache之前，要将“/System/Library/Caches/com.apple.dyld/dyld_shared_cache_armx”用iFunBox（不能用scp）从iOS拷贝到OSX中。然后从https://github.com/downloads/kennytm/Miscellaneous/dyld_decache[v0.1c].bz2下载dyld_decache。解压之后赋予其可执行权限，如下：
```
snakeninnysiMac:~ snakeninny$ chmod +x /path/to/dyld_decache\[v0.1c\]
```
然后开始提取二进制文件，如下：

```
snakeninnysiMac:~ snakeninny$ /path/to/dyld_decache\[v0.1c\] -o /where/to/store/decached/binaries/ /path/to/dyld_shared_cache_armx
  0/877: Dumping '/System/Library/AccessibilityBundles/AXSpeechImplementation.bundle/AXSpeechImplementation'...
  1/877: Dumping '/System/Library/AccessibilityBundles/AccessibilitySettingsLoader.bundle/AccessibilitySettingsLoader'...
  2/877: Dumping '/System/Library/AccessibilityBundles/AccountsUI.axbundle/AccountsUI'...
  
```

提取出的所有二进制文件都存放在了“/where/to/store/decached/binaries/”下。值得一提的是，逆向工程需要分析的二进制文件现在散落在OSX和iOS两个系统中，不方便查找，建议利用scp工具把iOS文件系统拷贝一份存在OSX里。





