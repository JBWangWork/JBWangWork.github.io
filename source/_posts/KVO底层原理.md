---
title: KVO底层原理—利用Runtime自定义KVO
date: 2019-08-16 16:46:44
author: Vincent
categories: 
- 底层原理
- 源码分析
tags: 
- KVO
- 源码分析
- 底层原理
---


![KVO底层原理—利用Runtime自定义KVO](https://upload-images.jianshu.io/upload_images/5741330-7fef9b095947a222.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


> KVO：Key-value observer，也就是键值观察，是Objective-C对观察者模式的实现，每当被观察对象的某个属性值发生改变时，注册的观察者便能得到通知。
当然想了解KVO，还要先对KVC有所了解：[KVC底层原理](https://www.jianshu.com/p/cce3a7b99c84)，本文利用Runtime实现自定义KVO，如果对Runtime不熟悉可以先了解下前几篇文章：[Runtime底层原理](https://www.jianshu.com/p/1ddd15e47343)。[KVO-官网直通车](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/KeyValueObserving/KeyValueObserving.html)

先简单介绍一下KVO使用：
- 添加观察：`addObserver:self forKeyPath：options:context:`
- 观察回调：`observeValueForKeyPath：ofObject ：change:  context:`
- 移除观察：`removeObserver: forKeyPath:`

> TIP：建议KVO还是手动添加移除。如果没有移除观察，会有隐藏奔溃隐患（单例），比如当观察者析构时不会自动移除，被观察对象继续发送消息, 像发送一个消息给已经释放的对象, 触发exception。

#### KVO原理：
KVO默认观察setter，使用`isa-swizzling`来实现自动键值观察，也就是被观察对象的isa会被修改，指向一个动态生成的子类NSKVONotifying_xxxx（isa在移除观察者之后复原，动态生成的类不会被移除），但是通过`object_getClass`获取的还是原来的类，该子类重写了观察对象的setter方法，还有`class`、`dealloc`方法和`_isKVOA`标识，并在重写setter方法中调用`– willChangeValueForKey`和`– didChangeValueForKey`，然后向父类发送消息。如果`automaticallyNotifiesObserversForKey`返回NO的时候可以手动观察
- 动态生成子类： NSKVONotifying_xxxx，用原来的类名做后缀
- 重写观察对象的setter，`class`、`dealloc`方法和`_isKVOA`标识
- 在重写setter方法中调用 – willChangeValueForKey和 – didChangeValueForKey
- 向父类发送消息

#### 自定义KVO
知道了KVO的原理后我们利用Runtime进行验证并自定义KVO的实现，在实现了系统KVO的功能基础上还__添加了自动移除观察者机制、监听利用block回调等__。

利用LLDB查看isa的指针，再利用Runtime查看添加观察前后的变化，可以通过下面的方法对原来的类和新增的NSKVONotifying_xxxx类进行对比
```
// 遍历方法 -- 判断imp指针是否改变也就是重写
- (void)getClassAllMethod:(Class)cls {
    if (!cls) return;
    unsigned int count = 0;
    Method *methodList = class_copyMethodList(cls, &count);
    for (int i = 0; i<count; i++) {
        Method method = methodList[i];
        SEL sel = method_getName(method);
        IMP imp = class_getMethodImplementation(cls, sel);
        NSLog(@"%@ --- %p",NSStringFromSelector(sel), imp);
    }
    free(methodList);
}

// 遍历属性
- (void)getClassProperty:(Class)cls {
    if (!cls) return;
    //获取类中的属性列表
    unsigned int propertyCount = 0;
    objc_property_t * properties = class_copyPropertyList(cls, &propertyCount);
    for (int i = 0; i<propertyCount; i++) {
        NSLog(@"属性的名称为 : %s",property_getName(properties[i]));
        /**
         特性编码 具体含义
         R readonly
         C copy
         & retain
         N nonatomic
         G(name) getter=(name)
         S(name) setter=(name)
         D @dynamic
         W weak
         P 用于垃圾回收机制
         */
        NSLog(@"属性的特性字符串为: %s",property_getAttributes(properties[i]));
    }
    //释放属性列表数组
    free(properties);
}

// 遍历变量
- (void)getClassAllIvar:(Class)cls {
    if (!cls) return;
    unsigned int count = 0;
    Ivar *ivarList = class_copyIvarList(cls, &count);
    for (int i = 0; i<count; i++) {
        Ivar ivar = ivarList[i];
        NSLog(@"%s",ivar_getName(ivar));
    }
    free(ivarList);
}

// 遍历类以及子类
- (void)getClasses:(Class)cls {
    if (!cls) return;
    // 注册类的总数
    int count = objc_getClassList(NULL, 0);
    // 创建一个数组，其中包含给定对象
    NSMutableArray *mArr = [NSMutableArray arrayWithObject:cls];
    // 获取所有已注册的类
    Class *classes = (Class *)malloc(sizeof(Class)*count);
    objc_getClassList(classes, count);
    for (int i = 0; i < count; i++) {
        if (cls == class_getSuperclass(classes[i])) {
            [mArr addObject:classes[i]];
        }
    }
    free(classes);
    NSLog(@"classes --- %@", mArr);
}
```

经过验证后开始自定义KVO实现系统功能，并额外加上自定义的一些功能。先添加用来保存KVO信息的Info类`VKVOInfo `用来保存信息，还有一个扩展`NSObject+VKVO`，主要实现系统的原有功能，再添加自定义的一些方法，比如自动移除观察者等。
- 首先先动态生成子类，并添加`setter`，`class`、`dealloc`方法
```
#pragma mark - 动态生成子类
- (Class)createChildClass:(NSString *)keyPath {
    NSString *oldName = NSStringFromClass([self class]);
    NSString *newName = [NSString stringWithFormat:@"%@%@", kVKVOPrefix, oldName];
    Class newClass = NSClassFromString(newName);
    // 如果内存不存在,创建生成新的类，防止重复创建生成新类
    if (newClass) return newClass;
    
    newClass = objc_allocateClassPair([self class], newName.UTF8String, 0);
    objc_registerClassPair(newClass);
    
    // 添加class方法
    SEL classSEL = NSSelectorFromString(@"class");
    Method classMethod = class_getInstanceMethod([self class], classSEL);
    const char *classType = method_getTypeEncoding(classMethod);
    class_addMethod(newClass, classSEL, (IMP)v_class, classType);
    
    // 添加setter方法
    SEL setterSEL = NSSelectorFromString(setterForGetter(keyPath));
    Method setterMethod = class_getInstanceMethod([self class], setterSEL);
    const char *setterType = method_getTypeEncoding(setterMethod);
    class_addMethod(newClass, setterSEL, (IMP)v_setter, setterType);
    
    // 添加dealloc方法
    SEL deallocSEL = NSSelectorFromString(@"dealloc");
    Method deallocMethod = class_getInstanceMethod([self class], deallocSEL);
    const char *deallocType = method_getTypeEncoding(deallocMethod);
    class_addMethod(newClass, deallocSEL, (IMP)v_dealloc, deallocType);
    
    return newClass;
}
```

- 把isa指针指向动态生成的KVONotifying子类（Person类会动态生成KVONotifying_Person）
```
object_setClass(self, newClass);
```

- 保存KVO的信息
```
VKVOInfo *KVOInfo = [[VKVOInfo alloc] initWithObserver:observer forKeyPath:keyPath options:options handleBlock:handleBlock];
    NSMutableArray *infoArr = objc_getAssociatedObject(self, (__bridge const void * _Nonnull)(kVKVOAssiociateKey));
    if (!infoArr) {
        infoArr = [NSMutableArray arrayWithCapacity:1];
        objc_setAssociatedObject(self, (__bridge const void * _Nonnull)(kVKVOAssiociateKey), infoArr, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
    }
    [infoArr addObject:KVOInfo];
```
下面是部分重要代码：在`setter`方法中，先将消息发送给原来的类，再利用block响应回调（这里也可以添加判断，利用block回调或者设置代理），也可以添加一些自定义的方法，比如去掉NSKeyValueObservingOptions参数。
```
static void v_setter(id self, SEL _cmd, id newValue) {
    NSString *keyPath = getterForSetter(NSStringFromSelector(_cmd));
    id oldValue = [self valueForKey:keyPath];
    /// Specifies the superclass of an instance.
    struct objc_super v_objc_super = {
        .receiver = self,
        .super_class = class_getSuperclass(object_getClass(self))
    };
    // 消息转发给父类
    void (*v_msgSendSuper)(void *, SEL, id) = (void *)objc_msgSendSuper;
    v_msgSendSuper(&v_objc_super, _cmd, newValue);
    
    // 响应回调
    NSMutableArray *infoArr = objc_getAssociatedObject(self, (__bridge const void * _Nonnull)(kVKVOAssiociateKey));
    for (VKVOInfo *info in infoArr) {
        if ([info.keyPath isEqualToString:keyPath]) {
            dispatch_async(dispatch_get_global_queue(0, 0), ^{
                if (info.options & NSKeyValueObservingOptionNew) {
                    if (info.handleBlock) {
                        info.handleBlock(info.observer, info.keyPath, info.options, newValue, oldValue);
                    }
                }
//                SEL obserSEL = @selector(observeValueForKeyPath:ofObject:change:context:);
//                void (*v_objc_msgSend)(id, SEL, id, id, id, void *) = (void *)objc_msgSend;
//                Class supperClass = (object_getClass(self));
//                v_objc_msgSend(info.observer, obserSEL, keyPath, supperClass, @{keyPath:newValue}, NULL);
            });
        }
    }
}
```
这里是[Demo地址：https://github.com/JBWangWork/VCustomKVO](https://github.com/JBWangWork/VCustomKVO)，本Demo已更新，去掉了options和context参数（系统context可以起到快速定位观察键的作用）。本Demo只适用于学习KVO底层原理。

该文章为记录本人的学习路程，也希望能够帮助大家，知识共享，共同成长，共同进步！！！

