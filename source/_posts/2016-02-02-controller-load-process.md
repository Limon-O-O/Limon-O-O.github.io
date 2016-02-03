---
layout: post
title: "[译] 揭秘控制器视图的加载过程"
date: 2016-02-02 04:20:57 +0800
comments: true
categories: 译文
---

原文链接:
[iOS: UIViewController’s view loading process demystified](http://szulctomasz.com/ios-uiviewcontrollers-view-loading-process-demystified/)

[@NatashaTheRobot](https://twitter.com/NatashaTheRobot)在她的博客里发过一篇关于测试控制器及视图加载问题的文章[The One Weird Trick For Testing View Controllers in Swift](http://natashatherobot.com/ios-testing-view-controllers-swift/)

>
她写道：
	这里的关键是苹果重载了控制器视图的getter去调用加载视图的方法，并做了许多我们没有权限访问的其它事情。如果有谁深入了解过其中的工作原理，尽情地说出来吧。

了解背后的工作原理确实是一个很有趣的事情。在她的鼓励下，我开始进行深入的探索。

我发现有两种情况：控制器被设置为 window 的根控制器或者 控制器不作为根控制器（例如，当你想要测试视图控制器，并从 Storyboard 中实例化了一个时）

## 控制器作为根控制器

	if self.window == nil {
	    self.window = UIWindow(frame: UIScreen.mainScreen().bounds)
	}

	let storyboard = UIStoryboard(name: "Main", bundle: NSBundle.mainBundle())
	let vc = storyboard.instantiateViewControllerWithIdentifier("ViewController")
	self.window!.rootViewController = vc

比如说，这部分程序流运行在 `-application:DidFinishLaunchingWithOptions:` 方法中，
![](http://szulctomasz.com/wp-content/uploads/2015/08/ex-1.png)
`UIWindow` 的 `-makeKeyAndVisible` 是第一个被调用的方法，此方法调用了 UIWindow 的私有方法 `-addRootViewControllerViewIfPossible` 去为其根控制器添加 view，并显示出来。`UIWindow`  获取根控制器的 `view` 属性之后，就开始了视图加载的一系列过程，首先被调用的方法是 `-loadViewIfRequired`，此方法再调用 `-loadView` 方法。`-loadView` 方法调用 `UIViewController` 的的内部方法来加载 Nib 以达到设置视图的目的。

今年（2015年）的 WWDC 上有个非常好的演讲，介绍了 `storyboard` 及 `nibs` 在运行时是如何运作的。[Implementing UI Design in Interface Builder](https://developer.apple.com/videos/wwdc/2015/?id=407)

在 `loadView` 设置好后，控制器调用它内部的方法 `-_window`，并读取了诸如`preferedInterfaceOrientation`、` supportedInterfaceOrientations`、`shouldAutorotate`的许多设置。事实上，控制器调用了 `-_window` 许多次，此外也调用了许多其它方法。

接着，`-viewDidLoad` 方法被调用，私有 `-__viewWillAppear` 调用 `-viewWillAppear`。视图即将被呈现出来，所以 `-willMoveToWindow:` `-willMoveToSuperview:` 和私有 `-_didMoveFromWindow:toWindow:` 等方法被调用。

下一步是为视图作自动布局，所以 `layoutMarginsDidChange, didMoveToWindow, didMoveToSuperview, updateViewConstraints, updateConstraints, layoutSublayersOfLayer, viewWillLayoutSubviews, layoutSubviews` 等一系列方法被调用

最后，`-viewDidAppear:`被调用了，视图也呈现出来了。


####.


## 普通控制器的加载测试


这是另外一个值得去研究的重要情况。在这种情况下，你不用创建 `window` 并把你的 `view controller` 加到其上来进行测试，只需要通过storyboard创建控制器的实例以进行测试。在看了 Natasha 的文章之前，我只知道直接通过访问控制器的 `view` 属性来获取视图。现在我知道在 iOS9 之后，`-loadViewIfNeeded` 方法和访问属性 `view` 的行为是一样的。并且在[Ørta](https://github.com/orta)的推荐下，我知道了两个更好的方法 去测试控制器视图的加载并确保其可用。`-beginAppearanceTransition:animated` 和 `-endApperanceTransition`。[这是他推荐的方法](https://github.com/artsy/eigen/blob/master/Artsy_Tests/Extensions/UIViewController+PresentWithFrame.m#L20-L22)

### 直接访问View属性

让我们看看直接访问 `view` 时的情况。

	let storyboard = UIStoryboard(name: "Main", bundle: NSBundle.mainBundle())
	let vc = storyboard.instantiateViewControllerWithIdentifier("ViewController")
	_ = vc.view


![](http://szulctomasz.com/wp-content/uploads/2015/08/ex-2.png)

好吧，许多方法没被调用。`window` 为 `nil`，视图可以被访问和加载，图上的加载过程却在 `-viewDidLoad` 中就结束了，所以我认为还有许多方法没被找出来。

让我们看看 `-beginApperanceTransition:animated:` 是怎样工作的。

### beginApperanceTransition:animated

	let storyboard = UIStoryboard(name: "Main", bundle: NSBundle.mainBundle())
	let vc = storyboard.instantiateViewControllerWithIdentifier("ViewController")
	vc.beginAppearanceTransition(true, animated: false)
	vc.endAppearanceTransition()

![](http://szulctomasz.com/wp-content/uploads/2015/08/ex-3.png)

非常好，这次的加载过程比第一个要详细。可以看到，这里 `window` 同样是 `nil`，所以控制器的视图，没有出现在加载过程中，也没有设置布局的代码在运行，仅仅是控制器的一些设置。

> 译者：这里可能很难理解，可以结合本文的三个图，图都是分三列，分别为 `UIWindow`、`UIViewController`、`UIView`，这里说 `window` 同样是 `nil` 的意思是，`UIWindow` 列没出现东西，原因是控制器不是根控制器，这里仅仅测试单独一个控制器。作者在这里想表达的意思是


## 总结

现在应该知道，视图的加载过程取决于控制器的上下文。

在第一个例子中，控制器作为根控制器，所以控制器和视图都进行了许多设置。

在最后的一个例子中，因为没有 `UIWindow `，只有控制器初始化了，连视图也没有初始化的需要 必要。这种情况就类似于模拟 `present` 一个控制器，因此 `-viewWillAppear` 和 `-viewDidAppear` 方法被调用了。这在某些情况下或许很重要。

最后，我要再谈谈第二个例子。
在不知道 `-beginApperanceTransition:animated` 之前，我以为 viewWill/viewDid 会被调用，但是并没有，就如以上图二所示。Ørta 提供的方法可能是测试控制器的最好途径之一。
