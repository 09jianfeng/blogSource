---
title: runtime二、Method Swizzling、面向切面编程AOP
date: 2016-05-31 10:25:07
categories: [iOS,Objective-C,Runtime]
tags: [iOS,Objective-C,Runtime]
---

# Method Swizzling

比如有这样一个需求，需要统计某个按钮的点击事件。利用Method Swizzling可以使得这个统计代码独立，减少代码之间的耦合。

示例如下

```
#import "MethodSwizzlViewController.h"
#import <objc/message.h>

@interface MethodSwizzlViewController ()

@end

@implementation MethodSwizzlViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    // Do any additional setup after loading the view.
}

- (void)didReceiveMemoryWarning {
    [super didReceiveMemoryWarning];
    // Dispose of any resources that can be recreated.
}
- (IBAction)btnClick2:(id)sender {
    UIAlertController *alertController = [UIAlertController alertControllerWithTitle:@"按钮点击事件2" message:@"一般这里做的是的一些跳转操作\n\n想要在这里插入一些统计逻辑" preferredStyle:UIAlertControllerStyleAlert];
    UIAlertAction *defaultAction = [UIAlertAction actionWithTitle:@"OK" style:UIAlertActionStyleDefault handler:^(UIAlertAction * _Nonnull action) {
    }];
    [alertController addAction:defaultAction];
    [self presentViewController:alertController animated:YES completion:nil];
}

- (IBAction)btnClick:(id)sender {
    UIAlertController *alertController = [UIAlertController alertControllerWithTitle:@"按钮点击事件1" message:@"一般这里做的是的一些跳转操作\n\n想要在这里插入一些统计逻辑" preferredStyle:UIAlertControllerStyleAlert];
    UIAlertAction *defaultAction = [UIAlertAction actionWithTitle:@"OK" style:UIAlertActionStyleDefault handler:^(UIAlertAction * _Nonnull action) {
    }];
    [alertController addAction:defaultAction];
    [self presentViewController:alertController animated:YES completion:nil];
}

/*
#pragma mark - Navigation

// In a storyboard-based application, you will often want to do a little preparation before navigation
- (void)prepareForSegue:(UIStoryboardSegue *)segue sender:(id)sender {
    // Get the new view controller using [segue destinationViewController].
    // Pass the selected object to the new view controller.
}
*/

#pragma mark - method swizzling

- (void)swizzled_btnClick:(id)sender
{
    // call original implementation, because of the method_exchangeImplementations()
    [self swizzled_btnClick:sender];
    
    // Logging
    NSLog(@"统计按钮1点击事件");
}

void swizzleMethod(Class class, SEL originalSelector, SEL swizzledSelector)
{
    // the method might not exist in the class, but in its superclass
    Method originalMethod = class_getInstanceMethod(class, originalSelector);
    Method swizzledMethod = class_getInstanceMethod(class, swizzledSelector);
    
    // class_addMethod will fail if original method already exists
    BOOL didAddMethod = class_addMethod(class, originalSelector, method_getImplementation(swizzledMethod), method_getTypeEncoding(swizzledMethod));
    
    // the method doesn’t exist and we just added one
    if (didAddMethod) {
        class_replaceMethod(class, swizzledSelector, method_getImplementation(originalMethod), method_getTypeEncoding(originalMethod));
    }
    else {
        method_exchangeImplementations(originalMethod, swizzledMethod);
    }
}

void (*old_btnClick2)(id self, SEL _cmd, id sender);
void new_btnClick2(id self, SEL _cmd, id sender);
void new_btnClick2(id self, SEL _cmd, id sender){
    old_btnClick2(self, _cmd, sender);
    NSLog(@"统计点击事件2");
}

void swizzleMethod2(Class classOrignal,SEL orignalSelector){
    Method originalMethod = class_getInstanceMethod(classOrignal, orignalSelector);
    old_btnClick2 = (void *)method_getImplementation(originalMethod);
    method_setImplementation(originalMethod, (IMP)new_btnClick2);
}

+(void)load{
    swizzleMethod([self class], @selector(btnClick:), @selector(swizzled_btnClick:));
    swizzleMethod2([self class],@selector(btnClick2:));
}


@end

```
上面用来做Method swizzling的主要是这几个接口

*  获取实例方法的Method的接口是 `Method class_getInstanceMethod(Class cls, SEL name) `对应的获取类方法的Method的接口是 `Method class_getClassMethod(Class cls, SEL name)`
*  给类添加方法的接口 `BOOL class_addMethod(Class cls, SEL name, IMP imp, const char *types) ` 
*  替换方法的函数指针(IMP)的接口 `IMP class_replaceMethod(Class cls, SEL name, IMP imp, const char *types) `
*  交换两个方法的实现的接口 `void method_exchangeImplementations(Method m1, Method m2) `
*  获取一个方法的函数指针 `IMP method_getImplementation(Method m)`
*  给一个函数设置他的函数指针 `IMP method_setImplementation(Method m, IMP imp) `       





               