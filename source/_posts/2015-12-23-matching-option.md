---
layout: post
title: "Matching Option"
date: 2015-12-23 22:36:43 +0800
comments: true
categories: Producter-Tip
---

```
{
    "articles": [
        {
            "type": "app", // AppSo文章
            "cellHeight": "100"
        },
        {
            "type": "number", // 数独文章
            "cellHeight": "200"
        },
        {
            "type": "mindStore", // MindStore文章
            "cellHeight": "300"
        }
    ]
}
```


假设服务器返回以上的JSON，客户端需要根据文章类型来作不同的布局。

第一时间可能会想到以下的方法来switch：

```Swift
let typeString = "app"
switch typeString {

  case "app":
    print("AppSo Article")

  case "number":
    print("Number Article")

  default:
    break
}
```
<br />

较为优雅的方法是用`enum`来管理类型：

```
enum Occupation: String {
  case AppSo = "app"
  case Number = "number"
}

let typeString = "mindStore"

switch Occupation(rawValue: typeString) {

  case .AppSo?:
    print("AppSo Article")

  case .Number?:
    print("Number Article")

  case nil:
    print("Article?")
}
```

<br />

抽取于：[Matching with Swift's Optional Pattern](http://www.figure.ink/blog/2015/12/6/matching-with-swifts-optional-pattern?utm_campaign=This%2BWeek%2Bin%2BSwift&utm_medium=web&utm_source=This_Week_in_Swift_65)
