---
layout: page
title: "private"
date: 2017-03-12 17:51
comments: false
footer: true
---

<br />
<p><a name="Notes"></a></p>
<br />

# Notes

### 界面渲染流程

点击 Button
点击屏幕，SpringBoard.app 通过 IPC(进程间通信) 转发消息给 App， 由 runloop 的 source1 处理，然后在下一个 runloop 由 source0 通过 UIApplication 把 UIEvent 转发给 UIWindow，然后再通过 hitTest ，找到相应的 view，然后看有没 UIResponder 响应，此时 button 响应，触发 `CA::Transaction::commit()` 提交中间状态，最终在 RunLoop 即将进入休眠（或者退出）时，由 QuartzCore Framework 内的 CoreAnimation 把中间状态通过 IPC 提交到 GPU。


Core Animation 的核心是 OpenGL ES 的一个抽象物，所以大部分的渲染是直接提交给 GPU 来处理。

UIView 是继承自 UIResponder 的，所以说 UIView 是可以响应事件的，而 CALayer 是不能的。
也就是说 UIView 负责处理用户交互，负责绘制内容的则是它持有的那个 CALayer。

CALayer 是负责绘制内容管理的一个类，而真正的绘制部分是由 CoreGraphics Framework 框架来处理的。CoreGraphics 中包括下图所示的各种类来处理绘制，比如：path 的绘图工作(如，CGPath)、变形操作(如，CGAffineTransform)、颜色管理(如，CGColor)、离屏渲染(如，CGBitmapContextCreateImage)、渲染模式(patterns)、渐变(gradients)、阴影效果、图形数据管理、图形创建、蒙版以及PDF文档的创建、显示和解析等等。

CoreGraphics 负责创建显示到屏幕上的数据模型，QuartzCore(CoreAnimation –> OpenGLES)负责把CoreGraphics 创建的数据模型真正显示到屏幕上。 CG 打头的类都是属于 CoreGraphics Framework

[CALayer drawInContext:] ()



着色器程序， 在 OpenGL ES 中着色器程序必须创建两种着色器：顶点着色器 (vertex shaders) 和片段着色器 (fragment shaders)


顶点着色器定义了在 2D 或者 3D 场景中几何图形是如何处理的。一个顶点指的是 2D 或者 3D 空间中的一个点。在图像处理中，有 4 个顶点：每一个顶点代表图像的一个角。顶点着色器设置顶点的位置，并且把位置和纹理坐标这样的参数发送到片段着色器。

顶点着色器，定义在 2D 或者 3D 场景中几何图形是如何处理的

经过 图元装配、光栅化
在光栅化(像素化)阶段，基本图元被转换为二维的片元(fragment)，fragment 表示可以被渲染到屏幕上的像素，它包含位置，颜色，纹理坐标等信息

片段着色器实现了一个通用的可编程操作片段的方法.片段着色器执行由光栅化生成的每个片段。

片段着色器计算出每个像素的最终颜色

片段着色器的目的就是确定一个像素的颜色

再进行一些列测试，剪裁测试、模版测试、深度测试

混合、抖动

帧缓冲区

视频控制器会按照 VSync 信号逐行读取帧缓冲区的数据，经过可能的数模转换传递给显示器显示

[iOS 事件处理机制与图像渲染过程](http://mp.weixin.qq.com/s?__biz=MzAwNDY1ODY2OQ==&mid=400417748&idx=1&sn=0c5f6747dd192c5a0eea32bb4650c160&scene=4#wechat_redirect)

[界面渲染的整体流程](http://blog.handy.wang/blog/2015/10/03/uiviewyu-calayerxie-zuo-xuan-ran-jie-mian-de-guo-cheng/)

CFRunLoopSourceRef 是事件产生的地方。Source有两个版本：Source0 和 Source1。

• Source0 只包含了一个回调（函数指针），它并不能主动触发事件。使用时，你需要先调用 CFRunLoopSourceSignal(source)，将这个 Source 标记为待处理，然后手动调用 CFRunLoopWakeUp(runloop) 来唤醒 RunLoop，让其处理这个事件。

• Source1 包含了一个 mach_port 和一个回调（函数指针），被用于通过内核和其他线程相互发送消息。这种 Source 能主动唤醒 RunLoop 的线程。


#### 为什么 `shouldRasterize`, `mask`, `shadows` 等会触发离屏渲染？
猜测，因为需要更高级的功能来处理，可能“窗口系统提供的“帧缓冲区并没”高级“附件来处理，所以需要新创建一个新的帧缓冲区来处理，由于新的帧缓冲区不是默认的帧缓冲区，渲染命令对窗口的可视输出不会产生任何影响。出于这个原因，它被称为离屏渲染（off-screen rendering）

比如：阴影，需要深度缓冲(depth buffer)和模板缓冲(stencil buffer)，而默认的帧缓冲区只有 color buffer，比如 GPUImage 的 FBO 默认只添加了 color buffer

```
// Attach color buffer to FBO on GL_COLOR_ATTACHMENT0
glFramebufferRenderbuffer(GLenum(GL_FRAMEBUFFER), GLenum(GL_COLOR_ATTACHMENT0), GLenum(GL_RENDERBUFFER), displayRenderbuffer)
```

> FBO 是一个容器，自身不能用于渲染，需要与 `纹理` 或 `渲染缓冲（renderbuffer）对象` 绑定在一起。
	Render Buffer Object（RBO）即为渲染缓冲对象，分为 color buffer(颜色)、depth buffer(深度)、stencil buffer(模板)
	

#### 关于 FBO离屏渲染
所谓的 FBO 就是Frame Buffer Object。之前我们使用 OpenGLES 渲染，都是直接渲染到屏幕上，FBO可以让我们的渲染不渲染到屏幕上，而是渲染到离屏Buffer中。这样的作用是什么呢？比如我们需要处理一张图片，在上传时增加时间的水印，这个时候不需要显示出来的。再比如我们需要对摄像头采集的数据，一个彩色原大小的显示出来，一个黑白的长宽各一半录制成视频。 
像这些情况，我们就可以使用到 FBO离屏渲染 技术了，当然 FBO 并不是仅仅局限于此。

FBO 是一个容器，自身不能用于渲染，需要与一些可渲染的缓冲区绑定在一起，像纹理或者渲染缓冲区。

Render Buffer Object（RBO）即为渲染缓冲对象，分为color buffer(颜色)、depth buffer(深度)、stencil buffer(模板)。 
在使用FBO做离屏渲染时，可以只绑定纹理，也可以只绑定Render Buffer，也可以都绑定或者绑定多个，视使用场景而定。如只是对一个图像做变色处理等，只绑定纹理即可。如果需要往一个图像上增加3D的模型和贴纸，则一定还要绑定depth Render Buffer。 
同 Texture 使用一样，FrameBuffer 使用也需要调用 GLES20.glGenFrameBuffers 生成 FrameBuffer，然后在需要使用的时候调用 GLES20.glBindFrameBuffer。

### 为什么在 `-drawRect:` 用了 CoreGraphics 会造成内存高涨？

Core Graphics绘制 - 如果对视图实现了-drawRect:方法，或者CALayerDelegate的-drawLayer:inContext:方法，那么在绘制任何东西之前都会产生一个巨大的性能开销。为了支持对图层内容的任意绘制，Core Animation必须创建一个内存中等大小的**寄宿图**。然后一旦绘制结束之后，必须把图片数据通过IPC传到渲染服务器。在此基础上，Core Graphics绘制就会变得十分缓慢，所以在一个对性能十分挑剔的场景下这样做十分不好。


## OpenGL ES

[Android OpenGLES2.0（十二）——FBO离屏渲染](http://www.voidcn.com/blog/junzia/article/p-6354808.html)
[模板缓冲区](http://www.twinklingstar.cn/2014/1176/stencil-buffer/)
[LearnOpenGL-帧缓冲区](https://learnopengl-cn.readthedocs.io/zh/latest/04%20Advanced%20OpenGL/05%20Framebuffers/)
[内存恶鬼drawRect](http://bihongbo.com/2016/01/03/memoryGhostdrawRect/)

## Autorelease

1. Autoreleasepool 与 Runloop 的关系
    主线程默认为我们开启 Runloop，Runloop 会自动帮我们创建 Autoreleasepool，并进行Push、Pop 等操作来进行内存管理，在即将退出 Loop 时调用 _objc_autoreleasePoolPop() 来释放自动释放池
    
    
2. Autorelease 对象什么时候释放？
    Autorelease 对象是在当前的 runloop 迭代结束时释放的，而它能够释放的原因是系统在每个 runloop 迭代中都加入了自动释放池 Push 和 Pop

3. 什么对象自动加入到 Autoreleasepool 中
    ##### 第一种
    当使用 `alloc/new/copy/mutableCopy` 进行初始化时，会生成并持有对象(也就是不需要 pool 管理，系统会自动的帮他在合适位置 release)
       `id obj = [NSMutableArray array];`
       这种情况会自动将返回值的对象注册到autorealeasepool，代码等效于：
       
       ```
       @autorealsepool{
        id __autorealeasing obj = [NSMutableArray array];
}
```

    ##### 第二种
    __weak修饰符只持有对象的弱引用

    ##### 第三种
    id的指针或对象的指针在没有显式指定时会被附加上__autorealeasing修饰符
    
# Runtime

```
struct objc_object {  
    Class isa  OBJC_ISA_AVAILABILITY;
};

struct objc_class {  
    Class isa  OBJC_ISA_AVAILABILITY;
#if !__OBJC2__
    Class super_class;
    const char *name;
    long version;
    long info;
    long instance_size;
    struct objc_ivar_list *ivars;
    **struct objc_method_list **methodLists**;
    **struct objc_cache *cache**;
    struct objc_protocol_list *protocols;
#endif
};

struct objc_method {  
    SEL method_name;
    char *method_types;    /* char指针，其实存储着方法的参数类型和返回值类型 */
    IMP method_imp; /* 指向了方法的实现，本质上是一个函数指针 */
};

struct objc_ivar {
    char *ivar_name                                          
    char *ivar_type                                          
    int ivar_offset                                          
#ifdef __LP64__
    int space                                                
#endif
} 
```


## isa

每一个对象都有一个名为 isa 的指针，指向该对象的类。每一个类描述了成员变量的列表，成员函数的列表，还有对象能够接收的消息列表

## KVO

当你观察一个对象时，一个新的类会动态被创建。这个类继承自该对象的原本的类，并重写了被观察属性的 setter 方法。自然，重写的 setter 方法，并插入 `willChangeValueForKey` 和 `didChangeValueForKey`。最后把这个对象的 isa 指针 ( isa 指针告诉 Runtime 系统这个对象的类是什么 ) 指向这个新创建的子类，对象就神奇的变成了新创建的子类的实例。

## +(void)load; +(void)initialize；有什么用处？

在Objective-C中，runtime会自动调用每个类的两个方法。+load会在类初始加载时调用，+initialize会在第一次调用类的类方法或实例方法之前被调用。这两个方法是可选的，且只有在实现了它们时才会被调用。 
共同点：两个方法都只会被调用一次。

## 消息发送、消息转发
`[obj foo];`

在objc动态编译时，会被转意为：

```
objc_msgSend(obj, @selector(foo));

objc_msgSend ( id self, SEL op, ... );

```

第一个参数 `id` 是一个指向类实例的指针，而这个类实例就是 `objc_object`，里面是含有一个 isa 指针，
isa 指针指向对象的类 `objc_class`，

`objc_class` 结构体内有：方法列表，成员变量列表，还有一个方法的 `cache`，

先在 `cache` 里有没缓存到这个方法，如果没有再去方法列表找，如果还没找到就到 `superclass` 好，如果找到了这个函数，就执行他的 IMP

如果最终还没找到，通常情况下抛出 `unrecognized selector sent to … ` 的异常。但在抛去异常之前，有三次拯救程序

1. 如果没有找到，Runtime 会发送 +resolveInstanceMethod: 或者 +resolveClassMethod: 尝试去 resolve 这个消息；

2. 如果 resolve 方法返回 NO，Runtime 就发送 -forwardingTargetForSelector: 允许你把这个消息转发给另一个对象；

3. 如果没有新的目标对象返回， Runtime 就会发送 -methodSignatureForSelector: 和 -forwardInvocation: 消息。你可以发送 -invokeWithTarget: 消息来手动转发消息或者发送 -doesNotRecognizeSelector: 抛出异常


###  Category
#### 为什么 Category 不能添加实例变量
在 Runtime 中，objc_class 结构体大小是固定的，不可能往这个结构体中添加数据，只能修改。所以 ivars 指向的是一个固定区域，之所以能在 Category 添加方法，那是因为 `methodLists` 是指向 `objc_method_list` 指针的指针

```
struct objc_ivar_list *ivars
struct objc_method_list **methodLists 
```

#### 为什么 Category 不能添加一个实例变量，而能添加属性，从 Category 的结构体可以看出

```
typedef struct category_t {
    const char *name;
    classref_t cls;
    struct method_list_t *instanceMethods;
    struct method_list_t *classMethods;
    struct protocol_list_t *protocols;
    struct property_list_t *instanceProperties;
} category_t;
```

#### 使用 Category 需要注意的地方

1. 在 Category 中是不能添加实例变量
2. Category 是在运行时加载，而不是在编译
3. Category 的方法被放到了新方法列表的前面，而原来类的方法被放到了新方法列表的后面，这也就是我们平常所说的 Category 的方法会“覆盖”掉原来类的同名方法，这是因为运行时在查找方法的时候是顺着方法列表的顺序查找的，它只要一找到对应名字的方法，就会罢休，殊不知后面可能还有一样名字的方法。

### 集合
[招聘一个靠谱的 iOS](https://dayon.gitbooks.io/-ios/content/chapter8.html)

[Objective-C Runtime](http://yulingtianxia.com/blog/2014/11/05/objective-c-runtime/)

[Objective-C 消息发送与转发机制原理](http://yulingtianxia.com/blog/2016/06/15/Objective-C-Message-Sending-and-Forwarding/)

[深入理解Objective-C：Category](http://tech.meituan.com/DiveIntoCategory.html)




<br />

<p><a name="animations"></a></p>

<br />


# Animations

[交互可控式动画转场](https://github.com/Touchwonders/Transition) -- 👍

[UITableView 末行插入 UI 效果扩展实用工具类](https://github.com/marty-suzuki/ReverseExtension) --- 把整个 TableView 倒转实现，思路很赞

[ CAMediaTimingFunction helper tool ](https://github.com/keefo/CATweaker)

[ 动画集合 ](https://github.com/cjwirth/awesome-ios-ui)

[Group Avatar](https://github.com/ttmdung203/MDGroupAvatarView)

[ProgressWindow](https://github.com/luowenxing/WXProgressWindow) --- 进度悬浮窗

[IBAnimatable](https://github.com/JakeLin/IBAnimatable/blob/master/Documentation/README.zh.md) --- 是一个帮助我们在Interface Builder和Swift playground里面设计UI, 交互, 导航模式, 换场和动画的开源库

[ 波浪 ](https://github.com/antiguab/BAFluidView)

[ 水波上涨填满 ](https://github.com/poolqf/FillableLoaders)

[ UIDynamics iOS 9 ](https://github.com/FancyPixel/BallSwift)

[ 点赞 ](https://github.com/okmr-d/DOFavoriteButton)

[粒子扩散效果的点赞按钮](https://github.com/dgytdhy/DGThumbUpButton)

[火柴文字下拉刷新](https://github.com/Fnoz/FNMatchPull)

[ A simple keyframe-based animation framework ](https://github.com/IFTTT/RazzleDazzle)

[ 加载动画集合 ](https://github.com/ninjaprox/NVActivityIndicatorView)

[ 一款Loading动画的实现思路 ](http://www.cocoachina.com/ios/20151202/14532.html)

[ 粘性加载动画 ](https://github.com/yoavlt/LiquidLoader)

[KYAnimatedPageControl 拥有两种动画样式的自定义 UIPageControl](https://github.com/KittenYang/KYAnimatedPageControl)

[利用贝塞尔曲线实现Q弹的下拉刷新](http://pandara.xyz/2015/10/29/jelly_refresh/?hmsr=toutiao.io&utm_medium=toutiao.io&utm_source=toutiao.io)

[水滴效果](https://github.com/PandaraWen/WaterDropViewDemo)

[ Label动画 ](https://github.com/overboming/ZCAnimatedLabel)

[滑动推动 Nav Bar 效果](https://github.com/andreamazz/AMScrollingNavbar)

[ NavigationBar 动画 ](https://github.com/gmertk/BusyNavigationBar)

[ NavigationBar的背景颜色，高度 ](https://github.com/ltebean/LTNavigationBar)

[ 动态改变UINavigationBar背景色 ](https://github.com/DanisFabric/RainbowNavigation/blob/master/README_CN.md)

[ 顺滑过渡导航栏（不同背景颜色） ](https://github.com/MoZhouqi/KMNavigationBarTransition) --- 👍

[ UILabel 动画 ](https://github.com/overboming/ZCAnimatedLabel)

[ 下拉刷新 PullToBounce ](https://github.com/entotsu/PullToBounce)

[ 菜单粘性按钮 ](https://github.com/yoavlt/LiquidFloatingActionButton)

[粘性动画菜单](https://github.com/yannickl/FlowingMenu)

[自适应边界的散开按钮](https://github.com/liuzhiyi1992/SpreadButton)

[瞬间爆炸菜单按钮](https://github.com/Nightonke/VHBoomMenuButton)

[浮动的 Button](https://github.com/noppefoxwolf/FlowBarButtonItem?utm_campaign=This%2BWeek%2Bin%2BSwift&utm_medium=web&utm_source=This_Week_in_Swift_71)

[浮动 Button](https://github.com/tinymind/LSFloatingActionMenu)

[折叠菜单](https://github.com/Yalantis/Context-Menu.iOS)

[折叠菜单 (Yalantis出品)](https://github.com/Yalantis/Persei)

[图片实现多层折叠效果](http://www.cocoachina.com/ios/20160104/14858.html)

[Cell 折叠动](https://github.com/Ramotion/folding-cell) --- 👍👍👍

[可展开型预览 TableView](https://github.com/liuzhiyi1992/ZYThumbnailTableView) --- 👍👍👍

[ 粉碎删除 ](https://github.com/MartinRGB/MTMaterialDelete)

[ 侧滑删除cell ](https://github.com/SergioChan/SCTableViewCell)

[ 如何制作一个炫酷好玩的爆炸效果 ](http://xxycode.com/ru-he-zhi-zuo-ge-xuan-ku-hao-wan-de-bao-zha-xiao-guo-2/)

[ Check box ](https://github.com/Boris-Em/BEMCheckBox)

[ 皮筋式弹性下拉即刷新 ](https://github.com/gontovnik/DGElasticPullToRefresh)

[皮筋式弹性下拉即刷新 文章 Elastic view animation using UIBezierPath ](http://iostuts.io/2015/10/17/elastic-bounce-using-uibezierpath-and-pan-gesture/)

[皮筋式动画转场效果](https://github.com/lkzhao/ElasticTransition)

[ 瞬间崩塌为小方块动画效果 ](https://github.com/Yalantis/StarWars.iOS)

[ page control ](https://github.com/TBXark/TKRubberIndicator)

[ HorizontalProgress 进程 ](https://github.com/AliThink/HorizontalProgress)

[ trello 的导航动效控件 ](https://github.com/SergioChan/SCTrelloNavigation)

[ Apple TV Parallax 效果的视图 ](https://github.com/DroidsOnRoids/MPParallaxView)

[ MaterialKit 是一个用 Swift 写的 Material Design 框架, 拥有多种漂亮的动画效果和样式 ](https://github.com/CosmicMind/MaterialKit)

[ Download Button Aniamtion ](https://github.com/Guidebook/gbkui-button-progress-view)

[Checkbox](https://github.com/vladislav-k/VKCheckbox)

[双滑块Slider控件](https://github.com/Magic-Unique/DoubleThumbSlider)

[捷输入并选择组件](https://github.com/Ramotion/reel-search) --- 👍👍

[心形波浪](https://github.com/AfryMask/AFWaveView)

[脉搏效果，适合附近的人之类的功能](https://github.com/shu223/Pulsator)

[各种输入框交互动画](https://github.com/mukyasa/MMTextFieldEffects)

[可拖拽的标签](https://github.com/lovels/LBTagView)

[Gradient 渐变 滑动](https://github.com/hyperoslo/Hue#user-content-gradients-1)

[联系人列表 HorizontalContacts](https://github.com/manuelescrig/MEVHorizontalContacts)

[仿 Evernote 的列表的下拉弹性效果](https://github.com/imwangxuesen/EvernoteAnimation)

[图片选择器](https://github.com/cbangchen/CBImagePicker)

[雅虎图片选择器](https://github.com/yahoo/YangMingShan)

[PageControls](https://github.com/popwarsweet/PageControls)

[全屏 blur](https://github.com/edwinbosire/SwiflyOverlay)

[涟漪效应](https://github.com/alsedi/RippleEffectView)

<br />
<br />

##CollectionView

[ CollectionView 转场动画 ](https://github.com/CezaryKopacz/CKWaveCollectionViewTransition)

[ZLSwipeableViewSwift](https://github.com/zhxnlai/ZLSwipeableViewSwift) --- 片滑动效果

[ 格瓦拉 cell移动动画Demo ](https://github.com/nathanwhy/HYAwesomeTransition)

[ UICollectionViews 拖动 ](https://github.com/nshintio/uicollectionview-reordering)

[ CollectionView实现的个人标签 ](https://github.com/alienjun/MyTags)

[ Wave CollectionView Transition ](https://github.com/CezaryKopacz/CKWaveCollectionViewTransition)

[ iOS 书本打开动画制作教程 ](http://t.cn/Rybxgpy?u=1783821582&amp;amp;amp;amp;amp;amp;m=3885611247173418&amp;amp;amp;amp;amp;amp;cu=1783821582)

[ 卡片 ](https://github.com/zhxnlai/ZLSwipeableViewSwift)

[ 卡片式垂直翻转动画 ](https://github.com/seedante/CardAnimation)

[ 相册翻开动画效果 ](https://github.com/seedante/SDECollectionViewAlbumTransition)

[ 瀑布流 ](https://github.com/demonnico/PinterestSwift)

[ 印象笔记交互效果 ](http://allsome.love/yin-xiang-bi-ji-jiao-hu-xiao-guo-de-shi-xian/?u=1783821582&amp;m=3905429706568663&amp;cu=1783821582&amp;ru=2029464644&amp;rm=3905408588445881)

[自定义UICollectionViewFlowLayout和UICollectionViewLayout的两个例子](https://github.com/SmallLang/UICollectionViewLayoutDemo)

[响应式CollectionView](https://medium.com/@victor_wang/build-your-cells-in-a-way-of-lego-fbf6a1133bb1#.eh7fkk93s)

<br />
<br />


##Animations By Emails

[Transitions with CoreImage](http://www.ios-animations-by-emails.com/posts/2015-may)

[Fun with Gradients and Masks](http://ios-animations-by-emails.com/posts/2015-july)

[fireworks](http://ios-animations-by-emails.com/posts/2015-november)

[使用CAReplicatorLayer创建动画](http://www.jianshu.com/p/76c588893b19?utm_campaign=hugo&utm_medium=reader_share&utm_content=note&utm_source=weibo)


<br />
<br />

<p><a name="frameworks"></a></p>

<br />
<br />
<br />

--------------------------------------------------------------

<br />
<br />
<br />

# Frameworks


[JLStickerTextView](https://github.com/luiyezheng/JLStickerTextView) --- 在图片上，添加文字，支持旋转，平移，缩放，编辑图的功能

[RaceMe](https://github.com/enochng1/RaceMe) --- 一个功能完整的运动追踪 App，做的很漂亮

[Erik](https://github.com/phimage/Erik) ---  WebKit的分装，较好支持CSS和JS

[Heimdall](https://github.com/henrinormak/Heimdall) --- 简单易用的加、解密安全框架（AES/RSA）库及示例

[KeychainAccess](https://github.com/kishikawakatsumi/KeychainAccess)

[ios-charts](https://github.com/danielgindi/ios-charts) ---   表格

[顶部标题滚动条](http://www.jianshu.com/p/b45655e23a42)

[XXYAudioEngine](https://github.com/xxycode/XXYAudioEngine) --- 基于 NSURLSession 和 AVAudioPlayer 封装的一个播放在线音乐的工具，可以把音乐下载至本地，支持清除缓存

[iOS Material Design 效果的下拉菜单](https://github.com/AssistoLab/DropDown)

[XXYAudioEngine](https://github.com/xxycode/XXYAudioEngine) --- 基于NSURLSession和AVAudioPlayer封装的一个播放在线音乐的工具，可以把音乐下载至本地，支持清除缓存

[VirwMonitor 点击测量视图位置、大小、背景、字体大小等信息](https://github.com/daisuke0131/ViewMonitor)

[Watchdog](https://github.com/wojteklu/Watchdog) --- iOS 实时卡顿监控

[netfox：iOS 网络调试工具](https://github.com/kasketis/netfox) --- 配置只需一行代码，可以调试包括第三方库在内的网络请求

[Filterpedia](https://github.com/FlexMonkey/Filterpedia) --- 基于 Core Image 框架，完整、强大的图片滤镜类库

[高性能 AttributedLabel](https://github.com/KyoheiG3/AttributedLabel)

[弹窗](https://github.com/IcaliaLabs/Presentr)

[KeyboardManager](https://github.com/hackiftekhar/IQKeyboardManager)

[tabBar 上的角标 Category 写法，无需继承子类](https://github.com/DeveloperLx/LxTabBadgePoint)

[Export your Live Photos as animated GIFs](https://github.com/neonichu/LiveGIFs)

[实现类似微信的 webView 导航效果，左滑返回上个网页](https://github.com/Roxasora/RxWebViewController)

[一个高性能的 APNG 库](https://github.com/onevcat/APNGKit)

[XAnimatedImage](https://github.com/khaledmtaha/XAnimatedImage) --- 一个高性能的 GIF 动画引擎基于 FLAnimatedImage

[Hodor](https://github.com/Aufree/Hodor?hmsr=toutiao.io&utm_medium=toutiao.io&utm_source=toutiao.io) --- 应用内直接更改应用语言而无需退出应用, 类似微信

[AudioBot](https://github.com/nixzhu/AudioBot) --- 录音Demo，录音时，同时也可以录入正在播放的音乐

[AudioKit](https://github.com/audiokit/AudioKit) --- 一个很棒的录音库，文档教程齐全，已有一个上架[App](http://matthewfecher.com/app-developement/swift-synth/)

[多级下拉菜单](https://github.com/Shannon-s-Dreamland/DropdownMenu)

[正则库](https://github.com/VerbalExpressions/SwiftVerbalExpressions)

[面向协议编程单元测试 Mock 框架](http://www.weibo.com/mygroups?gid=3771498714262571&wvr=6&leftnav=1)

[RatioAutoLayout](https://github.com/GJGroup/GJRatioAutoLayout) --- Autolayout 下，只需一个开关，即可让当前 view 进行等比缩放

[Macup](https://github.com/lra/mackup) --- dotfiles 管理，通过git，dropbox来进行备份，恢复自定义设置的文件

[支持对视图进行局部高亮](https://github.com/yukiasai/Gecco) --- 快速创建产品的新手指导界面

[阅读器](https://github.com/FolioReader/FolioReaderKit?utm_campaign=This%2BWeek%2Bin%2BSwift&utm_medium=web&utm_source=This_Week_in_Swift_71)

[PLCameraStreamingKit, 快速构建一款类似 Meerkat 或 Periscope 的手机直播应用](https://github.com/pili-engineering/PLCameraStreamingKit) --- 👍👍👍

[ZLPhotoBrowser](https://github.com/longitachi/ZLPhotoBrowser) --- 相册照片多选框架，支持拍照、预览快速多选

[Fusuma](https://github.com/ytakzk/Fusuma?utm_campaign=This%2BWeek%2Bin%2BSwift&utm_medium=email&utm_source=This_Week_in_Swift_79) --- Instagram-like photo browser and a camera feature 👍👍👍

[Duration](https://github.com/SwiftStudies/Duration) --- 多种方法测量代码片段执行时间工具类库。为了测量准确性，还提供多计次重复执行求平均方案

### 聊天界面
[NextGrowingTextView](https://github.com/muukii/NextGrowingTextView)

[图片选择器](https://github.com/cbangchen/CBImagePicker)

[ImagePicker is an all-in-one camera solution for your iOS app](https://github.com/hyperoslo/ImagePicker)

[图片裁切](https://github.com/TimOliver/TOCropViewController)

[TextAttributes](https://github.com/delba/TextAttributes?utm_source=newsletter&utm_medium=email&utm_campaign=week_127_is_ready)

[当 App 崩溃的时候，它会帮你自动截屏并打开邮件反馈页面，提交崩溃页面日志以及截屏](https://github.com/Lickability/PinpointKit)

[顶部标题滚动条](http://www.jianshu.com/p/b45655e23a42)

[Preheat](https://github.com/kean/Preheat) --- UITableView and UICollectionView 预加载

[Echarts](https://github.com/Pluto-Y/iOS-Echarts) --- 强大的图表库

[StatusProvider](https://github.com/mariohahn/StatusProvider) --- 空白状态时的占位

[一个全开源的纯 OC 实现的 RTMP 推流 SDK，支持 AAC、H264、美颜滤镜、AMF编解码](https://github.com/liuf1986/LFRtmp)

[弹出滑动选择器](https://github.com/hsylife/SwiftyPickerPopover)

[在屏幕上显示 FPS，CPU 使用率](https://github.com/dani-gavrilov/GDPerformanceView)

[Timeline Cell](https://github.com/kf99916/TimelineTableViewCell)
	
[FRDIntent](https://github.com/douban/FRDIntent) --- 一个处理 View Controller 内部和外部调用的开源库。可用于解除 View Controller 间耦合

[圆形进度条](https://github.com/HamzaGhazouani/HGCircularSlider)

[通知 Bar 效果](https://github.com/qiuncheng/NoticeBar)

[把 SVG 动画集成到下拉刷新中](https://github.com/strongself/MRefresh)

[图片选择器(带转场)](https://github.com/CosmicMind/Motion)

[漂亮的日历组件](https://github.com/patchthecode/JTAppleCalendar)

[把 SVG 动画集成到下拉刷新](https://github.com/strongself/MRefresh)

[模糊背景的弹出窗口](https://github.com/MarioIannotta/MIBlurPopup)

[iOS 网络变更通知](https://github.com/ezefranca/EFInternetIndicator)
	
[细腻的图片 Shadow 效果](https://github.com/PierrePerrin/PPMusicImageShadow)

[可以点击和显示更多的 TextView](https://github.com/jhurray/SelectableTextView)

[圆点翻页 PageControl](https://github.com/ChiliLabs/CHIPageControl)


<br />
<br />

<p><a name="cheatsheet"></a></p>

<br />
<br />
<br />

--------------------------------------------------------------

<br />
<br />
<br />

# CheatSheet

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
git pull origin master // --allow-unrelated-histories
cd ..
rake deploy
```

<br />


##### Can't ignore UserInterfaceState.xcuserstate

```
git rm --cached ProjectFolder.xcodeproj/project.xcworkspace/xcuserdata/myUserName.xcuserdatad/UserInterfaceState.xcuserstate
git commit -m "Removed file that shouldn't be tracked"
```

##### Fastlane Connection reset by peer

```
# brew uninstall ruby # not using homebrew's ruby

rm -rf ~/.gem
rm -rf ~/.gems
rm -rf ~/.rbenv
brew install rbenv ruby-built

# Add rbenv to bash so that it loads every time you open a terminal

echo 'if which rbenv > /dev/null; then eval "$(rbenv init -)"; fi' >> ~/.bash_profile
source ~/.bash_profile

# Install Ruby

rbenv install 2.3.1
rbenv global 2.3.1

```

```
# close teminal and re-open it

ruby -v # should output 2.3.1 now
sudo chown -R $(whoami): ~/.rbenv # important!!!
gem update --system
gem install fastlane # this will install a lot of gems, since we are not using the old 2.0.0 gems anymore.

```

Then you may need to modify your .bash_profile or .zshrc if you use zsh.
Add these 2 lines:

```
vim .zshrc
export PATH="$HOME/.rbenv/bin:$PATH"
eval "$(rbenv init -)"
```
> https://github.com/fastlane/fastlane/issues/6553





<br />
<br />

<p><a name="reading"></a></p>

<br />
<br />
<br />

--------------------------------------------------------------

<br />
<br />
<br />

#Reading

[Coordinators with Storyboards](http://www.apokrupto.com/blog-1/2016/3/17/coordinators-with)

## WebView
[WKWebView 那些坑](https://zhuanlan.zhihu.com/p/24990222)
[WKWebViewWithURLProtocol](https://github.com/WildDylan/WKWebViewWithURLProtocol)

## AVFoundation

[iOS微信小视频优化心得](https://mp.weixin.qq.com/s?__biz=MzAwNDY1ODY2OQ==&mid=207686973&idx=1&sn=1883a6c9fa0462dd5596b8890b6fccf6&scene=1&srcid=bquF9P7agLaSN3IU0FFr&key=18e81ac7415f67c4a63984a53bc911c050fb54e0890e4efac2d4d4f4b41ff63ff9b132efe16b0bcd17b23221ffadf5bb&ascene=0)

[VideoCaptureDemo](https://github.com/adriaan/VideoCaptureDemo) --- objc.io issue 23

[iOS8 Core Image In Swift：视频实时滤镜](http://blog.csdn.net/zhangao0086/article/details/39433519)

[AVReaderWriter](https://github.com/robovm/apple-ios-samples/tree/master/AVReaderWriterOfflineAudioVideoProcessing) --- Apple Demo

[SDAVAssetExportSession](https://github.com/rs/SDAVAssetExportSession)

[AVFoundation(一):基础知识](http://www.jianshu.com/p/485e946f80b4)

[AVFoundation Programming Guide](https://developer.apple.com/library/mac/documentation/AudioVideo/Conceptual/AVFoundationPG/Articles/05_Export.html#//apple_ref/doc/uid/TP40010188-CH9-SW1)

[断点拍摄](http://stackoverflow.com/questions/14905892/pause-resume-video-capture-for-same-file-with-avfoundation-in-ios#answer-14986631)

##Ask and Answer

[圆形和正方形之间的动画](http://stackoverflow.com/questions/24788283/ios-animate-morph-shape-from-circle-to-square)

[使用Autolayout实现UITableView的Cell动态布局和高度动态改变](http://codingobjc.com/blog/2014/10/15/shi-yong-autolayoutshi-xian-uitableviewde-celldong-tai-bu-ju-he-ke-bian-xing-gao/)

[UITableView点击两次才能Present](http://stackoverflow.com/questions/20320591/uitableview-and-presentviewcontroller-takes-2-clicks-to-display)

[iOS推送获取不到设备token：未找到应用程序的apsenvironment的权利字符串](http://www.it165.net/pro/html/201503/36130.html)

[假设有一个巨大（包含很多属性的）的结构体，然后实现 “==” 操作就会很麻烦，因为要每个属性都比较一遍才行](http://stackoverflow.com/questions/35023904/is-there-way-to-define-compare-function-automatically-for-struct-in-swi) --- 👍👍👍


##多 Target
[XCode: 7 Steps to Easily Switch Between Multiple Environments](http://www.teratotech.com/blog/xcode-7-steps-to-easily-switch-between-multiple-environments/)
[iOS multi-environment configuration](http://appfoundry.be/blog/2014/07/04/Xcode-Env-Configuration/)

[Change your API endpoint/environment using Xcode Configurations in Swift](https://medium.com/@danielgalasko/change-your-api-endpoint-environment-using-xcode-configurations-in-swift-c1ad2722200e#.9ghrll66j)

[环境变量配置(Debug & Release)](http://blog.startry.com/2015/07/24/iOS_EnvWithXcconfig/)

[CocoaPods and Lookback: The Build Configuration Journey](https://lookback.io/blog/cocoapods-by-configuration)

##包罗万象

[Extending Swift generic types](http://www.marisibrothers.com/2016/03/extending-swift-generic-types.html) --- extension 时指定 Element 类型

[深入探究Swift数组背后的协议、方法、拓展](http://www.jianshu.com/p/b93c07fa8c3d)

[Creating a Cheap Protocol-Oriented Copy of SequenceType](http://christiantietze.de/posts/2016/03/custom-enumerated-sequence/?utm_campaign=This%2BWeek%2Bin%2BSwift&utm_medium=web&utm_source=This_Week_in_Swift_78)

[iOS 视频边下边播--缓存播放数据流](http://www.jianshu.com/p/990ee3db0563) --- 👍👍👍

[在 iOS 开发中如何优雅地进行图片缩放](http://www.jianshu.com/p/af2d471f7b9c?utm_campaign=hugo&utm_medium=reader_share&utm_content=note&utm_source=weibo)

[最好的 Bezier 曲线教程](http://pomax.github.io/bezierinfo/)

[iOS 系统搜索集成](https://realm.io/cn/news/jack-nutting-search-api-ios/) --- Realm

[Swift 中的iOS设计模式（一）](http://www.devtalking.com/articles/ios-design-pattern-in-swift-1/)

[Swift 编程风格指南](http://swiftist.org/topics/165)

[Reader Submissions - New Year's 2016](http://nshipster.com/new-years-2016) --- 干货多 🍺

[Showing TODO as a warning in a Swift Xcode project](http://stackoverflow.com/questions/24183812/swift-warning-equivalent) --- 以⚠️的形式显示TODO、FIXME

[Lazy Initialization with Swift](http://mikebuss.com/2014/06/22/lazy-initialization-swift/)

[Swift90Days - 蛋疼的初始化过程](https://github.com/callmewhy/Swift90Days/blob/master/Day11-initialization.md)

[Error Handling in Swift: Might and Magic](http://nomothetis.svbtle.com/error-handling-in-swift)

[Swift 2.0: Why Guard is Better than If](http://natashatherobot.com/swift-guard-better-than-if/)

[2015 WWDC Best Practices](https://github.com/100mango/zen/blob/master/2015%20WWDC%20总结/2015%20WWDC%20Best%20Practices.md)

[LAZY 修饰符和 LAZY 方法](http://swifter.tips/lazy/)

[Lazy Properties in Structs](http://oleb.net/blog/2015/12/lazy-properties-in-structs-swift/)

[True Lazy Sequences](http://www.obqo.de/blog/2015/11/25/true-lazy-sequences/)

[Swift 2.0 数组遍历](http://swift.gg/2015/09/25/ask-erica-how-do-i-loop-from-non-zero-n-swiftlang/)

[Swift Optionals, Functional Programming, and You](http://www.mokacoding.com/blog/demistifying-swift-functor/) --- 函数式编程

[用同一个工程创建两个不同版本的应用](http://www.cocoachina.com/ios/20150916/13324.html) --- 如果同一个应用, 需要做一个带广告Lite版本, 一个不带广告的Pro版本. 那么问题来了, 该如何优雅的去实现呢？

[Presentation Controllers and Adaptive Presentations](https://pspdfkit.com/blog/2015/presentation-controllers/) --- 👍

[如何让 UIVisualEffectView 上面显示可读的文字](https://www.omnigroup.com/developer/how-to-make-text-in-a-uivisualeffectview-readable-on-any-background)

[CoreText使用教程](http://www.zoomfeng.com/blog/coretextshi-yong-jiao-cheng-%5B%3F%5D.html) --- 五篇

[理解 HTTPS 的工作原理](http://www.codeceo.com/article/https-worker.html)

[如何使用 iOS 9 App 瘦身功能](http://swift.gg/2016/01/07/app-thinning-appcoda/)

[64-bit Tips](http://blog.sunnyxx.com/2014/12/20/64-bit-tips/)

[黑幕背后的Autorelease](http://blog.sunnyxx.com/2014/10/15/behind-autorelease/)

[Objective-C内存管理如何理解Autorelease](http://mobile.51cto.com/iphone-284112.htm)

[UINavigationController和View Controller-based状态栏风格](http://cocoa.venj.me/blog/view-controller-based-status-bar-style-and-uinavigationcontroller/#comment-1501032470) ------ 指定状态栏风格

[Variable Argument Lists](http://gracelancy.com/blog/2014/05/05/variable-argument-lists/) ------ 可变参数函数的定义

[100个iOS开发/设计面试题汇总](http://www.imooc.com/wenda/detail/244497)

[使用Autolayout实现UITableView的Cell动态布局和高度动态改变](http://codingobjc.com/blog/2014/10/15/shi-yong-autolayoutshi-xian-uitableviewde-celldong-tai-bu-ju-he-ke-bian-xing-gao/)

[常用 Git 命令清单](http://www.ruanyifeng.com/blog/2015/12/git-cheat-sheet.html)

[ 手写识别组建 A Handwriting Recognition Component for iOS ](http://flexmonkey.blogspot.co.nz/2015/12/scribe-handwriting-recognition.html)

[ 加密解密介绍 ](<http://www.jianshu.com/p/98610bdc9bd6>)

[如何防止客户端被破解](http://tanqisen.github.io/blog/2014/06/06/how-to-prevent-app-crack/)

[动态修改UINavigationBar的背景色](http://tech.glowing.com/cn/change-uinavigationbar-backgroundcolor-dynamically/)

[iOS开发中用户密码应该保存在哪里](http://www.jianshu.com/p/4af3b8179136)

[由 App 的启动说起](http://oncenote.com/2015/06/01/How-App-Launch/) --- 👍

[State Machine for Layout Constraints](https://medium.com/@pcperini/a-protocol-oriented-state-machine-for-layout-constraints-2c6c94bbd844#.g92o77vfm) --- 通过协议实现 Layout 的状态机 👍👍👍

[Asynchronous error handling](http://alisoftware.github.io/swift/async/error/2016/02/06/async-errors/)


[Extending Swift generic types](http://www.marisibrothers.com/2016/03/extending-swift-generic-types.html) --- 为 extension 指定类型 👍👍👍

[Adapting Auto Layout Without Interface Builder](http://useyourloaf.com/blog/adapting-auto-layout-without-interface-builder/?utm_source=newsletter&utm_medium=email&utm_campaign=week_125_is_ready)

[手势签名](https://github.com/ipraba/EPSignature?utm_campaign=This%2BWeek%2Bin%2BSwift&utm_medium=email&utm_source=This_Week_in_Swift_89)

[iOS 程序 main 函数之前发生了什么](http://blog.sunnyxx.com/2014/08/30/objc-pre-main/)

#####Enum

[Swift 中枚举高级用法及实践](http://swift.gg/2015/11/20/advanced-practical-enum-examples/)

[用枚举作 Options](http://ericasadun.com/2015/10/19/sets-vs-dictionaries-smackdown-in-swiftlang/?utm_campaign=Swift%252BSandbox&utm_medium=email&utm_source=Swift_Sandbox_12)

[用枚举作 Rest API](http://swift.gg/2015/11/20/advanced-practical-enum-examples/#API_端点)


#####Protocol
[Protocol-Oriented Segue Identifiers in Swift](http://natashatherobot.com/protocol-oriented-segue-identifiers-swift/)

[Mixins 比继承更好](http://swift.gg/2015/12/15/mixins-over-inheritance/) --- 面向协议编程

[Protocol-Oriented-Networking in Swift](https://www.natashatherobot.com/protocol-oriented-networking-in-swift/) --- 👍👍👍

#### MapKit
[Add annotations and Polyline to MapView in Swift](http://rshankar.com/how-to-add-mapview-annotation-and-draw-polyline-in-swift/)

[怎么样创建一个像RunKeeper一样的App（一）](http://www.jianshu.com/p/9d998307dc21)

[How To Make an App Like RunKeeper with Swift: Part 2](https://www.raywenderlich.com/97945/make-app-like-runkeeper-swift-part-2)

[后台定位上传的代码实践](http://adad184.com/2015/07/22/how-to-deal-with-background-location-update/)

[一次对MKMapView的性能优化](http://adad184.com/2015/07/13/improve-performance-with-mkmapview/)

####翻译
[Programming iOS9 翻译](http://wdxtub.com/2015/12/22/programming-ios9-translation-1/)


##优化

[iOS 高性能异构滚动视图构建方案 —— LazyScrollView](http://pingguohe.net/2016/01/31/lazyscroll.html?hmsr=toutiao.io&utm_medium=toutiao.io&utm_source=toutiao.io)

[Build Time Analyzer for Xcode](https://github.com/RobertGummesson/BuildTimeAnalyzer-for-Xcode) --- 查看 Swift 文件编译的时间(插件)

[Profiling your Swift compilation times](http://irace.me/swift-profiling/?utm_campaign=iOS%2BDev%2BWeekly&utm_medium=email&utm_source=iOS_Dev_Weekly_Issue_234) --- 查看 Swift 文件编译的时间 👍👍👍👍👍👍

[猿题库 iOS 客户端架构设计](http://mp.weixin.qq.com/s?__biz=MjM5NTIyNTUyMQ==&mid=444322139&idx=1&sn=c7bef4d439f46ee539aa76d612023d43&scene=0#wechat_redirect) --- MVVM without Binding with DataController

[UICollectionView的数据预加载及图片加载逻辑的优化](http://blog.vars.me/blog/2015/04/26/UICollectionView-Optimizing/)

[iOS图形性能进阶与测试](https://github.com/100mango/zen/blob/master/WWDC%E5%BF%83%E5%BE%97%EF%BC%9AAdvanced%20Graphics%20and%20Animations%20for%20iOS%20Apps/Advanced%20Graphics%20and%20Animations%20for%20iOS%20Apps.md)

[iOS图片加载速度极限优化—FastImageCache解析](http://blog.cnbang.net/tech/2578/)

[iOS可执行文件瘦身方法](http://blog.cnbang.net/tech/2544/)

[iOS生产力之小工具合集](http://www.jianshu.com/p/0d972f45da86?utm_campaign=maleskine&utm_content=note&utm_medium=pc_author_hots&utm_source=recommendation)
[iOS性能优化](http://www.jianshu.com/p/9e1f0b44935c)

[10个加速Table Views开发的建议](http://ios.jobbole.com/83058/)

[使用 Swift 和 Objective-C 执行 iOS 内存管理的 7 个简单技巧](http://www.ibm.com/developerworks/cn/mobile/mo-ios-memory/)

[Perfect smooth scrolling in UITableViews](http://southpeak.github.io/blog/2015/12/20/perfect-smooth-scrolling-in-uitableviews/)

[Typed, yet Flexible Table View Controller](http://holko.pl/2016/01/05/typed-table-view-controller/) --- TableViewController 代码重构

### 缓存

[(慕课网)imooc iPhone3.3 接口数据缓存](http://www.jianshu.com/p/8a4dc775c051) --- 👍👍👍

[使用两行代码就能完成80%的缓存需求](https://github.com/ChenYilong/ParseSourceCodeStudy/blob/master/02_Parse的网络缓存与离线存储/iOS网络缓存扫盲篇.md) --- 👍👍👍

##底层
[iOS 事件处理机制与图像渲染过程](https://mp.weixin.qq.com/s?__biz=MzAwNDY1ODY2OQ==&mid=400417748&idx=1&sn=0c5f6747dd192c5a0eea32bb4650c160&scene=1&srcid=1119iAGp9HUSGEMTBjUSM7L0&key=d72a47206eca0ea9f5fd845c39ebee1715577b3db18f002c19b3747d6f2525f50d9ef530d825ee227e428e6dc5bd58a3&ascene=0&uin=MjY5MzMxNTMwMQ%3D%3D)

[Swift 中的弱引用](http://swift.gg/2015/12/28/friday-qa-2015-12-11-swift-weak-references/) --- 👍👍👍


##内存管理
[iOS开发之ARC](http://www.oschina.net/translate/automatic-reference-counting-on-ios)

[手把手教你ARC——iOS/Mac开发ARC入门和使用](http://onevcat.com/2012/06/arc-hand-by-hand/)

[IOS 5编程 内存管理 ARC技术概述](http://blog.csdn.net/nicktang/article/details/6792972)

[iPhone开发之深入浅出 (1) — ARC是什么](http://www.yifeiyang.net/development-of-the-iphone-simply-1/)


##线程
[Cocoa深入学习:NSOperationQueue、NSRunLoop和线程安全](http://blog.cnbluebox.com/blog/2014/07/01/cocoashen-ru-xue-xi-nsoperationqueuehe-nsoperationyuan-li-he-shi-yong/)

[iOS开发系列--并行开发其实很容易](http://www.cnblogs.com/kenshincui/p/3983982.html)

[iOS同步对象性能对比](http://ksnowlv.github.io/blog/2014/09/07/ios-tong-bu-suo-xing-neng-dui-bi/)

[iOS开发中设计并发任务技术与注意事项](http://geek.csdn.net/news/detail/60236)


##工具
[Bmob移动后端云服务平台](http://www.bmob.cn)

[和应用服务器、存储服务器说再见](https://leancloud.cn/docs/rest_api.html)

[Bug 提交](https://www.buglife.com)

##插件
[VWInstantRun](https://github.com/wangshengjia/VWInstantRun) --- 直接在Xcode中编译执行选中的Swift代码，打印输出到console  👍


##CoreData
[Core Data: 非标准数据类型总结](http://www.jianshu.com/p/5a84008307ad)

##硬件
[iOS的蓝牙连接、数据接收及发送](http://blog.csdn.net/dolacmeng/article/details/46457487)


##CocoaPods

[Cocoapods系列教程(三)——私有库管理和模块化管理](http://www.pluto-y.com/cocoapod-private-pods-and-module-manager/)

[用CocoaPods做iOS程序的依赖管理](http://blog.devtang.com/blog/2014/05/25/use-cocoapod-to-manage-ios-lib-dependency/)

[制作自己的CocoaPods Spec](http://gracelancy.com/blog/2013/08/11/make-your-own-cocoapods-spec/)

[CocoaPods进阶：本地包管理](http://www.iwangke.me/2013/04/18/advanced-cocoapods/)

[Setup a mirror of CocoaPods/Specs](http://lexrus.com/setup-a-mirror-of-cocoapods-specs.html)

[CocoaPods建立私有仓库](http://blog.csdn.net/agdsdl/article/details/45218987#0-tsina-1-51027-397232819ff9a47a7b7e80a40613cfe1)




