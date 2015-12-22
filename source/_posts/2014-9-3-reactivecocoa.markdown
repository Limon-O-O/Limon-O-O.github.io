---
layout: post
title: "ReactiveCocoa"
date: 2014-9-3 17:44:25 +0800
comments: true
categories: ReactiveCocoa
---

#ReactiveCocoa

##RACSignal

### 订阅RACSignal

1.RACSignal (Subscription) 
RACSignal (Subscription)类别可以看到所有的订阅事件的方法，每个方法都会将类型为(void (^)(id x))的block作为参数，当事件发生时block中的代码会执行，例如-subscribeNext:方法会传入一个block作为参数，当Signal的next事件发出后，block会接收到事件并执行。

2.UIKit Category 
RAC为UIKit添加了很多类别来让我们可以订阅UI组件的事件，比如UITextField (RACSignalSupport)中的rac_textSignal会在文本域内容变化时发出next事件。 
事件包含的内容可以是类型，只要是对象就行，如果是一些数字，布尔值等字面量，可以用@()语法装箱成NSNumber。

### 操纵Signal 

#####1.RACStream (Operations) 
`-filter`:uses a block to test each value. Returns a new stream with only those values that passed.

	- (instancetype)filter:(BOOL (^)(id value))block;
		RACSequence *numbers = [@"1 2 3 4 5 6 7 8 9" componentsSeparatedByString:@" 	"].rac_sequence;

	// Contains: 2 4 6 8
	RACSequence *filtered = [numbers filter:^ BOOL (NSString *value) {
    	return (value.intValue % 2) == 0;
	}];
`-map`:Maps block across in the receiver , and transform the values. Returns a new stream with the mapped values.

	- (instancetype)map:(id (^)(id value))block;
		RACSequence *letters = [@"A B C D E F G H I" componentsSeparatedByString:@" 	"].rac_sequence;

	// Contains: AA BB CC DD EE FF GG HH II
	RACSequence *mapped = [letters map:^(NSString *value) {
    	return [value stringByAppendingString:value];
	}];
`-flattenMap`:Maps block across the values in the receiver and flattens the result. It is used to signal of signal.简单理解为：取出内部的Signal.

	[[[self.signInButton
 		rac_signalForControlEvents:UIControlEventTouchUpInside]
   			map:^id(id x) {
     			return [self signInSignal]; // 此signal内含一个BOOL signal
   		}]
   		subscribeNext:^(id x) {
     		NSLog(@"Sign in result: %@", x);
   	}];
打印出来的不是BOOL，`-subscribeNex`t: will execute the block whenever the signal sends a value. 但这value不包括 signal of signals：an outer signal that contains an inner signal.

	2014-01-08 21:00:25.919 RWReactivePlayground[33818:a0b] Sign in result:	
							<RACDynamicSignal: 0xa068a00> name:+createSignal:
```
[[[self.signInButton
   rac_signalForControlEvents:UIControlEventTouchUpInside]
   flattenMap:^id(id x) { // subscribe to the inner signal within the outer signal’s
     return [self signInSignal];
   }]
   // Output: Sign in result: 0
   subscribeNext:^(id x) {
     NSLog(@"Sign in result: %@", x);
   }];
```
####2.RACSignal (Operations)


## RACObserve

	// When self.username changes, logs the new name to the console.
	//
	// RACObserve(self, username) creates a new RACSignal that sends the current
	// value of self.username, then the new value whenever it changes.
	// -subscribeNext: will execute the block whenever the signal sends a value.
	[RACObserve(self, username) subscribeNext:^(NSString *newName) {
  		NSLog(@"%@", newName);
	}];
####过滤signal
	// Only logs names that starts with "j".
	//
	// -filter returns a new RACSignal that only sends a new value when its block
	// returns YES.
	[[RACObserve(self, username)
  		filter:^(NSString *newName) {
      		return [newName hasPrefix:@"j"];
  		}]
  		subscribeNext:^(NSString *newName) {
      		NSLog(@"%@", newName);
  	}];
  	


## Chip
`distinctUntilChanged`：Returns a stream of values for which -isEqual: returns NO when compared to the previous value.
用来确保signal只会发送不同的值,比较数值流中当前值和上一个值，如果不同，就返回当前值，简单理解,它将这一次的值与上一次做比较，当相同时（也包括- isEqual:）被忽略掉。

	RACSignal *validSearchSignal =
    [[RACObserve(self, searchText)
      map:^id(NSString *text) {
         return @(text.length > 3);
      }]
      distinctUntilChanged]; // ensure this signal only emits values when the state changes.
  
  	[validSearchSignal subscribeNext:^(id x) {
   		NSLog(@"search text is valid %@", x);
  	}];
  
  	self.executeSearch =
    [[RACCommand alloc] initWithEnabled:validSearchSignal // validSearchSignal决定button能不能点击
      signalBlock:^RACSignal *(id input) {
        return  [self executeSearchSignal];
      }];
      
例子二：比如UI上一个Label绑定了一个值，根据值更新显示的内容:

	RAC(self.label, text) = [RACObserve(self.user, username) distinctUntilChanged];
	self.user.username = @"sunnyxx"; // 1st
	self.user.username = @"sunnyxx"; // 2nd
	self.user.username = @"sunnyxx"; // 3rd 
所以，对于相同值可以忽略的情况，果断加上它吧。


`-takeUntilBlock`：对于每个next值，运行block，当block返回YES时停止取值，如：
	
	[[self.inputTextField.rac_textSignal takeUntilBlock:^BOOL(NSString *value) {
    	return [value isEqualToString:@"stop"];
	}] subscribeNext:^(NSString *value) {
    	NSLog(@"current value is not `stop`: %@", value);
	}];

>Note：停止取值的意思是，输入stop之后，无论输入什么都不会再取值，即输入stop之后，不会再有任何输出。
还有一个例子：

	- (RACSignal*) rac_RequestStateSignal
	{
    	return [[RACObserve(self, state)
        	takeUntilBlock:^ BOOL (NSNumber *state){
            	return [state intValue] == iRequestStateComplete;
        	}]
        	flattenMap:^(NSNumber *state){
            	if ([state intValue] == iRequestStateErrored)
            	{ 
                	// Create a meaningful NSError here if you can.
                	return [RACSignal error:nil];
            	}
            	else
            	{ 
                	return [RACSignal return:state];
            	}
        	}];
	}
	
`-switchToLatest `方法用于signal-of-signals，它总是输出最新的信号的值。

	RACSubject *letters = [RACSubject subject];
	RACSubject *numbers = [RACSubject subject];
	RACSubject *signalOfSignals = [RACSubject subject];

	RACSignal *switched = [signalOfSignals switchToLatest];

	// Outputs: A B 1 D
	[switched subscribeNext:^(NSString *x) {
    	NSLog(@"%@", x);
	}];

	[signalOfSignals sendNext:letters]; // 打印letters信号内的值
	[letters sendNext:@"A"];
	[letters sendNext:@"B"];

	[signalOfSignals sendNext:numbers]; // 打印numbers信号内的值
	[letters sendNext:@"C"];
	[numbers sendNext:@"1"];

	[signalOfSignals sendNext:letters]; // 打印letters信号内的值
	[numbers sendNext:@"2"];
	[letters sendNext:@"D"];
	

`-rac_signalForSelector:fromProtocol:`

这个方法主要是把 protocal 转为一个 Signal 便于使用。值得注意的是这个函数返回的是一个 RACTuple。 这个 RACTuple 包含了 Selector 方法里面所有的参数

`-rac_liftSelector:withSignalsFromArray:`
这个方法它的意思是当传入的 Signals 都至少sendNext过一次，接下来只要其中任意一个signal有了新的内容。就会去触发第一个 selector 参数的方法。
> [ReactiveCocoa 用 RACSignal 替代 Delegate](http://iiiyu.com/2014/12/26/learning-ios-notes-thirty-six/)

