---
layout: post
title: "花式自定义下标"
date: 2016-01-05 17:56:42 +0800
comments: true
categories: Producter-Tip
---

# 花式自定义下标

### 一、枚举作 Key

```swift
let dictionaryOne = [
        MyDictionaryKeys.Name.rawValue: "Limon",
        MyDictionaryKeys.Age.rawValue: Int(18)
]
let name = dictionaryOne[.Name]!
let age = dictionaryOne[.Age]!
```

```swift
enum MyDictionaryKeys: String {
    case Name
    case Age
}

extension Dictionary {

    subscript(customKey: MyDictionaryKeys) -> AnyObject? {

        guard let key = customKey.rawValue as? Key else {
            return nil
        }

        for (k, value) in self {
            if key == k {
                return value as? String
            }
        }

        return nil
    }
}

extension NSDictionary {

    subscript(customKey: MyDictionaryKeys) -> AnyObject? {
        return objectForKey(customKey.rawValue)
    }
}
```

<br />



### 二、声明 Key 时，同时定义 Value 的类型

```swift
let userID = dictionaryTwo[.tip_UserIDKey]!
print(userID is Int) // Always true
```

```swift
extension DictionaryKeys {
    static let tip_UserIDKey = DictionaryKey<Int?>("userID")
}
```



```swift

extension NSDictionary {

    subscript(key: DictionaryKey<String?>) -> String? {
        get { return objectForKey(key.value) as? String }
    }

    subscript(key: DictionaryKey<Int?>) -> Int? {
        get { return objectForKey(key.value) as? Int }
    }
}

class DictionaryKeys {
    private init() {}
}

class DictionaryKey<ValueType>: DictionaryKeys {

    let value: String

    init(_ key: String) {
        self.value = key
    }
}
```

<br />


### 三、用枚举获取 Set 的元素

```swift
let qqAccount = accountSet[.QQ]
let weChatAccount = accountSet[.WeChat]

print("样式三、qqAppID: \(qqAccount!.appID)  weChatAppID: \(weChatAccount!.appID)")
```

<br />

源码：[SubscriptSwift](<https://github.com/Limon-catch/SubscriptSwift>)
