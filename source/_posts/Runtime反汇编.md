---
title: Runtime底层原理总结--反汇编分析消息转发
date: 2019-06-22 20:52:09
author: Vincent
top: true
categories: 
- 底层原理
- 源码分析
tags: 
- Runtime
- 反汇编
- 源码分析
---


![反汇编分析消息转发](https://upload-images.jianshu.io/upload_images/5741330-0353c6ce7788cf39.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


> 消息转发：发送一个消息，也就是sel查找imp，当没有找到imp，接下来进入动态方法解析，如果开发者并没有处理，会进入消息转发。
#### 消息转发
前几篇文章介绍了[Runtime底层原理](https://www.jianshu.com/p/1ddd15e47343)和[动态方法解析总结](https://www.jianshu.com/p/a7db9f0c82d6)

，我们知道如果前面的动态方法解析也没有解决问题的话，那么就会进入消息转发`_objc_msgForward_impcache`方法，会有快速消息转发和慢速消息转发。
`_objc_msgForward_impcache`方法会从C转换到汇编部分`__objc_msgForward_impcache`进行快速消息转发，执行闭源`__objc_msgForward`。
![汇编部分__objc_msgForward](https://upload-images.jianshu.io/upload_images/5741330-3eb7d8eb18b221b2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如果我们的方法没有查找到会报错`_forwarding_prep_0`
![崩溃信息](https://upload-images.jianshu.io/upload_images/5741330-9ef2ef6fcc43679a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


但是我们在源代码中找不到该方法，除了前面文章--[Runtime底层原理--动态方法解析、消息转发源码分析](https://www.jianshu.com/p/1ddd15e47343)提到的方法外，我们可以用反汇编分析消息转发。

首先进入`/Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/Library/CoreSimulator/Profiles/Runtimes/iOS.simruntime/Contents/Resources/RuntimeRoot/System/Library/Frameworks/CoreFoundation.framework`中，找到可执行文件`CoreFoundation`，将该文件拖入hopper中，找到`CFInitialze`可以看到了`___forwarding_prep_0___`
![forwarding_prep_0](https://upload-images.jianshu.io/upload_images/5741330-0b90c8c5ea93efc0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

进入`___forwarding_prep_0___`内部，
![____forwarding___](https://upload-images.jianshu.io/upload_images/5741330-05fba25f458f9938.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

从崩溃堆栈信息中看到有执行forwarding，进入forwarding内部，里面判断了_objc_msgSend_stret、_objc_msgSend、taggedpointer之后有个`forwardingTargetForSelector:`，判断`forwardingTargetForSelector:`没有实现，没有实现跳转到loc_126fc7，
![forwardingTargetForSelector](https://upload-images.jianshu.io/upload_images/5741330-0a615756710dd4d5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

进入loc_126fc7，判断僵尸对象之后执行方法签名`methodSignatureForSelector:`，如果有值，获取Name、是否有效的位移等，之后会响应`_forwardStackInvocation:`，

![_forwardStackInvocation](https://upload-images.jianshu.io/upload_images/5741330-828729004c8eb7f0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如果没有响应`_forwardStackInvocation:`，则会响应`forwardInvocation:`，给rdi发送rax消息，rdi就是NSInvocation，rax就是sel`forwardInvocation:`，__这就是消息转发的流程__

![selforwardInvocation](https://upload-images.jianshu.io/upload_images/5741330-025bfb972fbb00fd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

对`_objc_msgForward`部分进行反汇编成OC代码：
```

int __forwarding__(void *frameStackPointer, int isStret) {
    id receiver = *(id *)frameStackPointer;
    SEL sel = *(SEL *)(frameStackPointer + 8);
    const char *selName = sel_getName(sel);
    Class receiverClass = object_getClass(receiver);
    
    // 调用 forwardingTargetForSelector:
    if (class_respondsToSelector(receiverClass, @selector(forwardingTargetForSelector:))) {
        id forwardingTarget = [receiver forwardingTargetForSelector:sel];
        if (forwardingTarget && forwarding != receiver) {
            if (isStret == 1) {
                int ret;
                objc_msgSend_stret(&ret,forwardingTarget, sel, ...);
                return ret;
            }
            return objc_msgSend(forwardingTarget, sel, ...);
        }
    }
    
    // 僵尸对象
    const char *className = class_getName(receiverClass);
    const char *zombiePrefix = "_NSZombie_";
    size_t prefixLen = strlen(zombiePrefix); // 0xa
    if (strncmp(className, zombiePrefix, prefixLen) == 0) {
        CFLog(kCFLogLevelError,
              @"*** -[%s %s]: message sent to deallocated instance %p",
              className + prefixLen,
              selName,
              receiver);
        <breakpoint-interrupt>
    }
    
    // 调用 methodSignatureForSelector 获取方法签名后再调用 forwardInvocation
    if (class_respondsToSelector(receiverClass, @selector(methodSignatureForSelector:))) {
        NSMethodSignature *methodSignature = [receiver methodSignatureForSelector:sel];
        if (methodSignature) {
            BOOL signatureIsStret = [methodSignature _frameDescriptor]->returnArgInfo.flags.isStruct;
            if (signatureIsStret != isStret) {
                CFLog(kCFLogLevelWarning ,
                      @"*** NSForwarding: warning: method signature and compiler disagree on struct-return-edness of '%s'.  Signature thinks it does%s return a struct, and compiler thinks it does%s.",
                      selName,
                      signatureIsStret ? "" : not,
                      isStret ? "" : not);
            }
            if (class_respondsToSelector(receiverClass, @selector(forwardInvocation:))) {
                NSInvocation *invocation = [NSInvocation _invocationWithMethodSignature:methodSignature frame:frameStackPointer];
                
                [receiver forwardInvocation:invocation];
                
                void *returnValue = NULL;
                [invocation getReturnValue:&value];
                return returnValue;
            } else {
                CFLog(kCFLogLevelWarning ,
                      @"*** NSForwarding: warning: object %p of class '%s' does not implement forwardInvocation: -- dropping message",
                      receiver,
                      className);
                return 0;
            }
        }
    }
    
    SEL *registeredSel = sel_getUid(selName);
    
    // selector 是否已经在 Runtime 注册过
    if (sel != registeredSel) {
        CFLog(kCFLogLevelWarning ,
              @"*** NSForwarding: warning: selector (%p) for message '%s' does not match selector known to Objective C runtime (%p)-- abort",
              sel,
              selName,
              registeredSel);
    } // doesNotRecognizeSelector
    else if (class_respondsToSelector(receiverClass,@selector(doesNotRecognizeSelector:))) {
        [receiver doesNotRecognizeSelector:sel];
    }
    else {
        CFLog(kCFLogLevelWarning ,
              @"*** NSForwarding: warning: object %p of class '%s' does not implement doesNotRecognizeSelector: -- abort",
              receiver,
              className);
    }
    
    // The point of no return.
    kill(getpid(), 9);
}

```

该文章为记录本人的学习路程，希望能够帮助大家！！！


