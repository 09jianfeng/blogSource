---
title: dyld_decache 导出系统库的可执行文件
date: 2015-04-5 17:29:51
categories: [iOS,逆向]
tags: [iOS,逆向]
---

## 原理
[Howettblog 这里有介绍](http://blog.howett.net/2009/09/cache-or-check/)

## iOS8

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


## iOS9

先用homebrew安装wget

```
brew install wget
```

安装好 wget后，执行下面的命令：

```
cd ~
mkdir dsc_extractor
cd dsc_extractor
wget http://opensource.apple.com/tarballs/dyld/dyld-210.2.3.tar.gz
tar xvf dyld-210.2.3.tar.gz
```

打补丁

```
cd dyld-210.2.3/launch-cache/
touch dsc_extractor.patch
```
然后复制下面的代码到 dsc_extractor.patch

```
--- dyld-210.2.3/launch-cache/dsc_extractor.cpp  2012-05-21 02:35:15.000000000 -0400
+++ dyld-210.2.3/launch-cache/dsc_extractor.cpp	2013-07-26 16:05:03.000000000 -0400
@@ -37,6 +37,7 @@
 #include <mach-o/arch.h>
 #include <mach-o/loader.h>
 #include <Availability.h>
+#include <dlfcn.h>
 
 #define NO_ULEB 
 #include "Architectures.hpp"
@@ -456,7 +457,7 @@
 }
 
 
-#if 0 
+/* #if 0 */
 
 typedef int (*extractor_proc)(const char* shared_cache_file_path, const char* extraction_root_path,
 													void (^progress)(unsigned current, unsigned total));
@@ -468,7 +469,7 @@
 		return 1;
 	}
 	
-	void* handle = dlopen("/Developer/Platforms/iPhoneOS.platform/usr/lib/dsc_extractor.bundle", RTLD_LAZY);
+	void* handle = dlopen("/Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/usr/lib/dsc_extractor.bundle", RTLD_LAZY);
 	if ( handle == NULL ) {
 		fprintf(stderr, "dsc_extractor.bundle could not be loaded\n");
 		return 1;
@@ -484,7 +485,7 @@
 	fprintf(stderr, "dyld_shared_cache_extract_dylibs_progress() => %d\n", result);
 	return 0;
 }
-#endif
+/* #endif */
```

继续：

```
patch < dsc_extractor.patch
```
如果报错：[下载这个文件](http://7xibfi.com1.z0.glb.clouddn.com/uploads/default/original/2X/e/e4ce267c5583ef72198d2d59df2dcb2f2f62bd2e.patch)，替换文件名为dsc_extractor.patch


编译：

```
clang++ -o dsc_extractor dsc_extractor.cpp dsc_iterator.cpp
```
编译完后会生成一个 `dsc_extractor` 可执行文件，在目录`~/dsc_extractor/dyld-210.2.3/launch-cache`下。

使用：

```
/path/to/dsc_extractor /path/to/dyld_shared_cache_arm64 /path/to/decached/binaries/
```

Done
