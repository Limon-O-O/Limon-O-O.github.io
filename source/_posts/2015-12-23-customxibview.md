---
layout: post
title: "优雅地自定义XibView"
date: 2015-12-23 21:38:17 +0800
comments: true
categories: Producter-Tip
---

##优雅地自定义XibView

好习惯，先上[源码](https://github.com/Limon-catch/XibView)。

先创建一个UIView文件和一个Xib文件，在Xib文件里设置如下，

![](https://raw.githubusercontent.com/Limon-catch/XibView/master/XibView/Image/XibView_Step1.png)

在UIView文件中，不是用`-awakeFromNib()`作为构造器，而是正常的`-init(frame: CGRect)`。

```swift

override init(frame: CGRect) {
	super.init(frame: frame)

	xibSetup()
}

```
而`-xibSetup()`才是关键，具体可看源码。

<br />
如果需要UIView和Xib文件建立控件属性关联，是设置Xib文件的File`s Owner，***而不是设置View的Custom Class***。

![](https://raw.githubusercontent.com/Limon-catch/XibView/master/XibView/Image/XibView_Step2.png)

设置了File`s Owner就可以像往常一样拖线了。


如果想在`Main.storyboard`文件中直接使用此Xib，同时也想在SB中设置属性，那怎么使用呢？

1. 在SB中加入一个UIView，将其Class设置成`XibView`<br />
2. 使用`@IBInspectable`


```
@IBInspectable var title: String? {
	get {
	    return xibLabel.text
	}
	set {
	    xibLabel.text = newValue
	}
}
```

<br />
添加了`@IBInspectable`之后，就可以像系统自带的控件一样设置属性了。

![](https://raw.githubusercontent.com/Limon-catch/XibView/master/XibView/Image/XibView_Step3.png)
