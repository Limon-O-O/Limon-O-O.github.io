---
layout: post
title: "ReactiveCocoa的三种基本的模式"
date: 2014-6-10 09:35:03 +0800
comments: false
categories: ReactiveCocoa
---

#ReactiveCocoa
---
###三种基本的模式
在ReactiveCocoa中有三种基本的模式：责任链、分割和组合模式（chaining, splitting, and combining）。

####一、 Chaining模式
Chaining，将一个已有的signal转换为一个新的signal。常用的操作是创建一个新的signal，再对它使用filter:、map:或startWith:等方法。

```
RAC(self.textField.text) = [[[RACSignal interval:1] startWith:[NSDate date]] map:^id(NSDate *value) {  
    NSDateComponents *dateComponents = [[NSCalendar currentCalendar] components:NSMinuteCalendarUnit | NSSecondCalendarUnit fromDate:value];  
      
    return [NSString stringWithFormat:@"%d:%02d", dateComponents.minute, dateComponents.second];  
}]; 

```
我们将textFiled的text属性绑定为三个串连的signals的结果。首先，我们创建一个间隔信号，这个信号每隔一秒钟就发送当前时间。间隔信号在没有启动的时候是不会有值的，所以我们使用startWith:启动起来。最后，使用map:将signal的NSDate值转换为一个NSString字符串，这个字符串将会被赋值到textField的text属性上。
![Chaining](http://teehanlax.com.s3.amazonaws.com/wordpress/wp-content/uploads/chaining.png)

 Chaining是最常用的操作，而且它通常不使用局部变量，而是像上面那样串连起来操作。下面的代码与上面的代码是等同的。
 
```
RACSignal *intervalSignal = [RACSignal interval:1];  
RACSignal *startedIntervalSignal = [intervalSignal startWith:[NSDate date]];  
RACSignal *mappedIntervalSignal = [startedIntervalSignal map:^id(NSDate *value) {  
    NSDateComponents *dateComponents = [[NSCalendar currentCalendar] components:NSMinuteCalendarUnit | NSSecondCalendarUnit fromDate:value];  
      
    return [NSString stringWithFormat:@"%d:%02d", dateComponents.minute, dateComponents.second];  
}];  
   
RAC(self.textField.text) = mappedIntervalSignal;  
```

####二、Splitting模式

Splitting与chaining比较类似，也是将signal转换为其它的sginal，不同之处在于，Splitting会重复使用signal。Splitting看起来要复杂些，其实也就是一个signal使用多次。

```
RACSignal *dateComponentsSignal = [[[RACSignal interval:1] startWith:[NSDate date]] map:^id(NSDate *value) {  
    NSDateComponents *dateComponents = [[NSCalendar currentCalendar] components:NSMinuteCalendarUnit | NSSecondCalendarUnit fromDate:value];  
    return dateComponents;  
}];  
   
RAC(self.minuteTextField.text) = [dateComponentsSignal map:^id(NSDateComponents *dateComponents) {  
    return [NSString stringWithFormat:@"%d", dateComponents.minute];  
}];  
   
RAC(self.secondTextField.text) = [dateComponentsSignal map:^id(NSDateComponents *dateComponents) {  
    return [NSString stringWithFormat:@"%d", dateComponents.second];  
}];  
```
在上面这个例子中，创建了一个signal，即局部变量：dateComponentsSignal。接着再用dateComponentsSignal创建两个新的signal，并将它们分别与两个textfield的text属性进行绑定。

> Note: 把dateComponentsSignal看作一个局部变量，就像,int a = 9; 然后多次使用a来计算。
 
![Splitting](http://teehanlax.com.s3.amazonaws.com/wordpress/wp-content/uploads/Splitting.png) 
 
####三、Combining模式

combining就是将几个signal结合起来创建出一个新的signal。比如“登录”按钮，只有在“用户名”与“密码”输入框中的文本长度都超过6时才能被点击，否则处于不可用的状态。那么我们可以为“登录”按钮的enabled状态创建一个signal，这个signal则是由“用户名”与“密码”框它们两个自己的signal组合起来：

```
RAC(self.submitButton.enabled) = [RACSignal combineLatest:@[self.usernameField.rac_textSignal, self.passwordField.rac_textSignal] reduce:^id(NSString *userName, NSString *password) {  
    return @(userName.length >= 6 && password.length >= 6);  
}]; 
```

在这里，我们将“登录”按钮的enable状态绑定到使用combineLatest:reduce:方法创建的signal上。这个方法的第二个参数是一个block，这个block的参数是combineLatest中的参数的**最新值**的组合。我们将两个文本框的text signal一起传到combineLatest，在reduce的block中，该block也就会接收到两个NSString的参数，这个block的工作就是将两个参数值组合起来生成一个值，然后返回。该方法的说明：

```
// +combineLatest:reduce: takes an array of signals, executes the block with the
// latest value from each signal whenever any of them changes, and returns a new
// RACSignal that sends the return value of that block as values.
```


![Combining](http://teehanlax.com.s3.amazonaws.com/wordpress/wp-content/uploads/combining.png)

Combining常用于两种情况：

 1. 需要同时满足多种条件。
 2. 在多个signal中进行选择。

优秀文章
----------
[Getting Started with ReactiveCocoa](http://www.teehanlax.com/blog/getting-started-with-reactivecocoa/)
 