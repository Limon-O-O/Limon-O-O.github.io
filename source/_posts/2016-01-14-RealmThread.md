---
layout: post
title: "Realm Thread 浅析"
date: 2016-01-14 02:27:14 +0800
comments: true
categories:
---


两个同一：同一个 realm、同一个线程，两者缺一不可。

打开 App，首先从 Realm 中获取首页的数据，此时在 **主线程** 调用 `-fetchArticlesFromRealmByType(.Home)`，此方法返回一个元组： `result: (articles: [Article], articleObjects: [ArticleObject])`，前者是模型，后者是 Realm 的 `Object`。

然后再进行网络请求，网络请求完之后，**主线程** 刷新数据，并保存数据最新的数据和删除旧的数据（[ArticleObject]）。
保存数据直接可以实例化一个 Realm，所以不需要考虑 **两个同一**。

重点讲删除数据的两个同一：

```swift
func deleteObject(articleObject: ArticleObject) {
  guard let articleObject = articleObject else {
    return
  }
  let _ = try? articleObject.realm?.write() {
    articleObject.realm?.delete(objects)
  }
}
```

### 同一个线程
Queue.Main.execute() {
  print("delete action on thread: \(NSThread.currentThread())")
  deleteAction()
}

如果打印出来的内存地址和在 `-fetchArticlesFromRealmByType(.Home)` 打印的不同，此时就会 Crash: `Realm accessed from incorrect thread.`

> 如果是在串行队列中，需要持有此队列，就算 `-dispatch_queue_create(key, _)` 中的 key 每次创建都是同一个 key，也不能保证所有串行任务都在同一个队列对象中。

### 同一个 realm
可以直接从 `ArticleObject` 获取 `realm`，前提是数组内所有的 Object 都是在同一个 `realm` 内。

```
let _ = try? articleObject.realm?.write() {
  articleObject.realm?.delete(objects)
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



```swift
func deleteObject(articleObject: ArticleObject) {
  guard let articleObject = articleObject where !articleObject.invalidated && !(articleObject.realm?.isEmpty ?? true) else {
    return
  }
  let _ = try? articleObject.realm?.write() {
    articleObject.realm?.delete(objects)
  }
}
```
