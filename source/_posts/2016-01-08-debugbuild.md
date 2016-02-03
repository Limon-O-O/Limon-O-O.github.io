---
layout: post
title: "Debug 模式与 Release 模式的区分"
date: 2016-01-08 11:31:30 +0800
comments: true
categories: Producter-Tip
---

# Debug 模式与 Release 模式的区分

<br />
区分 Debug 模式与 Release 模式有两种方法，此文的目的是告诉读者尽量避免用 `-DDEBUG`。
####第一种方法：

在 (Build Settings -> Swift Compiler - Custom Flags) 中加入 `-DDEBUG`，
[Stackoverflow](http://stackoverflow.com/questions/24111854/in-absence-of-preprocessor-macros-is-there-a-way-to-define-practical-scheme-spe/#answer-24112024)。

![](http://i.stack.imgur.com/dqp5H.png)

但是在 NSHipster 中有提到不推荐此方法 [Avoiding -DDEBUG in Swift](http://nshipster.com/new-years-2016/#avoiding--ddebug-in-swift)

<br />
####第二种方法

通过 Preprocessor Macros (预处理宏命令) 来区分模式。

<br />

1.新建一个 `PreProcessorMacros.h` 文件，代码如下

```swift
#import <Foundation/Foundation.h>

@interface PreProcessorMacros : NSObject

#ifndef PreProcessorMacros_h
#define PreProcessorMacros_h

extern BOOL const DEBUG_BUILD;

#endif

@end


@implementation PreProcessorMacros

#ifdef DEBUG
    BOOL const DEBUG_BUILD = YES;
#else
    BOOL const DEBUG_BUILD = NO;
#endif

@end

```

<br />

2.在 Bridged Header: `#import "PreProcessorMacros.h"`
> 也可直接把步骤一的代码放进 Bridged Header

<br />

3.测试

```
if DEBUG_BUILD {
    print("It's Debug build")
} else {
    print("It's Release build")
}
```

**⌘⇧<** 进入下图可模式切换

![](https://raw.githubusercontent.com/Limon-O-O/DEBUGBUILD/master/images/switch.png)

<br />

## 原理

关键的代码在：
```
#ifdef DEBUG
    BOOL const DEBUG_BUILD = YES;
#else
    BOOL const DEBUG_BUILD = NO;
#endif
```
此处的 `DEBUG` 究竟来自哪？

进入 'Build Settings' -> 搜索 'Preprocessor Macros'，真相大白

![](https://raw.githubusercontent.com/Limon-O-O/DEBUGBUILD/master/images/PreprocessorMacros.png)

<br />
源码：[DEBUGBUILD](https://github.com/Limon-O-O/DEBUGBUILD)

抽取于干货极多的：[Reader Submissions -
New Year's 2016](http://nshipster.com/new-years-2016/)  🍺
