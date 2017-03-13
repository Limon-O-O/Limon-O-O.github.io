---
layout: post
title: "玩转Mantle"
date: 2014-4-23 10:05:15 +0800
comments: false
categories: Mantle
---


## MTLModel

MTLModel provides an easy way to map NSDictionary objects to Objective-C classes and vice-versa.

假设返回的JSON为：

```
 {
  "uid": 1,
  "title": "Found a bug",
  "url": "https://api.github.com/repos/octocat/Hello-World/issues/1347",
  "number": 1347,
  "state": "open",
  "user": {
    "login": "octocat",
    "id": 1
  },
  "assignee": {
    "login": "octocat",
    "id": 1,
    "type": "User"
  }
  "created_at": "2011-04-22T13:33:48Z",
  "awesome": true  
}
```

继承`MTLModel`遵守`<MTLJSONSerializing>`协议

```
#import <Mantle.h>

typedef enum : NSUInteger {
    GHIssueStateOpen,
    GHIssueStateClosed
} GHIssueState;

@interface GHIssue : MTLModel<MTLJSONSerializing>
@property (nonatomic, copy, readonly) NSNumber *uid;
@property (nonatomic, copy, readonly) NSURL *URL;
@property (nonatomic, copy, readonly) NSNumber *number;
@property (nonatomic, assign, readonly) GHIssueState state;
@property (nonatomic, copy, readonly) NSString *reporterLogin;
@property (nonatomic, copy, readonly) NSDate *updatedAt;
@property (nonatomic, strong, readonly) GHUser *assignee;
@property(nonatomic, getter=isAwesome) BOOL awesome;
@property (nonatomic, copy) NSString *title;


```
`<MTLJSONSerializing>`协议告诉Mantle序列化该对象如何从JSON映射到Objective-C的属性。

###NSValueTransformer

在.m文件，实现`<MTLJSONSerializing>`的协议`@required`的方法`+ (NSDictionary *)JSONKeyPathsByPropertyKey`
它指明了如何把json的keypath和Model的属性对应起来



```
+ (NSDictionary *)JSONKeyPathsByPropertyKey {
	// properties defined in header  :  key in JSON Dictionary，本地字段在前，服务端字段在后
    return @{
        @"URL": @"url",
        @"HTMLURL": @"html_url",
        @"reporterLogin": @"user.login",
        @"assignee": @"assignee",
        @"updatedAt": @"updated_at"
    };
}
```

因为如果属性名和JSON的键名一致时，可以省略不写映射，例如title

以上分了几种情况：
Mantle使用主要看属性property中的类型，主要分几种：

1. NSNumber、NSString....
2. NSArray
3. NSURL
4. NSDate
5. 枚举
6. 模型 	
7. BOOL


先介绍一个方法：`+ (NSValueTransformer *)JSONTransformerForKey:(NSString *)key`

```
// Specifies how to convert a JSON value to the given property key. If
// reversible, the transformer will also be used to convert the property value
// back to JSON.
//
// If the receiver implements a `+<key>JSONTransformer` method, MTLJSONAdapter
// will use the result of that method instead.
//
// Returns a value transformer, or nil if no transformation should be performed.
+ (NSValueTransformer *)JSONTransformerForKey:(NSString *)key;

```
以下方法的命名都要遵从：`SEL selector = MTLSelectorWithKeyPattern(key, "JSONTransformer");`

1、一般像tittle，`@"title": @"title"`不用写这行也自动转化。我个人认为从服务器返回的所有数据都是字符串类型。所有理论上，uid还要用`NSValueTransformer`转化，如下：

	+ (NSValueTransformer *)uidJSONTransformer{
    	return [MTLValueTransformer reversibleTransformerWithForwardBlock:^id(NSString *string) {
        return @([string integerValue]);
    } reverseBlock:^id(NSNumber *number) {
        return [number stringValue];
    }];
}


但是，如果你的uid属性类型定义为NSNumber，还是不需用上面的方法，应该是Mantle自己处理了。

>如果你定义的uid类型为NSUInteger，还是需要uidJSONTransformer

还有一种情况是`@"reporterLogin": @"user.login"`
   login是在JSON内层的元素，需要用下面这个方法
   
   ```
   + (NSValueTransformer *)reporterLoginJSONTransformer {
    	return [MTLValueTransformer reversibleTransformerWithForwardBlock:^(NSArray *values) {
        	return [values firstObject];
    	} reverseBlock:^(NSString *str) {
        	return @[str];
    	}];
	}
   ```

2、NSArray装模型

```
{
  name: "Bob",
  cars: [
    { make: "ford", year: "1972" },
    { make: "mazda", year: "2000" }
  ],
}
```

```
@interface CarModel : MTLModel

@property (nonatomic, strong) NSString *make;
@property (nonatomic, strong) NSString *year;

@end

@interface PersonModel : MTLModel

@property (nonatomic, strong) NSString *name;
@property (nonatomic, strong) NSArray *cars; // NSArra元素是Car模型

@end
```
```
+ (NSValueTransformer *)carsJSONTransformer {
    return [NSValueTransformer mtl_JSONArrayTransformerWithModelClass:CarModel.class];
}
```


3、NSURL
	
	+ (NSValueTransformer *)URLJSONTransformer {
    	return [NSValueTransformer valueTransformerForName:MTLURLValueTransformerName];
	}			
	
4、NSDate


```

+ (NSDateFormatter *)dateFormatter {
    NSDateFormatter *dateFormatter = [[NSDateFormatter alloc] init];
    dateFormatter.locale = [[NSLocale alloc] initWithLocaleIdentifier:@"en_US_POSIX"];
    dateFormatter.dateFormat = @"yyyy-MM-dd'T'HH:mm:ss'Z'";
    return dateFormatter;
}

+ (NSValueTransformer *)updatedAtJSONTransformer { // 把"updated_at": "2011-04-22T13:33:48Z"传给ForwardBlock
    return [MTLValueTransformer reversibleTransformerWithForwardBlock:^(NSString *str) {
        return [self.dateFormatter dateFromString:str]; // 把dateFormatter再传给reverseBlock
    } reverseBlock:^(NSDate *date) {
        return [self.dateFormatter stringFromDate:date];
    }]
}


```


如果JSON的数据格式是Unix时间戳，"date": 1418202490,


```
+ (NSValueTransformer *)dateJSONTransformer {
    return [MTLValueTransformer reversibleTransformerWithForwardBlock:^(NSString *str) {
        return [NSDate dateWithTimeIntervalSince1970:str.floatValue];
    } reverseBlock:^(NSDate *date) {
        return [NSString stringWithFormat:@"%f",[date timeIntervalSince1970]];
    }];
}

```

	
5、枚举

```
+ (NSValueTransformer *)stateJSONTransformer {
    return [NSValueTransformer mtl_valueMappingTransformerWithDictionary:@{
        @"open": @(GHIssueStateOpen),
        @"closed": @(GHIssueStateClosed)
    }];
}
```

6、模型

```
+ (NSValueTransformer *)assigneeJSONTransformer {
    return [NSValueTransformer mtl_JSONDictionaryTransformerWithModelClass:GHUser.class];
}
```

7、 BOOL

```
+ (NSValueTransformer *)awesomeJSONTransformer {
    return [NSValueTransformer valueTransformerForName:MTLBooleanValueTransformerName];
}
```


##空对象处理

在模型.m文件添加`-setNilValueForKey`，Mantle是基于KVC给property赋值的，KVC提供了`-setNilValueForKey`方法，让我们为nil指定一个合理的替代值

```
@interface MTLModel (KTVNullableScalar)
@end
@implementation MTLModel (KTVNullableScalar)
- (void)setNilValueForKey:(NSString *)key {
    [self setValue:@0 forKey:key];  // For NSInteger/CGFloat/BOOL
}
@end
```




##Create model objects from JSON

```
// create NSDictionary from JSON data
NSData JSONData = ... // the JSON response from the API
NSError *error = nil;
NSDictionary *JSONDict = [NSJSONSerialization JSONObjectWithData:JSONData options:0 error:&error];

// create model object from NSDictionary using MTLJSONSerialisation
CATProfile *profile = [MTLJSONAdapter modelOfClass:CATProfile.class fromJSONDictionary:JSONDict error:NULL];
```


##Create JSON from model objects

```
// create NSDictionary from model class using MTLJSONSerialisation
CATProfile *profile = ...
NSDictionary *profileDict = [MTLJSONAdapter JSONDictionaryFromModel:profile];
NSError *error = nil;
// convert NSDictionary to JSON data
NSData *JSONData = [NSJSONSerialization dataWithJSONObject:profileDict options:0 error:&error];
```
>Note：如果不想JSONData含有模型的某个属性，可以在模型JSONKeyPathsByPropertyKey方法中
	return @{ @"Name": NSNull.null };



Serialize 序列化
----------
待续...


优秀文章
----------

[Simplify iOS Models With Mantle – An Intro](http://spin.atomicobject.com/2014/06/23/ios-models-mantle/)

[mantle](http://www.objc.at/mantle)

[iOS的Mantle实战](http://www.cnblogs.com/ipinka/p/4041835.html)

[为什么唱吧iOS 6.0选择了Mantle](http://ke.gitcafe.io/2014/10/13/Why-Changba-iOS-choose-Mantle/)


