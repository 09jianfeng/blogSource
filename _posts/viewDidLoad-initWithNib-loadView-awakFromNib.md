---
title: 'viewDidLoad,initWithNib,loadView,awakeFromNib'
date: 2014-08-10 15:21:08
categories: [iOS,UIKit]
tags: [iOS,UIKit]
---

## loadView

无论XIB还是代码创建都会调用loadView方法。self.view为nil时才会被调用。
手工创建视图时，loadView被调用时self.view还为nil。一般在该方法中手工定制view。
XIB创建视图时，loadView仍会被调用、loadView被调用时XIB定制的视图还没创建完成，若是再覆写该方法的话、会将XIB定制的视图覆盖掉。
所以，纯手工定制视图时，一般在该方法中写；XIB定制视图时、不要覆写该方法。 

PS：一般情况下都不会去调用这个方法

## viewDidLoad

无论XIB还是代码创建都会调用viewDidLoad方法。
手工创建视图时，viewDidLoad被调用时self.view已经创建完成。可在在该方法中进一步定制视图。
XIB创建视图时，viewDidLoad仍会被调用，viewDidLoad被调用时self.view已经创建完成。可在在该方法中进一步定制视图。
所以，无论那种方式定制视图、都可以覆写该方法。

## initWithNibName

一般情况下调用 init方法或者调用initWithNibName方法实例化UIViewController, 不管调用哪个方法最终都会调用initWithNibName方法。
        当控制器被initWithNibName:并加入到导航控制器的栈中时，它不会加载nib文件，直到nib文件被实际显示。因此控制器在nib文件中定 义的内容，例如label，可能还没有实例化。此时label可能只是一个nil指针，需要额外使用代码中实现的属性来存储信息。可以在 viewWillAppear：方法中对niv实例化的对象属性进行设置。

## initWithCoder

initWithCoder是一个类在IB中创建,在IB中被调用跳转.那么这个controller的initWithCoder会被调用。

## awakeFromNib
在使用IB的时候才会涉及到此方法的使用，当.nib文件被加载的时候，会发送一个awakeFromNib的消息到.nib文件中的每个对象，每个对象都可以定义自己的awakeFromNib函数来响应这个消息，执行一些必要的操作。

调用顺序: super init->initWithNibName(其实在super init中) ->int after self created->loadView->ViewDidLoad

## 区别

（1）awakeFromNib和initWithCoder:差别
awakeFromNib 从xib或者storyboard加载完毕就会调用
initWithCoder: 只要对象是从文件解析来的，就会调用
同时存在会先调用initWithCoder:

（2）initWithCoder: & initWithFrame:
initWithCoder：使用文件加载的对象调用（如从xib或stroyboard中创建）
initWithFrame：使用代码加载的对象调用（使用纯代码创建）

注意：所以为了同时兼顾从文件和从代码解析的对象初始化，要同时在initWithCoder: 和 initWithFrame: 中进行初始化
