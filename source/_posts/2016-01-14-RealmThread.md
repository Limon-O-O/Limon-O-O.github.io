---
layout: post
title: "Realm Thread 浅析"
date: 2016-01-14 02:27:14 +0800
comments: true
categories: Realm
---


两个同一：同一个 realm、同一个线程，两者缺一不可。

打开 App，首先从 Realm 中获取首页的数据，此时在 **主线程** 调用 `-fetchArticlesFromRealmByType(.Home)`，此方法返回一个元组： `result: (articles: [Article], articleObjects: [ArticleObject])`，前者是模型，后者是 Realm 的 `Object`。

然后再进行网络请求，网络请求完之后，**主线程** 更新 UI，并保存数据最新的数据和删除旧的数据（[ArticleObject]）。
保存数据直接可以实例化一个 Realm，所以不需要考虑 **两个同一**。

重点讲删除数据的两个同一：

```swift
func fetchArticlesFromRealmByType(articleType: ArticleType, result: (articles: [Article], articleObjects: [ArticleObject]) -> Void) {

  // 省略部分代码...

  // 成功从本地数据库中获取数据之后，在 **主线程*** 回调，在外面更新 UI，并持有 articleObjects，待网络请求完，删除此 articleObjects
  let articleRealmObjects = realm.objects(ArticleObject)
  result(articles: articles, articleObjects: articleRealmObjects)

  // 注意，Object 不能切换线程，意思就是说，你如果在 Background 线程得到 articleRealmObjects，不能再切换到主线程回调出去，推荐使用通知。
  Queue.Background.execute() {
    let articleRealmObjects = realm.objects(ArticleObject)
    Queue.Main.execute() {
      result(articles: articles, articleObjects: articleRealmObjects)
    }
  }

}

```

```swift
func deleteObject(articleObjects: [ArticleObject]) {
  guard let articleObjects = articleObjects else {
    return
  }
  let _ = try? articleObjects.first?.realm?.write() {
    articleObjects.first?.realm?.delete(articleObjects)
  }
}
```

### 同一个线程

待网络请求完后，想要删除旧的 `articleObjects`，应该在 `articleObjects` 同一线程内删除，在此例子中，`articleObjects` 是从 `Queue.Main` 中得到，那删除时也要在此线程删除，代码如下：

```
Queue.Main.execute() {
  print("delete action on thread: \(NSThread.currentThread())")
  deleteObject(self.articleObjects)
}
```

如果打印出来的内存地址和在 `-fetchArticlesFromRealmByType(.Home)` 打印的不同，此时就会 Crash: `Realm accessed from incorrect thread.`

> 如果是在串行队列中，需要持有此队列，就算 `-dispatch_queue_create(key, _)` 中的 key 每次创建都是同一个 key，也不能保证所有串行任务都在同一个队列对象中。

### 同一个 realm
可以直接从 `ArticleObject` 获取 `realm`，前提是数组内所有的 Object 都是在同一个 `realm` 内。(还没遇到过，一个数组内放不同 realm 的 Object 这样变态的需求 -.- )

```
let _ = try? articleObjects.first?.realm?.write() {
  articleObjects.first?.realm?.delete(articleObjects)
}
```

这里值得注意的是，删除了此 `articleObject`之后，如果你还访问了此对象的自定义属性，就会 Crash: `Object has been deleted or invalidated.`

例如：

```
let _ = try? articleObject.realm?.write() {
  articleObject.realm?.delete(objects)
  print(articleObject.title)
}
```

因为 `articleObject` 已经无效了，因此稳妥点，在访问前先判断此 `Object` 的有效性。



```
func deleteObject(articleObjects: [ArticleObject]) {
  guard let articleObject = articleObjects.first where !articleObject.invalidated && !(articleObject.realm?.isEmpty ?? true) else {
    return
  }
  let _ = try? articleObject.realm?.write() {
    articleObject.realm?.delete(objects)
  }
}
```


在 [NSHipster](http://nshipster.com/new-years-2016/#swiftier-gcd) 找到的 GCD 封装

```
protocol ExcutableQueue {
    var queue: dispatch_queue_t { get }
}

extension ExcutableQueue {
    func execute(closure: () -> Void) {
        dispatch_async(queue, closure)
    }
}

enum Queue: ExcutableQueue {
    case Main
    case UserInteractive
    case UserInitiated
    case Utility
    case Background

    var queue: dispatch_queue_t {
        switch self {
        case .Main:
            return dispatch_get_main_queue()
        case .UserInteractive:
            return dispatch_get_global_queue(QOS_CLASS_USER_INTERACTIVE, 0)
        case .UserInitiated:
            return dispatch_get_global_queue(QOS_CLASS_USER_INITIATED, 0)
        case .Utility:
            return dispatch_get_global_queue(QOS_CLASS_UTILITY, 0)
        case .Background:
            return dispatch_get_global_queue(QOS_CLASS_BACKGROUND, 0)
        }
    }
}
```
