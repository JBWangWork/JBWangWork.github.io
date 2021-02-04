---
title: Runtime底层原理--IMP查找流程、动态方法解析、消息转发源码分析
date: 2019-04-10 18:48:01
author: Vincent
categories: 
- 底层原理
- 源码分析
tags: 
- Runtime
---


![Runtime底层原理](https://upload-images.jianshu.io/upload_images/5741330-3e21c38f5be9a8d5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

了解了[Runtime函数含义](https://www.jianshu.com/p/eddc9bdb46ea)，我们就可以直接使用Runtime的API了，那接下来继续探究Runtime的源码，经过源码分析来更加深刻的了解Runtime原理。

#### 开发应用
> 都知道Runtime很重要，但是有很多小伙伴根本不了解，或者只是知道可以替换方法啊、可以交换两个方法的调用，项目中也用不到，
> 从进入iOS开始，写了无数个`[[objc alloc] init]`，这个到底在干嘛？初始化和init？alloc和init到底做了什么？

##### 通过汇编查看方法调用
```
        Person *person = [Person alloc];
        Person *person1 = [person init];
        Person *person2 = [person init];
        NSLog(@"%p-----%p------%p", person, person1, person2);
```

这里会输出什么呢？

```
0x10102e1a0-----0x10102e1a0------0x10102e1a0
```

来，让我们断点看下，`alloc`和`init`是怎么调用的

![objc_msgSend](https://upload-images.jianshu.io/upload_images/5741330-df50f7ae402013ac.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们看到调用`alloc`和`init`都调起了`objc_msgSend`，接下来跟着符号断点走

![libobjc](https://upload-images.jianshu.io/upload_images/5741330-7706f8564d5b447d.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![callAlloc](https://upload-images.jianshu.io/upload_images/5741330-7f2f3a95f30891e6.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

进入`libobjc`库的dylib之后走`+[NSObject alloc]`方法，指针调起`_objc_rootAlloc`，进入`_objc_rootAlloc`方法，继续调起`callAlloc`，通过寄存器，可以看到alloc已经通过类创建实例对象

![类对象](https://upload-images.jianshu.io/upload_images/5741330-6839fe399aa384bd.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

`init`按照同样方法 依然可以通过汇编看出方法调用顺序，可以用真机进行测试并打印 

##### 通过编译C++
当新的对象被创建时，其内存同时被分配，实例变量也同时被初始化。对象的第一个实例变量是一个指向该对象的类结构的指针，叫做 isa。通过该指针，对象可以访问它对应的类以及相应的父类。在 Objective-C 运行时系统中对象需要有 isa 指针，我们一般创建的从 NSObject 或者 NSProxy 继承的对象都自动包括 isa 变量。接下来看下对象被创建的过程
首先，我们通过clang命令
```
$ xcrun -sdk iphoneos clang -arch arm64 -rewrite-objc main.m -o testMain.c++
```
也可以用`clang -rewrite-objc main.m -o test.c++`命令，只不过会有很多警告、代码会更长（大概9万多行）。
编译main函数中的OC代码为C++代码
```
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        Person *p = [[Person alloc] init];
        [p run];
  
    }
    return 0;
}
```
编译后多一个testMain.c++文件，打开后在代码最后面会发现我们的main函数
```
int main(int argc, const char * argv[]) {
    /* @autoreleasepool */ { __AtAutoreleasePool __autoreleasepool; 
        Person *p = ((Person *(*)(id, SEL))(void *)objc_msgSend)((id)((Person *(*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("Person"), sel_registerName("alloc")), sel_registerName("init"));
        ((void (*)(id, SEL))(void *)objc_msgSend)((id)p, sel_registerName("run"));

    }
    return 0;
}
```

可以看出，我们的方法调用会编译成objc_msgSend，

![person对象](https://upload-images.jianshu.io/upload_images/5741330-6b959a3252ac08d6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

由此还会发现对象的本质其实就是一个结构体

#### 下层通讯(通过源码查看objc_msgSend内部实现)
首先我们到[苹果open source](https://upload-images.jianshu.io/upload_images/5741330-3acf8ade7f319c5f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)官网下载最新源码
![源码](https://upload-images.jianshu.io/upload_images/5741330-3acf8ade7f319c5f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在**方法调用的时候，会发送`objc_msgSend`消息，`objc_msgSend`会根据sel找到函数实现的指针imp**，进而执行函数，那sel是如何找到imp的呢？
`objc_msgSend`在发送消息时候根据sel查找imp有两种方式
- 快速（通过汇编的缓存快速查找）
- 慢速（C配合C++、汇编一起查找）
先看下objc_class

![objc_class](https://upload-images.jianshu.io/upload_images/5741330-da01958ba32dd572.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

bits中包含各种数据，cache（每个类都有一个）用来存储方法select和imp，select和imp会以哈希表形式存在
`objc_msgSend`在快速查找的时候，就是通过汇编查找objc_class中的cache，如果找到则直接返回，否则通过C的lookup，找到后再存入cache

##### 汇编部分快速查找

首先调用`objc_msgSend`会走到ENTRY

![ENTRY](https://upload-images.jianshu.io/upload_images/5741330-4d838bce5a43c8e6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

先判断p0检查是否为空和tagged pointer（特殊类型）判断，调用`LNilOrTagged`进行isa处理，通过isa找到相应类class，最后调用`LGetIsaDone`来执行`CacheLookup`在缓存中查找imp，如果查找到直接调起imp否则调起objc_msgSend_uncached，objc_msgSend_uncached有两种情况

![CacheLookup](https://upload-images.jianshu.io/upload_images/5741330-41e6fb32bc6de43d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

首先，第一个是CacheHit，直接调起imp，第二个是CheckMiss，之后调用objc_msgSend_uncached，第三个就是add，下面是CacheHit和CheckMiss的宏

![CacheLookup macro](https://upload-images.jianshu.io/upload_images/5741330-a88860fb2c1871b0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

那如果在缓存中没有查找到imp，调起`objc_msgSend_uncached`，在方法列表中找到imp之后再`TailCallFunctionPointer`调起imp
```
    STATIC_ENTRY __objc_msgSend_uncached
	UNWIND __objc_msgSend_uncached, FrameWithNoSaves

	// THIS IS NOT A CALLABLE C FUNCTION
	// Out-of-band p16 is the class to search
	
	MethodTableLookup      // 方法列表中找到imp
	TailCallFunctionPointer x17
```

**重点:MethodTableLookup是怎么操作的**
> 小知识点：通过method list查找method，下面是method_t的结构，method其实是一个哈希表，sel和imp是键值对
> ```
> struct method_t {
>     SEL name;
>     const char *types;       // 参数类型
>     MethodListIMP imp;
>     struct SortBySELAddress :
>         public std::binary_function<const method_t&,
>                                     const method_t&, bool>
>     {
>         bool operator() (const method_t& lhs,
>                          const method_t& rhs)
>         { return lhs.name < rhs.name; }
>     };
> };
>
> ```
进入`MethodTableLookup`之后，调起了`__class_lookupMethodAndLoadCache3`，如下图

![MethodTableLookup](https://upload-images.jianshu.io/upload_images/5741330-e53cc3c30a78bbe9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

`__class_lookupMethodAndLoadCache3`是C方法，再次进入`_class_lookupMethodAndLoadCache3`方法，**注意，因为这里由汇编跳转到C，所以要全局搜索`_class_lookupMethodAndLoadCache3`，要删去一个`"_"`**,下面是`_class_lookupMethodAndLoadCache3`函数

```
/***********************************************************************
* _class_lookupMethodAndLoadCache.
* Method lookup for dispatchers ONLY. OTHER CODE SHOULD USE lookUpImp().
* This lookup avoids optimistic cache scan because the dispatcher 
* already tried that.
**********************************************************************/
IMP _class_lookupMethodAndLoadCache3(id obj, SEL sel, Class cls)
{
    return lookUpImpOrForward(cls, sel, obj, 
                              YES/*initialize*/, NO/*cache*/, YES/*resolver*/);
}
```

##### C/C++部分查找

调起`lookUpImpOrForward `，因为当前cls对象已经经过汇编编译到结构，有了isa，并且在cache中没有找到，所以这里的initialize为YES，cache为NO，resolver为YES

![image.png](https://upload-images.jianshu.io/upload_images/5741330-71ad5c6ba37152f2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

进入`lookUpImpOrForward`，这里再次判断是否存在cache，如果有则直接快速查找，但是这里是NO，所以不会走。接下来走`checkIsKnownClass`判断是否是已经声明的类，如果没有则报错"Attempt to use unknown class %p."，之后走`realizeClass `判断是否已经实现，如果就相应赋值data。

![realizeClass](https://upload-images.jianshu.io/upload_images/5741330-8916248bb091fe67.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

data赋值后走`_class_initialize`初始化cls，接下来开始`retry`操作。
**前方高能**
再次进行cache_getImp，why？并发啊，还有重映射（在初始化init的时候有个remap（class）第一次通过汇编找不到，但是在加载类的时候对当前类进行重映射）
![cache_getImp](https://upload-images.jianshu.io/upload_images/5741330-cbe3379e8e73bf14.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

接下来开始先在自己的class_rw_t的methods中根据sel查找方法返回method_t

![method_t](https://upload-images.jianshu.io/upload_images/5741330-22158fedc376e3de.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如果拿到Method后保存到缓存中，保证以后调用可以直接走汇编的CacheHit快速查找，如果拿不到则继续从父类开始查找，直到找到NSObject(因为NSObject的父类为nil)，如果找到imp则一样保存在缓存中，如果到最后还是没有查找到，则进入动态方法解析。
![父类查找方法](https://upload-images.jianshu.io/upload_images/5741330-348eeda298ad6d52.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### 动态方法解析
如果前面一系列操作还是没有找到方法，那么就会进行动态方法解析，动态方法解析只执行一次

![动态方法解析](https://upload-images.jianshu.io/upload_images/5741330-9cbb884006c3e59c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

首先执行`_class_resolveMethod`，这里会执行`+resolveClassMethod` 或者 `+resolveInstanceMethod`。
![class resolveMethod](https://upload-images.jianshu.io/upload_images/5741330-c5cda417fae04acb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

先判断当前cls是否为元类，如果是元类则执行`_class_resolveClassMethod`，再执行`_class_resolveInstanceMethod`，如果不是元类则直接执行`_class_resolveInstanceMethod`，`_class_resolveInstanceMethod`内部调用objc_msgSend实现消息发送，对cls发送了`SEL_resolveInstanceMethod`类型的消息，所以在方法中会走到`resolveInstanceMethod`方法。

![class resolveInstanceMethod](https://upload-images.jianshu.io/upload_images/5741330-0ed2701ca3678b01.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

为什么元类最后也执行了`_class_resolveInstanceMethod`方法呢？因为类方法以实例对象的形态存在元类里面，比如类方法中没有找到方法，会去元类中查找，元类中没有再继续去根元类中查找，最后会查到NSObject。
###### 代码示例：
.h实现
```
- (void)run;
+ (void)eat;
```

.m实现(没有实现-run方法和+eat方法)
```
- (void)walk {
    NSLog(@"%s",__func__);
}
+ (void)drink {
    NSLog(@"%s",__func__);
}

// .m没有实现,并且父类也没有,那么我们就开启动态方法解析
//- (void)walk{
//    NSLog(@"%s",__func__);
//}
//+ (void)drink{
//    NSLog(@"%s",__func__);
//}


#pragma mark - 动态方法解析

+ (BOOL)resolveInstanceMethod:(SEL)sel{
    if (sel == @selector(run)) {
        // 我们动态解析我们的 对象方法
        NSLog(@"对象方法解析走这里");
        SEL walkSEL = @selector(walk);
        Method readM= class_getInstanceMethod(self, walkSEL);
        IMP readImp = method_getImplementation(readM);
        const char *type = method_getTypeEncoding(readM);
        return class_addMethod(self, sel, readImp, type);
    }
    return [super resolveInstanceMethod:sel];
}


+ (BOOL)resolveClassMethod:(SEL)sel{
    if (sel == @selector(eat)) {
        // 我们动态解析我们的 对象方法
        NSLog(@"类方法解析走这里");
        SEL drinkSEL = @selector(drink);
        // 类方法就存在我们的元类的方法列表
        // 类 类犯法
        // 元类 对象实例方法
        //        Method hellowordM1= class_getClassMethod(self, hellowordSEL);
        Method drinkM= class_getInstanceMethod(object_getClass(self), drinkSEL);
        IMP drinkImp = method_getImplementation(drinkM);
        const char *type = method_getTypeEncoding(drinkM);
        NSLog(@"%s",type);
        return class_addMethod(object_getClass(self), sel, drinkImp, type);
    }
    return [super resolveClassMethod:sel];
}
```

##### 消息转发
经历了动态方法决议还没有找到，会进入苹果尚未开源的消息转发，继续查找方法，`_objc_msgForward_impcache`再次跨域到汇编。

![消息转发](https://upload-images.jianshu.io/upload_images/5741330-c9fa6f055cdbc9d6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

走到`__objc_msgForward_impcache`后执行`__objc_msgForward`

![__objc_msgForward_impcache](https://upload-images.jianshu.io/upload_images/5741330-bb97c18875125e49.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

没有了源码实现，但是我们可以通过`instrumentObjcMessageSends`函数来打印调用堆栈信息。可以进入`instrumentObjcMessageSends`内部看下具体实现。

![instrumentObjcMessageSends](https://upload-images.jianshu.io/upload_images/5741330-f7a9ba5a7514aff9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

先判断了是否可以写入日志信息等，接下来同步日志文件

![logMessageSend](https://upload-images.jianshu.io/upload_images/5741330-e67afa0438c44bd0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

所以我们每次运行会在`/private/tmp`文件下多一个`msgSends-xxx`文件，里面是所有调用过程

![堆栈调用信息](https://upload-images.jianshu.io/upload_images/5741330-7879482357d545ad.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如果还没有找到的话最后会报错调用`__objc_forward_handler`

![__objc_forward_handler](https://upload-images.jianshu.io/upload_images/5741330-0c3110e83a8752ff.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这也是我们在方法报错的时候会报`unrecognized selector sent to instance %p " "(no message forward handler is installed)"`错误的原因，会提示出元类信息，`+`或者`-`方法，方法的名字还有SEL方法编号
###### 代码示例：
```
#pragma mark - 实例对象消息转发

- (id)forwardingTargetForSelector:(SEL)aSelector{
    NSLog(@"%s",__func__);
    //    if (aSelector == @selector(run)) {
    //        // 转发给Student对象
    //        return [Student new];
    //    }
    return [super forwardingTargetForSelector:aSelector];
}

- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector{
    NSLog(@"%s",__func__);
    if (aSelector == @selector(run)) {
        // forwardingTargetForSelector 没有实现，就只能方法签名了
        return [NSMethodSignature signatureWithObjCTypes:"v@:@"];
    }
    return [super methodSignatureForSelector:aSelector];
}

- (void)forwardInvocation:(NSInvocation *)anInvocation{
    NSLog(@"%s",__func__);
    NSLog(@"------%@-----",anInvocation);
    anInvocation.selector = @selector(walk);
    [anInvocation invoke];
}

#pragma mark - 类消息转发

+ (id)forwardingTargetForSelector:(SEL)aSelector{
    NSLog(@"%s",__func__);
    return [super forwardingTargetForSelector:aSelector];
}
//

+ (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector{
    NSLog(@"%s",__func__);
    if (aSelector == @selector(walk)) {
        return [NSMethodSignature signatureWithObjCTypes:"v@:@"];
    }
    return [super methodSignatureForSelector:aSelector];
}

+ (void)forwardInvocation:(NSInvocation *)anInvocation{
    NSLog(@"%s",__func__);
    
    NSString *sto = @"奔跑吧";
    anInvocation.target = [Student class];
    [anInvocation setArgument:&sto atIndex:2];
    NSLog(@"%@",anInvocation.methodSignature);
    anInvocation.selector = @selector(run:);
    [anInvocation invoke];
}
```

现在我们应该也知道了为什么`objc_msgSend`的源码用的汇编，因为汇编可以通过寄存器x0-x31来保留未知参数来跳转到任意的指针，还有汇编更高效一点，而C满足不了。
#### 言而总之，总而言之
> Runtime就是C、C++、汇编实现的一套API，给OC增加的一个运行时功能，也就是我们平时所说的运行时。
在运行工程时工程会被装载到内存，来提供运行时功能。

该文章为记录本人的学习路程，希望能够帮助大家！！！


