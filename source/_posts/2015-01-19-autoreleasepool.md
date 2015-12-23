---
layout: post
title: "内存管理 - Autorelease"
date: 2015-01-19 16:12:34 +0800
comments: false
categories: autorelease 
---

Autorelease实际上只是把对release的调用延迟了，对于每一个Autorelease，系统只是把该Object放入了当前的Autorelease pool中，当该pool被释放时，该pool中的所有Object会被调用release，也就是计数器会减1，但是自动释放池被销毁了，里面的对象并不一定会被销毁。

简而言之，autorelease pool 避免了频繁申请/释放内存。

##pool什么时候被销毁？
在每一个Runloop结束时，当前栈顶的Autorelease pool会被销毁，此pool内的OC对象会被release。
那什么是一个Runloop呢？ 一个UI事件，Timer call， delegate call， 都会是一个新的Runloop


在Iphone项目中，大家会看到一个默认的Autorelease pool，程序开始时创建，程序退出时销毁，按照对Autorelease的理解，岂不是所有autorelease pool里的对象在程序退出时才release， 这样跟内存泄露有什么区别？

答案是，对于每一个Runloop， 系统会隐式创建一个Autorelease pool，这样所有的release pool会构成一个栈式结构，遵循先进后出，在每一个Runloop结束时，当前栈顶的Autorelease pool会被销毁，这样这个pool里的每个OC对象会被release。


##pool嵌套，栈式结构

对于每一个Runloop， 系统会隐式创建一个Autorelease pool，这样所有的release pool会构成一个栈式结构，遵循先进后出

```
#import <Foundation/Foundation.h> 
#import "Person.h" 
int main(int argc, const char * argv[]) 
{ 
     
    // 自动释放池1 
    @autoreleasepool { 
         
 // 对象的释放交给 自动释放池去管理 不用再写[person release] 
        Person *person = [[Person alloc] init];  
         
        // 再创建一个自动释放池2 
        @autoreleasepool { 
             
            Person *person2 = [[Person alloc] init]; 
        } 
        
        Person *person3 = [[Person alloc] init];    
    } 
    return 0; 
}
```
pool2先被销毁，最后才是pool1，即释放顺序：person2 -> person3/person2

####

####一下内容整理于 #[黑幕背后的Autorelease](http://blog.sunnyxx.com/2014/10/15/behind-autorelease/)

在ARC下，`@autoreleasepool{}`创建一个AutoreleasePool，随后编译器将其改写成下面的样子：

	void *context = objc_autoreleasePoolPush();
	// {}中的代码
	objc_autoreleasePoolPop(context);
	
而这两个函数都是对AutoreleasePoolPage的简单封装，所以自动释放机制的核心就在于这个类。



	class AutoreleasePoolPage 
	{
		.....
    	magic_t const magic;
    	id *next;
    	pthread_t const thread;
    	AutoreleasePoolPage * const parent;
    	AutoreleasePoolPage *child;
    	uint32_t const depth;
    	uint32_t hiwat;
		....
	}

* AutoreleasePool并没有单独的结构，而是由若干个AutoreleasePoolPage以双向链表的形式组合而成（分别对应结构中的parent指针和child指针）
* AutoreleasePool是按线程一一对应的（结构中的thread指针指向当前线程）
* AutoreleasePoolPage每个对象会开辟4096字节内存（也就是虚拟内存一页的大小），除了上面的实例变量所占空间，剩下的空间全部用来储存autorelease对象的地址
* 上面的id *next指针作为游标指向栈顶最新add进来的autorelease对象的下一个位置
* 一个AutoreleasePoolPage的空间被占满时，会新建一个AutoreleasePoolPage对象，连接链表，后来的autorelease对象在新的page加入


##总结

* autorelease方法不会改变对象的引用计数器，只是将这个对象放到自动释放池中；
* 自动释放池实质是当自动释放池销毁后调用对象的release方法，不一定就能销毁对象（例如如果一个对象的引用计数器>1则此时就无法销毁）；
* 由于自动释放池最后统一销毁对象，因此如果一个操作比较占用内存（对象比较多或者对象占用资源比较多），最好不要放到自动释放池或者考虑放到多个自动释放池；
* ObjC中类库中的静态方法一般都不需要手动释放，内部已经调用了autorelease方法；

[黑幕背后的Autorelease](http://blog.sunnyxx.com/2014/10/15/behind-autorelease/)

[Objective-C之内存管理](http://www.cnblogs.com/kenshincui/p/3870325.html#autoreleasepool)

