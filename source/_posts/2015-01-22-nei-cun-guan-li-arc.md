---
layout: post
title: "内存管理 - ARC"
date: 2015-01-22 18:20:16 +0800
comments: false
categories: 内存管理
---

##什么是ARC
Automatic Reference Counting，自动引用计数，即ARC。

在ARC下，不需用retain,release,autorelease,dealloc来管理内存，它提供了自动评估内存生存期的功能，并且在编译期间自动加入合适的管理内存的方法。编译器也会自动生成dealloc函数。

###ARC工作原理
ARC并不是一项运行时的服务，实际上它是由Clang front-end提供的两段过程。下图演示了这两段过程。在front-end段时，Clang检查每个预处理文件的对象和属性。然后它跟据一些固定的规则将retain，release和autorelease语句加入其中。

> 实际上，ARC在调用retain,release,autorelease,dealloc方法时，并不通过Objective-C消息派送机制，而是直接调用C。比如，ARC会调用与retain等价的objc_retain

![](http://limons-gitimage.stor.sinaapp.com/CFigure4.gif)

举例来说：

* 如果对象被分配内存并处于一个方法当中，它会在这个方法的结尾处获得一个release语句
* 如果是一个类属性，它的release语句会加入到类的dealloc方法中
* 如果这个对象是用来返回的或者它是一个容器对象，它会加入一个autorelease语句
* 如果这个对象是弱引用，把它放在一边不管它。

>译于：[Automatic Reference Counting on iOS](http://www.drdobbs.com/mobile/automatic-reference-counting-on-ios/240000820)

###代码演示

####MRC下
```
@class Bar;
@interface Foo
{
@private
    NSString *myStr;
}
@property (readonly) NSString *myStr;
  
- (Bar *)foo2Bar:(NSString *)aStr;
- (Bar *)makeBar;
//...
@end
  
  
@implementation Foo;
@dynamic myStr;
  
– (Bar *)foo2Bar:(NSString *)aStr
{
    Bar *tBar;
      
    if (![self.myStr isEqualToString:aStr])
    {
        [aStr retain];
        [myStr release];
        myStr = aStr;
    }  
    return ([self makeBar]);
}
  
- (Bar *)makeBar
{
    Bar *tBar
    //...
    //... conversion code goes here
    //...
    [tBar autorelease];
    return (tBar);
}
//...
  
- (void)dealloc
{
    [myStr release];
    [super dealloc];
}
@end
```

####ARC下
```
@class Bar;
@interface Foo
{
@private
    NSString *myStr;
}
@property(readonly) NSString *myStr;
  
- (Bar *)foo2Bar:(NSString *)aStr;
- (Bar *)makeBar;
//...
@end
  
  
@implementation Foo;
@dynamic myStr;
  
– (Bar *)foo2Bar:(NSString *)aStr
{
    Bar *tBar;
      
    if (![self.myStr isEqualToString:aStr])
    {
        myStr = aStr;
    }  
    return ([self makeBar]);
}
  
- (Bar *)makeBar
{
    Bar *tBar
    //...
    //... conversion code goes here
    //...
    return (tBar);
}
//...
@end
```
> 以上代码来源于：[Automatic Reference Counting on iOS](http://www.drdobbs.com/mobile/automatic-reference-counting-on-ios/240000820)

##ARC下内存管理 --- 内存管理语义

ARC帮助我们解决90%内存管理为题，剩余的10%包括：

* 属性(property)的内存管理语义是用strong、还是weak？
* 循环引用（retain cycle）问题
* CoreFoundation对象不归ARC管理

###属性
属性(property)是Objective-C的一项特性，用于封装对象中的数据。Objective-C对象通常会把其所需要的数据保存为各种实例变量。实例变量一般通过getter、setter来访问。

`@property(nonatomic, strong) NSArray *emailList;`
 
此`@property`会自动生成三个东西：_emailList实例变量、getter、setter

属性用于封装数据，而数据则要有“具体的所有权语义”(concrete ownership semantic)，也称为内存管理语义。

回归本题，在ARC下，我们通常用**内存管理语义**来管理内存，下面这一组语义仅会影响setter。例如，用setter设定一个新值时，它是应该“保留”(retain)此值呢，还是只将其赋给底层实例变量就好？这些取决于你是用strong，还是copy，还是assign等


* `assign`  对“纯量类型”(scalar type)的简单赋值

		-(void)setState:(int)State{  
    		_state = state;  
		}

* `unsafe_unretained`  此语义与`assign`相同，但是它适用于"对象类型"，该语义表达一种"非拥有关系"(unretained),当对象被销毁时，属性值不会自动清空("不安全",unsafe)，这一点与`weak`有区别

* `weak`  该语义表达一种"非拥有关系"，为这种属性设置新值时，设置方法既不保留新值，也不释放旧值，即仅作简单赋值，类似`assign`，当对象销毁，属性值被赋值为`nil`

* `strong`  该语义表达一种"拥有关系"，为这种属性值设置新值时，设置方法会先保留新值，再释放旧值，最后把新值设置上去


		-(void)setName:(NSArray *)emailList{	
			if ( emailList != _emailList){
				[emailList retain]; // 保留新值
				[_emailList release]; // 释放旧值
				_emailList = emailList;
			}   
		}


此setter将保留新值并释放旧值，然后更新实例变量，令其指向新值。顺序很重要，假如两个值都指向对象A，还未保留新值就先把旧值释放了，那么先执行的release操作就可能导致对象A被回收，而后续的retain操作并无法令对象A复生，于是实例变量就成了悬挂指针

> 当所指向的对象被释放或者收回，此情况下该指针便称悬垂指针（也叫迷途指针）。

* copy  此语义与strong类似，不同在于setter并不保留新值，而是将其`copy`，如下：
	
	
		@property (nonatomic, copy) NSString *name;
	
		-(void)setName:(NSString*)name{  
     		if ( name != _name){  
          		[_name release];
            	_name = nil;
          		_name = [name copy]; 
     		}  
		}

使用copy可以用来保护数据的封装性，比方说，传递过来的新值是一个NSMutableString的实例，此时若不`copy`，当设置完属性值之后，该可变新值在对象不知情的情况下遭人更改。所以，这时就要`copy`一份“不可变”(immutable)的字符串，确保对象中的字符串值不会不会无意间变动。


###ARC下内存管理 --- 基本规则

ARC下内存管理的一个基本规则：只要某个对象被任一strong指针指向，那么它将不会被回收。

####strong

在MRC下，如果想令一个对象继续存活，需用retain来递增计数器，不过ARC已经在编译时帮我们加入了retain、release等，现在唯一需要的是，用一个强指针指向该对象，只要指针没有被空置，该对象会一直在堆上。

> 实际上，不论MRC和ARC下，还是在用引用计数来管理内存，具体请看：[内存管理 - 引用计数器](http://limon-.github.io/blog/2015/01/13/nei-cun-guan-li-yin-yong-ji-shu-qi/)

在默认情况下，所有的实例变量和局部变量都是strong类型的。可以说strong类型的指针在行为上和MRC时代的retain是比较相似的。


####weak

weak常用在两个对象间存在包含关系时：对象A有一个strong指针指向对象B，并持有它，而对象B中也有一个weak指针指回对象B，从而避免了循环持有。

一个常见的例子就是delegate设计模式，viewController中有一个strong指针指向它所负责管理的UITableView，而UITableView中的dataSource和delegate指针都是指向viewController的weak指针。可以说，weak指针的行为和MRC时代的assign有一些相似点，但是考虑到weak指针更聪明些（会自动指向nil）

![](http://limons-gitimage.stor.sinaapp.com/arcpic7.png)

>代理模式可以简单认为，两个对象之间形成"半保留环"（A对象强指针指向B对象，B对象弱指针指向A对象）


###ARC下内存管理 --- 循环引用

接下来简单讲讲容易循环引用的场景，详情请看[官方文档](https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/ProgrammingWithObjectiveC/EncapsulatingData/EncapsulatingData.html#//apple_ref/doc/uid/TP40011210-CH5-SW3)的相关资料

####委托模式
	
	@property (nonatomic,weak) id<ComposeToolbarButtonTypeDelegate> delegate; 

用`weak`形成“半保留环”，防止循环引用


####block
这个网上大把，可以看看[block使用小结、在arc中使用block、如何防止循环引用](http://www.cnbluebox.com/?p=255)


###ARC下内存管理 --- self

ARC下，方法内的self既不是strong也不是weak，而是unsafe_unretained的（init系列方法的self除外）。
换而言之，在普通方法中，ARC不会对`self`做`retain`和`release`操作，生命周期全由调用方来决定，如果调用方没有保证`self`在被调用方法中的生命周期，可能在此方法中运行到一半，`self`就被释放了，可能程序就崩溃了。




##总结
* ARC管理对象生命期的基本办法：在编译期间，在合适的地方插入`retain`和`release`操作
* 在ARC下，我们通常通过`内存管理语义`来管理内存，常见的有：`strong`、`weak`、`copy`


##番外
研读了@sunny大大的[《ARC对self的内存管理》](http://blog.sunnyxx.com/2015/01/17/self-in-arc/)之后作一些笔记，


##参考资料

[Objective-C Automatic Reference Counting (ARC)](http://clang.llvm.org/docs/AutomaticReferenceCounting.html) ----- 关于ARC的详细文档

[ARC对self的内存管理](http://blog.sunnyxx.com/2015/01/17/self-in-arc/) ----- 专门对ARC下self的剖析
