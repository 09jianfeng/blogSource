---
title: NSObject与id的区别
date: 2017-03-07 15:22:09
tags: [iOS,Objective-C]
---

id is a special keyword used in Objective-C to mean “some kind of object.” It does not contain isa pointer(isa, gives the object access to its class and, through the class, to all the classes it inherits from), So you lose compile-time information about the object.

```
NSString* aString = @"Hello";
id anObj = aString;  
```

NSObject contain isa pointer. 

```
NSString* aString = @"Hello";
NSObject* anObj = aString;
```

Consider the following code:

```
id someObject = @"Hello, World!";
[someObject removeAllObjects];

```

In this case, someObject will point to an NSString instance, but the compiler knows nothing about that instance beyond the fact that it’s some kind of object. The removeAllObjects message is defined by some Cocoa or Cocoa Touch objects (such as NSMutableArray) so the compiler doesn’t complain, even though this code would generate an exception at runtime because an NSString object can’t respond to removeAllObjects.

Rewriting the code to use a static type:

```
NSString *someObject = @"Hello, World!";
[someObject removeAllObjects];
```

means that the compiler will now generate an error because removeAllObjects is not declared in any public NSString interface that it knows about.