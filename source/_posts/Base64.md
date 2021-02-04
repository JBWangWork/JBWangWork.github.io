---
title: 了解Base64编码解码
date: 2019-01-05 18:29:21
author: Vincent
categories: 
- 算法
- 加密原理
tags: 
- 算法
- 非对称加密原理
- OpenSSL
---


![了解Base64编码解码](https://upload-images.jianshu.io/upload_images/5741330-161cc8afdcaec6f5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


>我们经常说Base64，那Base64到底是什么呢？
[**Base64**](https://zh.wikipedia.org/zh-hans/Base64)是一种基于64个可打印字符来表示[二进制数据](https://zh.wikipedia.org/wiki/%E4%BA%8C%E8%BF%9B%E5%88%B6 "二进制")的表示方法，常用于在通常处理文本[数据](https://zh.wikipedia.org/wiki/%E6%95%B0%E6%8D%AE "数据")的场合，表示、传输、存储一些二进制数据，会将不便于查看的二进制数据用Base64进行表示。所以Bsea64经常用于密码学中，因为密码学通常用二进制进行加密，加密的结果用Base64编码来表示并传输。

我们想了解Base64，其实看下面的Base64索引表就可以了。
![Base64索引表](https://upload-images.jianshu.io/upload_images/5741330-f9b1a93c393d27f8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在Base64中的可打印字符包括[字母](https://zh.wikipedia.org/wiki/%E6%8B%89%E4%B8%81%E5%AD%97%E6%AF%8D "拉丁字母")`A-Z`、`a-z`、[数字](https://zh.wikipedia.org/wiki/%E6%95%B0%E5%AD%97 "数字")`0-9`共有62个字符，加上`+`、`/`共64个字符，实际上还有一个字符`=`来作为后缀。比如：编码Man
![编码Man](https://upload-images.jianshu.io/upload_images/5741330-578423235d58ea16.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

当Base64对一个二进制数据进行编码时，每6个[位元](https://zh.wikipedia.org/wiki/%E4%BD%8D%E5%85%83 "位元")为一个单元，对应某个可打印字符。3个[字节](https://zh.wikipedia.org/wiki/%E5%AD%97%E8%8A%82 "字节")有24个位元，对应于4个Base64单元，即3个字节可由4个可打印字符来表示，所以最少要24个比特位。如果不足24位，就在后面补0，后面补的0就会用`=`来表示，所以`=`也只会在最后面。

#### 终端演示Base64编码

```
// 通过Base64将111图片进行编码，生成111.txt文件
$ base64 111.png -o 111.txt
// 对111.txt文件解码，生成222.png
$ base64 111.txt -o 222.png -D
```
![Base64编码](https://upload-images.jianshu.io/upload_images/5741330-a811b9c976ba1fb5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

左侧的图片是原始文件，通过Base64编码后输出右侧111.txt文件，再对111.txt文件解码还原。

#### 代码演示Base64编码
Base64也是在iOS7以后出现的，接下来用代码简单操作一下
```
//
//  ViewController.m
//  Base64
//
//  Created by Vincent on 2019/1/14.
//  Copyright © 2019 Vincent. All rights reserved.
//

#import "ViewController.h"

@interface ViewController ()

@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    // Do any additional setup after loading the view, typically from a nib.
}

- (NSString *)getBase64Encode:(NSString *)encodeStr {
    // 将传进来的string转成NSData，再进行Base64编码
    NSData *data = [encodeStr dataUsingEncoding:NSUTF8StringEncoding];
    return [data base64EncodedStringWithOptions:0];
}

- (NSString *)getBase64Decode:(NSString *)decodeStr {
     // 由于传过来的是Base64编码字符串，则不需要先转二进制再解码，可以直接通过NSData初始化方法解码
    NSData *data = [[NSData alloc] initWithBase64EncodedString:decodeStr options:0];
    // 将data转成string
    return [[NSString alloc] initWithData:data encoding:NSUTF8StringEncoding];
    
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    NSLog(@"----编码:%@", [self getBase64Encode:@"abc"]);
    NSLog(@"####解码:%@", [self getBase64Decode:[self getBase64Encode:@"abc"]]);
}

@end

```
- 打印结果
```
----编码:YWJj
####解码:abc
```
- 终端验证
```
// 通过Base64将abc进行编码
$  echo -n abc | base64
YWJj
```
验证通过！！！但是通过Base64编码，我们会发现编码结果会变大1/3。
该文章为记录本人的学习路程，希望能够帮助大家！！！

