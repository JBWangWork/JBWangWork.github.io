---
title: Runloop底层原理--源码分析
date: 2019-04-05 10:39:29
author: Vincent
categories: 
- 底层原理
tags: 
- Runtime
---


![Runtime底层原理](https://upload-images.jianshu.io/upload_images/5741330-7f293e1c5c8cb366.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


[Runtime官方文档介绍直通车](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtVersionsPlatforms.html#//apple_ref/doc/uid/TP40008048-CH106-SW2)

> 扩展：编译时
看到运行时就会想到编译时，编译时主要是将源代码翻译成可识别的机器语言，如果编译时类型检查等翻译过程中发现语法分析之类有错误会给出相应的提示。比如OC，swift，Java等高级语言的可读性比较强，但是一般不会被机器直接识别，所以需要将他们编译成机器语言（汇编等），转为二进制


#### Runtime简介
在 Objective-C 中，消息是直到运行的时候才和方法实现绑定的。编译器会把一个消息表达式，
```
[receiver message]
```
转换成一个对消息函数objc_msgSend的调用。该函数有两个主要参数:消息接收者和消息对应的方法——也就是方法编号（name），**发送消息时候，通过sel找到函数实现的指针imp**
```
objc_msgSend(receiver, selector)
```
编译器将自动插入调用该消息函数的代码，同时接收消息中的任意数目的参数:
```
objc_msgSend(receiver, selector, arg1, arg2, ...)
```
**该消息函数做了动态绑定所需要的一切:**
- 它首先找到选标所对应的方法实现。因为不同的类对同一方法可能会有不同的实现，所以找到的方法实现依赖于消息接收者的类型。
- 然后将消息接收者对象(指向消息接收者对象的指针)以及方法中指定的参数传给找到的方法实现。
- 最后，将方法实现的返回值作为该函数的返回值返回。

**消息机制的关键在于编译器为类和对象生成的结构。每个类的结构中至少包括两个基本元素:**
- 指向父类的指针。
- 类的方法表。方法表将方法选标和该类的方法实现的地址关联起来。

#### 和运行时系统的交互
Objective-C 程序有三种途径和运行时系统交互:
- 通过Objective-C源代码
大部分情况下，运行时系统在后台自动运行，当编译Objective-C类和方法时，编译器为实现语言动态特性将自动创建一些数据结构和函数。运行时系统的主要功能就是根据源代码中的表达式发送消息。
- 通过Foundation框架中 NSObject 的方法
Cocoa程序中绝大部分类都是NSObject类的子类，所以大部分都继承了NSObject类的方法，因而继承 了NSObject的行为。然而，某些情况下， NSObject类仅仅定义了完成某件事情的模板，而没有提供所有需要的代码（比如 description 方法）。某些 NSObject 的方法只是简单地从运行时系统中获得信息，从而允许对象进行一定程度的自我检查（比如methodForSelector方法返回指定方法实现的地址）。
- 通过调用运行时系统函数
直接调用运行时系统给我们提供的API接口

#### Runtime函数注释
```

// 1.objc_xxx系列函数  宏观使用,如类与协议的空间分配,注册,注销等操作
// 函数名称     函数作用
objc_getClass     获取Class对象
objc_getMetaClass     获取MetaClass对象
objc_allocateClassPair     分配空间,创建类(仅在 创建之后,注册之前 能够添加成员变量)
objc_registerClassPair     注册一个类(注册后方可使用该类创建对象)
objc_disposeClassPair     注销某个类
objc_allocateProtocol     开辟空间创建协议
objc_registerProtocol     注册一个协议
objc_constructInstance     构造一个实例对象(ARC下无效)
objc_destructInstance     析构一个实例对象(ARC下无效)
objc_setAssociatedObject     为实例对象关联对象
objc_getAssociatedObje*ct     获取实例对象的关联对象
objc_removeAssociatedObjects     清空实例对象的所有关联对象

// 2.class_xxx系列函数   类的内部,如实例变量,属性,方法,协议等相关问题
函数名称     函数作用
class_addIvar     为类添加实例变量
class_addProperty     为类添加属性
class_addMethod     为类添加方法
class_addProtocol     为类遵循协议
class_replaceMethod     替换类某方法的实现
class_getName     获取类名
class_isMetaClass     判断是否为元类
objc_getProtocol     获取某个协议
objc_copyProtocolList     拷贝在运行时中注册过的协议列表
class_getSuperclass     获取某类的父类
class_setSuperclass     设置某类的父类
class_getProperty     获取某类的属性
class_getInstanceVariable     获取实例变量
class_getClassVariable     获取类变量
class_getInstanceMethod     获取实例方法
class_getClassMethod     获取类方法
class_getMethodImplementation     获取方法的实现
class_getInstanceSize     获取类的实例的大小
class_respondsToSelector     判断类是否实现某方法
class_conformsToProtocol     判断类是否遵循某协议
class_createInstance     创建类的实例
class_copyIvarList     拷贝类的实例变量列表
class_copyMethodList     拷贝类的方法列表
class_copyProtocolList     拷贝类遵循的协议列表
class_copyPropertyList     拷贝类的属性列表

// 3.object_xxx系列函数   对象的角度,如实例变量
函数名称     函数作用
object_copy     对象copy(ARC无效)
object_dispose     对象释放(ARC无效)
object_getClassName     获取对象的类名
object_getClass     获取对象的Class
object_setClass     设置对象的Class
object_getIvar     获取对象中实例变量的值
object_setIvar     设置对象中实例变量的值
object_getInstanceVariable     获取对象中实例变量的值 (ARC中无效,使用object_getIvar)
object_setInstanceVariable     设置对象中实例变量的值 (ARC中无效,使用object_setIvar)

// 4.method_xxx系列函数   方法内部,如方法的参数及返回值类型和方法的实现
函数名称     函数作用
method_getName     获取方法名
method_getImplementation     获取方法的实现
method_getTypeEncoding     获取方法的类型编码
method_getNumberOfArguments     获取方法的参数个数
method_copyReturnType     拷贝方法的返回类型
method_getReturnType     获取方法的返回类型
method_copyArgumentType     拷贝方法的参数类型
method_getArgumentType     获取方法的参数类型
method_getDescription     获取方法的描述
method_setImplementation     设置方法的实现
method_exchangeImplementations     替换方法的实现

// 5.property_xxx系列函数   属性*内部,如属性的特性等
函数名称     函数作用
property_getName     获取属性名
property_getAttributes     获取属性的特性列表
property_copyAttributeList     拷贝属性的特性列表
property_copyAttributeValue     拷贝属性中某特性的值

// 6.protocol_xxx系列函数 协议相关
函数名称     函数作用
protocol_conformsToProtocol     判断一个协议是否遵循另一个协议
protocol_isEqual     判断两个协议是否一致
protocol_getName     获取协议名称
protocol_copyPropertyList     拷贝协议的属性列表
protocol_copyProtocolList     拷贝某协议所遵循的协议列表
protocol_copyMethodDescriptionList     拷贝协议的方法列表
protocol_addProtocol     为一个协议遵循另一协议
protocol_addProperty     为协议添加属性
protocol_getProperty     获取协议中的某个属性
protocol_addMethodDescription     为协议添加方法描述
protocol_getMethodDescription     获取协议中某方法的描述

// 7.ivar_xxx 系列函数  实例变量相关
函数名称     函数作用
ivar_getName     获取Ivar名称
ivar_getTypeEncoding     获取类型编码
ivar_getOffset     获取偏移量

// 8.sel_xxx系列函数   方法编号相关
函数名称     函数作用
sel_getName     获取名称
sel_getUid     注册方法
sel_registerName     注册方法名
sel_isEqual     判断方法是否相等

// 9.imp_xxx系列函数   方法实现相关
函数名称     函数作用
imp_implementationWithBlock     通过代码块创建IMP
imp_getBlock     获取函数指针中的代码块
imp_removeBlock     移除IMP中的代码块

```
如果想继续探究Runtime底层原理，下篇是[Runtime源码分析]([https://www.jianshu.com/p/1ddd15e47343](https://www.jianshu.com/p/1ddd15e47343)
)，包括动态方法解析和消息转发。
该文章为记录本人的学习路程，希望能够帮助大家！！！


