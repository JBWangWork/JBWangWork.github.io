---
title: Xcode无线调试
date: 2019-06-13 09:50:57
author: Vincent
categories: 
- 测试
tags: 
- Xcode调试
---

>Xcode无线调试是WWDC2017的一个新功能，首先要满足iOS11以上，Xcode9以上；

首先，把iOS11以上的iOS设备连接到Xcode9，`shift + Commond + 2`快速打开设备列表，或者在菜单中打开window，找到Device and simulators。
![设备列表](https://upload-images.jianshu.io/upload_images/5741330-7238d561bae28e04.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
打开后，勾选Connect via network，成功后拔掉数据线。
![连接局域网](https://upload-images.jianshu.io/upload_images/5741330-01c0b1accb60a294.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在左侧列表中，右击刚刚连接的iOS设备，找到Connect via IP address
![设置局域网](https://upload-images.jianshu.io/upload_images/5741330-c61c242cf41572b0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

输入iOS设备的IP地址，connect连接成功。
![设置IP地址](https://upload-images.jianshu.io/upload_images/5741330-d12495841f163111.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

>IP地址在iOS设备的设置->无线局域网，找到当前的网络，进入详情中查看IP地址。

现在iOS设备不需要连接数据线就可以进行调试了，这个也是很早就开始用了，只不过最近突然出现点问题，重新配置并记录下。

如果出现`The device must have a passcode set in order to allow this operation`错误，那么给手机设置一个密码，再次重新操作就OK了。希望能够帮助大家！！！


