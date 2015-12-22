---
layout: post
title: "内存管理 - 引用计数"
date: 2015-01-13 14:27:14 +0800
comments: false
categories: 内存管理
---


##内存管理概述
在Objective-C中，系统并不会自动释放堆中的内存，其他高级语言C#、JAVA都有垃圾回收机制，但在Objective-C中并没有垃圾回收机制，那么Objective-C中的内存是如何管理？
Objective-C语言使用引用计数器来管理内存，每个对象都有个可以递增或递减的计数器。如果想使某个对象继续存活，那就递增其引用计数器；用完之后，就递减其计数器；当计数器为0，就表示没人关注此对象，此对象就会被销毁。




##引用计数工作原理
在引用计数架构下，对象有个计数器，用以表示当前有多少个事物想令此对象继续存活。
NSObject协议声明了三个方法用于操作计数器

* Retain 递增计数器
* Release 递减计数器
* autorelease 待销毁autorelease pool时，再递减pool内的全部计数器

1.对一个对象发送alloc、retain、new、copy消息，计数器 +1

2.对一个对象发送release消息，计数器 -1

3.当一个对象的引用计数器为0，对象就被回收(dealloced)，也就是说，系统会将对象其占用的内存标记为“可重用”(reuse)，放到“可用内存池”(avaiable pool)

###手动管理内存的三点原则

* 如果需要持有一个对象，那么对其发送retain
* 如果之后不再使用该对象，那么需要对其发送release（或者autorealse）
* 每一次对retain,alloc或者new的调用，需要对应一次release或autorealse调用

###图表演示

下图演示了一个对象自创建出来之后经历一次retain及两次release操作的过程

![Memory](http://limons-gitimage.stor.sinaapp.com/Memory.png)


下图所示的对象图中，ObjectB和ObjectC都引用了ObjectA，若B和C都不再使用A，则其计数器为0，便被回收。还有OrderObject想令B和C继续存活，而应用程序里又有另外的对象想令OrderObject继续存活，如果按“引用树“回溯，那么会发现一个“根对象”，在iOS中，则是UIApplication对象。此”根对象”是应用程序启动时创建的单例。

![Memory](http://limons-gitimage.stor.sinaapp.com/Memory-2.png)

###代码演示
	- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
		NSMutableArray *array = [[NSMutableArray alloc] init];
		NSNumber *number = [[NSNumber alloc] initWithInt:1337]; 
	
		[array addObject:number]; //number的引用计数器+1（此时number的引用计数器至少为2）
		[number release]; // -1

		// do something with `array’
	
		[array release];
		return YES;
	}
>在调用数组的`-addObject`方法时，数组也会在number上调用retain，以期令此对象存活

##引用计数的应用场景
在上面的代码演示中，还不看出引用计数的真正用处。在函数内使用一个临时的对象，通常不需要修改引用计数，只需在返回前将该对象销毁即可。
引用计数真正的应用场景是，用于对象之间传递和共享数据。

##引用计数的注意要点

###retainCount可能永远不为0
可能有人会测试对象释放时，看retainCount是否为0，代码如下：

	NSObject *object = [[NSObject alloc] init];
    NSLog(@"%lu",(unsigned long)[object retainCount]);
    [object release];
    NSLog(@"%lu",(unsigned long)[object retainCount]);

但是，打印的结果是 1  1

最后一次输出，引用计数并没有变成0，原因是在最后一次 release 时，系统马上就回收了内存，就没有再将retainCount减1，因为不将值从1变成0，可以减少一次内存操作，加快对象的回收，只有在系统不打算这么优化时，计数值才会递减为0

###悬挂指针

当所指向的对象被释放或者收回，此情况下该指针便称悬垂指针（也叫迷途指针）。

> 某些编程语言允许未初始化的指针的存在，而这类指针即为野指针。

为什么对象被回收了，向其发送消息不会崩？代码如下：

    NSNumber *number = [[NSNumber alloc] initWithInt:1337];

    [number release];
    
    NSLog(@"number = %@",number);

内存已经被回收，如果向其发送消息，可能使程序崩溃。为什么说“可能”，而没说“一定”，是因为对象的所占的内存在“解除分配”(deallocated)之后，只是放回了“可用内存池”(avaiable pool)。如果在执行NSLog时尚未覆写对象内存，那么该对象仍然有效。
>由此可见，因过早释放对象而导致的bug很难调试

为了防止此情况的发送，一般release之后都会清空指针

	NSNumber *number = [[NSNumber alloc] initWithInt:1337];

    [number release];
    number = nil; 
   
>在Objective-C中，向nil发送消息不会出错。

##总结
* Objective-C通过`引用计数机制`来管理内存
* 对象创建之后，其引用计数器至少为1，`retain`和`release`分别会递增及递减计数器
