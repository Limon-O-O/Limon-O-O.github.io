---
layout: post
title: "CheatSheet"
date: 2015-12-23 04:22:06 +0800
comments: true
categories: CheatSheet
coderay_line_numbers:
---



#### Swift Either enum

```swift
enum Either<A, B>{
  case Left(A)
  case Right(B)
}

let x: Either<SomeType, AnotherType> = ...

switch x {
case .Left(let someTypeValue):
  // do something with someTypeValue
case .Right(let anotherTypeValue):
  // do something with anotherTypeValue
}
```

一个数据源，两种类型

```swift

let data: Either<CellModel, BannerModel> = ...

func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
  return data.count
}

func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
    let item = data[indexPath.row]

    let cell = tableView.dequeueReusableCell(withIdentifier: identifier, for: indexPath)

    switch item {
    case .Left(let pizza):
        configure(cell, with: pizza)
    case .Right(let ad):
        configure(cell, with: ad)
    }

    return cell
}

```


#### Swift 几种不错的遍历

```swift

for i in 1...10 where i % 2 == 0 {
    print("\(i) is even")
}

for word in ["Lorem", "ipsum", "dolor", "sit", "amet", "consectetur",
    "adipiscing", "elit"] where word.characters.count > 5 {
     print(word, "is a long word")
}


let items: [String?] = [nil, nil, "Hello", nil, "World"]
for case let item? in items {
    print(item)
}

let mySwitch = UISwitch()
let myView = UIView()
let myDate = NSDate()

let myItems: [NSObject] = [mySwitch, myDate, myView]
for case let item as UIView in myItems {
    print(item.frame)
}


protocol MyCustomProtocol {
    var frame: CGRect {get}
}

struct Bar {}
struct Foo: MyCustomProtocol {
    let frame: CGRect = .zero
}
extension UIView: MyCustomProtocol {}

let otherItems = [Bar(), Foo(), UIView()] as [Any]
for case let item as MyCustomProtocol in otherItems {
    print(item.frame)
}
```

<br />

#### 字典合并

```
var parameters = [
    "page": "1",
    "offset": "20"
]

let params = ["type": "news"]

parameters.ifr_merge(params) // Output: ["offset": "20", "type": "news", "page": "1"]


extension Dictionary {

    mutating func ifr_merge<K, V>(dictionaries: Dictionary<K, V>...) {
        for dict in dictionaries {
            for (key, value) in dict {
                if let v = value as? Value, k = key as? Key {
                    self.updateValue(v, forKey: k)
                }
            }
        }
    }
}
```

<br />

#### 优化 Cell 的使用体验

```
collectionView.registerNib(UINib(nibName: NewsCell.ifr_className, bundle: nil), forCellWithReuseIdentifier: NewsCell.ifr_className)
let cell = collectionView.dequeueReusableCellWithReuseIdentifier(NewsCell.ifr_className, forIndexPath: indexPath) as! NewsCell

extension UICollectionReusableView {
    static var ifr_className: String {
        return "\(self)"
    }
}
```

<br />

#### 优化 ReloadData、Insert 的使用体验

```
var wayToUpdate: UICollectionView.WayToUpdate = .None
wayToUpdate = page == 1 ? .ReloadData : .Insert(indexPaths)
wayToUpdate.performWithCollectionView(self.collectionView)
```

```
extension UICollectionView {

    enum WayToUpdate {

        case None
        case ReloadData
        case Insert([NSIndexPath])

        var needsLabor: Bool {

            switch self {
            case .None:
                return false
            case .ReloadData:
                return true
            case .Insert:
                return true
            }
        }

        func performWithCollectionView(collectionView: UICollectionView) {

            switch self {
                case .None:
                    break
                case .ReloadData:
                    collectionView.reloadData()
                case .Insert(let indexPaths):
                    collectionView.insertItemsAtIndexPaths(indexPaths)
            }
        }
    }
}
```

<br />

#### 字体的适配

```
UIFont.ifr_adaptiveFont(.Regular(fontSize: 16.0)
```

```
enum FontWeight {
    case Light(fontSize: CGFloat)
    case Regular(fontSize: CGFloat)
    case Medium(fontSize: CGFloat)
    case Bold(fontSize: CGFloat)
}

extension UIFont {

    class func ifr_adaptiveFont(weight: FontWeight) -> UIFont {

        var weightString = ""
        var size: CGFloat = 16.0

        switch weight {

            case .Light(let fontSize):
                weightString = "-Light"
                size = fontSize

            case .Regular(let fontSize):
                weightString = ""
                size = fontSize

            case .Medium(let fontSize):
                weightString = "-Medium"
                size = fontSize

            case .Bold(let fontSize):
                weightString = "-Bold"
                size = fontSize
        }

        guard let systemFont = UIFont(name: "PingFangSC\(weightString)", size: size) ??
            UIFont(name: "STHeitiSC\(weightString)", size: size) else {
                return UIFont.systemFontOfSize(size)
        }

        return systemFont
    }
}
```

<br />

#### HTML 字符转义

```swift
var ifr_htmlEntityDecode: String {
    var text = stringByReplacingOccurrencesOfString("&quot;", withString: "\"")
    text = text.stringByReplacingOccurrencesOfString("&apos;", withString: "'")
    text = text.stringByReplacingOccurrencesOfString("&lt;", withString: "<")
    text = text.stringByReplacingOccurrencesOfString("&gt;", withString: ">")
    text = text.stringByReplacingOccurrencesOfString("&amp;", withString: "&")

    return text
}
```

<br />

#### repeat 一张小图片

```swift

let patternImage = UIImage(named: "comment_bottomLine")
bottomImageView.image = nil
bottomImageView.backgroundColor = UIColor(patternImage: patternImage!)

```

<br />

#### NSURL 解析

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

#### 获取版本号和 Build 号
```
NSBundle.mainBundle().infoDictionary?["CFBundleVersion"] as? String // Build
NSBundle.mainBundle().infoDictionary?["CFBundleShortVersionString"] as? String // 版本号
```

<br />

#### 判断一个 view 是否是另一个 view 的子视图

```swift
- (BOOL)isDescendantOfView:(UIView *)view;
```
<br />

#### 隐藏或显示"隐藏文件"

```
defaults write com.apple.finder AppleShowAllFiles -boolean true; killall Finder
```

<br />


#### 设置圆角（左上角，右上角）

```
UIBezierPath(roundedRect: ScreenBounds, byRoundingCorners: [.TopLeft, .TopRight], cornerRadii: CGSize(width: cornerRadius, height: 0.0))
```
<br />

#### 阴影
```
whiteView.layer.shadowColor = UIColor.redColor().CGColor
whiteView.layer.shadowOffset = CGSizeMake(0, 1)
whiteView.layer.shadowOpacity = 1.0
whiteView.layer.shadowRadius = 20.0
whiteView.layer.shadowPath = UIBezierPath(rect: whiteView.bounds).CGPath
```

<br />

#### BasicAnimation

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

#### 禁止 `WKWebView` 长按复制

```
let source = "var style = document.createElement('style'); style.type = 'text/css'; style.innerText = '*:not(input):not(textarea) { -webkit-user-select: none; -webkit-touch-callout: none; }'; var head = document.getElementsByTagName('head')[0]; head.appendChild(style);";

let script: WKUserScript = WKUserScript(source: source as String, injectionTime: .AtDocumentEnd, forMainFrameOnly: true)

// Create the user content controller and add the script to it
let userContentController: WKUserContentController = WKUserContentController()
userContentController.addUserScript(script)

// Create the configuration with the user content controller
let configuration: WKWebViewConfiguration = WKWebViewConfiguration()
configuration.userContentController = userContentController
```

<br />

#### 禁止 `WKWebView` 放大缩小

```
// Javascript that disables pinch-to-zoom by inserting the HTML viewport meta tag into <head>
let source: NSString = "var meta = document.createElement('meta');" +
    "meta.name = 'viewport';" +
    "meta.content = 'width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no';" +
    "var head = document.getElementsByTagName('head')[0];" +
"head.appendChild(meta);";

let script: WKUserScript = WKUserScript(source: source as String, injectionTime: .AtDocumentEnd, forMainFrameOnly: true)

// Create the user content controller and add the script to it
let userContentController: WKUserContentController = WKUserContentController()
userContentController.addUserScript(script)

// Create the configuration with the user content controller
let configuration: WKWebViewConfiguration = WKWebViewConfiguration()
configuration.userContentController = userContentController

```

<br />

#### 根据触摸点判断是否 touch 点击了某个 view

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


#### 隐藏键盘(点击屏幕任意位置)

```
-(void)touchesBegan:(NSSet *)touches withEvent:(UIEvent *)event{
	[super touchesBegan:touches withEvent:event];
	[self.view endEditing:YES];
}
```
<br />


在包含 `UITableView` 视图中添加单击手势
如果在包含 `UITableView` 视图中添加单击手势，这个单击手势会屏蔽掉 `UITableView `的 `-didSelectRowAtIndexPath`

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

简单点的就将单击手势的 `cancelsTouchesInView` 设置为 NO 即可

```swift
singleTap.cancelsTouchesInView = false
```
> 默认为YES，若NO，Gesture Recognizers和hit-test view同时响应触摸序列

<br />


#### Octopress
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

deploy 时 `non-fast-forward` 问题解决方案

```
cd octopress/_deploy
git pull origin master
cd ..
rake deploy
```

<br />


##### Can't ignore UserInterfaceState.xcuserstate

```
git rm --cached ProjectFolder.xcodeproj/project.xcworkspace/xcuserdata/myUserName.xcuserdatad/UserInterfaceState.xcuserstate
git commit -m "Removed file that shouldn't be tracked"
```

