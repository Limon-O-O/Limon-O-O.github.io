---
layout: post
title: "#01-字面量语法"
date: 2014-5-26 12:40:19 +0800
comments: true
categories: Effective-OC
---

##字面量数值
普通创建：


	NSNumber *num = [NSNumber numberWithInt:1];


字面量创建


	NSNumber *nub = @1;

	int a = 5;
	float b = 5.32f;
	NSNumber *num = @(a * b);


##字面量数组

普通创建

	NSArray *fruits = [NSArray arrayWithObjects:@"Lemon",@"Apple",nil];
	[fruits objectAtIndex:1]; // Apple

字面量创建

	NSArray *fruits = @[@"Lemon",@"Apple"];
	fruits[1]; // Apple

`fruits[1]`这称为取下标操作(subscripting)

##nil
字面量创建数组，若数组元素对象中有nil，会抛出异常。因为字面量语法实际上是一种语法糖(syntactic sugar)，其效果等于是先创建了一个数组，然后把方括号内的所有对象都加到这个数组。



	NSObject *nilObject = nil;
	NSArray *fruits = @[@"Lemon",nilObject,@"Apple"]; // 崩了

如果不用字面量创建，用`NSArray arrayWithObjects:`则不会抛出异常，只会提前结束。如下例子，遇到nil结束，fruits只有一个元素


	NSObject *nilObject = nil;
	NSArray *fruits = [NSArray arrayWithObjects:@"Lemon",nilObject,@"Apple",nil];

>微妙的差别表明，使用字面量语法更安全。向数组插入nil通常说明程序有错，而通过异常可以更快地发现这个错误。

##字面量字典
值 -> 键：普通创建

	NSDictionary *personData = [NSDictionary dictionaryWithObjectsAndKeys:@"Tom",@"name",[NSNumber numberWithInt:18],@"age", nil];
	[personData objectForKey:@"name"]; // Tom
	

键 -> 值：字面量语法创建	
	
	NSDictionary *personData = @{@"name": @"Tom",@"age":@18};
	personData[@"age"]; // 18


##可变数组和字典
如果数组与字典对象是可变的(mutable)，也可以通过下标修改可变数组或字典的值

	mutableArray[1] = @"dog";
	mutableDictionary[@"name"] = @"Mary";

普通做法：

	[mutableArray replaceObjectAtIndex:1 withObject:@"dog"];
	[mutableDic setObject:@"Mary" forKey:@"name"];
	
##局限
使用字面量语法创建出来的字符串、数组、字典对象都是不可变的(immutable),若想可变版本的对象，则需要赋值一份

	NSMutableDictionary *mutableDic = [@{@"name": @"Tom",@"age":@18} mutableCopy];

##要点

* 应该使用字面量语法创建字符串、数值、数组、字典。与创建此类对象的常规方法相比，更加简明扼要。
* 应该通过下标来访问数组和字典
* 用字面量语法创建数组或字典时，若值中有nil，会抛出异常。
 

##番外
JSON格式，一对 `{ }` 代表一个字典，一对`[]`代表一个数组

	{
        "result": 1,

        "school_list": [
            {
            	"id": "5",
                "school_name": "清华大学"             
            },
            {
            	"id": "6",
                "school_name": "北京邮电大学"
            }
        ],

        "school_near": "北京交通大学"
    }
    
从JSON**字典**中通过键`school_list`取出数组对象

	NSArray *arr = json[@"school_list"];
	NSArray *arr = [json objectForKey:@"school_list"];
	
arr数组的元素是一个个字典，遍历数组得到字典内的学校名

	for( int i=0; i<arr.count; i++){
            NSString *schoolName = arr[i][@"school_name"];
            NSString *schoolName = [[arr objectAtIndex:i] objectForKey:@"school_name"];
        }
   
其实arr数组就是PHP内的二维数组，只不过在OS中，

* 数组 - 下标为数字（PHP中的索引数组）
* 字典 - 下标为字符串（PHP中的关联数组） 

上面例子中二维数组arr的遍历，通过数字下标遍历二维数组中的所有元素，然后通过字符串下标取出学校名。
>下标也称为键