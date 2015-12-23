---
layout: post
title: "CheatSheet"
date: 2015-12-23 02:22:06 +0800
comments: true
categories: CheatSheet
---

NSURL 解析

```swift

extension NSURL {
    var prt_URLItems: [String: String]? {

        let components = NSURLComponents(URL: self, resolvingAgainstBaseURL: false)

        guard let items = components?.queryItems else {
            return nil
        }

        var infos = [String: String]()
        items.forEach {
            infos[$0.name] = $0.value
        }
        return infos
    }
}

```


```swift

let items = NSURL(string: "http://www.weibo.com/1783821582/profile?rightmod=1&wvr=6&mod=personinfo")?.prt_URLItems
print(items)

// Optional(["mod": "personinfo", "rightmod": "1", "wvr": "6"])

```

<br />


判断一个view是否是另一个view的子视图

```swift
- (BOOL)isDescendantOfView:(UIView *)view;
```
<br />

隐藏或显示"隐藏文件"

```
defaults write com.apple.finder AppleShowAllFiles -boolean true; killall Finder
```

<br />


设置圆角（左上角，右上角）

```
UIBezierPath(roundedRect: ScreenBounds, byRoundingCorners: [.TopLeft, .TopRight], cornerRadii: CGSize(width: cornerRadius, height: 0.0))
```
<br />


BasicAnimation

```
let pathAnimation = CABasicAnimation(keyPath: "path")
pathAnimation.fromValue = startPath
pathAnimation.toValue = endPath
pathAnimation.duration = 0.3
pathAnimation.timingFunction = CAMediaTimingFunction(name: kCAMediaTimingFunctionEaseOut) // animation curve is Ease Out
pathAnimation.fillMode = kCAFillModeBoth // keep to value after finishing
pathAnimation.removedOnCompletion = false // don't remove after finishing

shapeLayer.addAnimation(pathAnimation, forKey: pathAnimation.keyPath)
```
>[Fun with CAShapeLayer](http://jamesonquave.com/blog/fun-with-cashapelayer/)

<br />


根据触摸点判断是否touch点击了某个View

```
override func touchesBegan(touches: Set<NSObject>, withEvent event: UIEvent) {
    super.touchesBegan(touches, withEvent: event)

    if let location = (touches.first as? UITouch)?.locationInView(self.view) {
        let searchCircleY = bottomView.frame.origin.y + searchCircleImageView.frame.origin.y
        let frame = CGRect(origin: CGPoint(x:searchCircleImageView.frame.origin.x, y:searchCircleY) , size: searchCircleImageView.size)
        if CGRectContainsPoint(frame, location) { // 点击在searchCircleImageView
            dismissViewControllerAnimated(true, completion: nil)
        }
    }
}
```
<br />


隐藏键盘(点击屏幕任意位置)

```
-(void)touchesBegan:(NSSet *)touches withEvent:(UIEvent *)event{
	[super touchesBegan:touches withEvent:event];
	[self.view endEditing:YES];
}
```
<br />


在包含UITableView视图中添加单击手势
如果在包含UITableView视图中添加单击手势，这个单击手势会屏蔽掉UITableView的`didSelectRowAtIndexPath`

在单击点位于UITableView内的时候取消响应

```
- (BOOL)gestureRecognizerShouldBegin:(UIGestureRecognizer *)gestureRecognizer{
	CGPoint point = [gestureRecognizer locationInView:self];
	if(CGRectContainsPoint(menuTableView.frame, point)){
    	return NO;
	}
	return YES;
}
```

简单点的就将单击手势的cancelsTouchesInView设置为NO即可

```swift
singleTap.cancelsTouchesInView = false
```
> 默认为YES，若NO，Gesture Recognizers和hit-test view同时响应触摸序列

<br />


Octopress
```
rake "new_post[Post Title]" // zsh下
rake generate                                  生成html文件
rake preview
rake deploy                                    部署文章（博客）
```
```
git add .
git commit -m 'initial source commit'
git push origin source
```
<br />
