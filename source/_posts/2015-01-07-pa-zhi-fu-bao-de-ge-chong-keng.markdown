---
layout: post
title: "爬支付宝的各种坑"
date: 2015-01-07 13:57:57 +0800
comments: false
categories: 支付宝
---

爬了几天坑，在笔者崩溃之际终于集成好了，在这里分享一下经验。

笔者用的是`支付宝移动支付SDK标准版(iOS 2.1.2)`，申请的手机快捷支付
##密钥
利用支付宝给的`openssl.exe`工具生成一共三种密钥，私钥、公钥，PKCS8类型的私钥
 
 - 公钥：放在支付宝上，去掉首尾的`-----BEGIN PUBLIC KEY-----``-----END PUBLIC KEY-----`
 - PKCS8类型的私钥：如果选择在手机端签名，此私钥放在`NSString *privateKey = @"PKCS8类型的私钥"`
 - 私钥：此私钥带有`-----BEGIN RSA PRIVATE KEY-----` `-----END RSA PRIVATE KEY-----`，是专门给PHP用的，即在服务器端(PHP)签名，需要用到此私钥
 > Note：推荐服务器端签名，不过需要注意的是，PHP端用的是带`-----BEGIN RSA PRIVATE KEY-----` `-----END RSA PRIVATE KEY-----`的私钥。
 

##集成支付宝
按照文档给的流程理论上想运行Demo是挺容易的，需要注意：

- Demo是在本地签的名，所以Demo上填写`NSString *privateKey = @"PKCS8类型的私钥"`

如果认为看似Demo超简单，你就错了。
###坑一：Undefined symbols for architecture x86_64:


笔者爬过的坑，`ssl库`不支持x86_64就是64位模拟器。（若在服务器端签名不需引入ssl库）
> 程序界公认女神@念茜给出的答案。。。

也就是以下这种错误

![alipay error](http://limons-gitimage.stor.sinaapp.com/alipay.png)

针对这个错误，笔者尝试了各种方法，Architectures的各种设置，甚至都是一行一行对着Demo来改的，还有C++ Flage等等，最终还是不行。

和客服聊了一个多小时，最后客服把支付宝最老的版本和13年的版本给笔者。。。。挺好人的。。。。

###坑二：rsaSign()和rsaVerify()

作为一个技术渣，iOS我是搞不掂了，还是搞PHP吧。。。

吐血的是，笔者用支付宝给的PHP版Demo，用它里面的方法`rsaSign()`签名，然后再用`rsaVerify()`验签，结果`rsaVerify()`返回的结果永远都是false，后来笔者终于准备出坑了，醒悟这个`rsaVerify()`应该不是验证这签名的。

后来百度了解了点皮毛：

RSA非对称密钥

① 假设A、B机器进行通信，已A机器为主；

② A首先需要用自己的私钥为发送请求数据签名，并将公钥一同发送给B；

③ B收到数据后，需要用A发送的公钥进行验证，已确保收到的数据是未经篡改的；

④ B验签通过后，处理逻辑，并把处理结果返回，返回数据需要用A发送的公钥进行加密（公钥加密后，只能用配对的私钥解密）；

⑤ A收到B返回的数据，使用私钥解密，至此，一次数据交互完成。

> `rsaVerify`笔者猜测是验证支付宝的回调的。换句话说，签名直接就用`rsaSign()`就好了，就以下这几行代码

```
<?php
include('alipay_rsa.function.php');

$data = $_POST['data'];
$sign = rsaSign($data,'../key/rsa_private_key.pem');

$arr = array(
	'sign' => $sign
);

echo json_encode($arr);
?>
```
以为这样就爬完了？

###坑三：urlEncoded
想着服务器直接返回`sign`直接再拼接一下就提交给支付宝，结果又掉坑里。

	NSString *signedString = json[@"sign"];
    NSLog(@"signedString-----%@",signedString);
        
    NSString *orderString = [NSString stringWithFormat:@"%@&sign=\"%@\"&sign_type=\"%@\"",
                                 orderSpec, signedString, @"RSA"];
        
    [[AlipaySDK defaultService] payOrder:orderString fromScheme:appScheme callback:^(NSDictionary *resultDic).....
    
又跑去和客服聊天，笔者问客服：服务器返回的`sign`应该还需要`urlEncoded`对吧，客服回我说：不用啊，服务器已经弄了啊。

	/**
 	* RSA签名
 	* @param $data 待签名数据
 	* @param $private_key_path 商户私钥文件路径
 	* return 签名结果
 	*/
	function rsaSign($data, $private_key_path) {
    	$priKey = file_get_contents($private_key_path);
    	$res = openssl_pkey_get_private($priKey);
    	openssl_sign($data, $sign, $res);
    	openssl_free_key($res);
		//base64编码
    	$sign = base64_encode($sign);
    	return $sign;
	}
	
客服说`base64_encode($sign)`完就可以了不需要`urlEncoded`。笔者果断还是不相信客服。

 	NSString *signedString = json[@"sign"];
 	NSLog(@"signedString-----%@",signedString);
    
    // 深坑啊......    
 	signedString = [self urlEncodedString:signedString];
        
 	NSString *orderString = [NSString stringWithFormat:@"%@&sign=\"%@\"&sign_type=\"%@\"",
                                 orderSpec, signedString, @"RSA"];
        
 	[[AlipaySDK defaultService] payOrder:orderString fromScheme:appScheme callback:^(NSDictionary *resultDic)....

需要在`RSADataSigner.m`中抽离`-urlEncodedString`方法出来。。。坑啊，明明都不需要手机端签名了，当然不会去看手机端签名的具体流程啦。又没看到哪里说明服务器端返回的`sign`需要`urlEncoded`，连客服都不知道需要`urlEncoded`.....

###心塞，不说了.......

