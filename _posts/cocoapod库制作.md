---
title: cocoapod库制作
date: 2017-02-09 10:36:16
tags: [iOS,打包]
---

# 构建共有pod

## 创建pod工程

```
pod lib create HostSetting
```

## 检查设置
```
pod lib lint HostSetting.podspec
```

在HostSetting.podspec 文件中编辑开源库的版本、描述、依赖、工程在github上的链接等。字段都已经创建好了，替换默认的内容就行。

## 开始添加代码
展开Pod工程下面，有个Development Pods -> HostSetting -> HostSetting ->Classes 在这下面添加需要开源的文件。 添加完后 cd example, pod install。
就可以在example中使用了。

## 把工程上传到github
在github上创建仓库，并且把工程上传到github，给做好了的开源库打上tag。打的tag的version应该要和.podspec文件中的s.version字段的版本号对上。

```
> git tag 0.1.0
> git push origin 0.1.0
```

## 验证

```
pod spec lint HostSetting.podspec
```

## 验证通过后就push
在push前，需要注册. 执行完下面的命令后，会发个链接到你的email，登录email验证一下。

```
pod trunk register xxxx@gmail.com 'hostSetting' --description='hostSetting session'
```

然后就可以执行下面的命令，发布你的代码了

```
pod trunk push HostSetting.podspec
```


[参考](http://www.jianshu.com/p/32ba94d41861)