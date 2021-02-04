---
title: KVC底层原理--YYModel简述
date: 2019-07-15 15:16:18
author: Vincent
categories: 
- 底层原理
- 源码分析
tags: 
- KVC
- 源码分析
- 底层原理
---


![KVC底层原理](https://upload-images.jianshu.io/upload_images/5741330-29717fbfde50dc23.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


YYModel的作用就是字典转模型，在了解YYModel前，我们先了解下KVC的知识。
> KVC：也称之键值编码，是一种采用了`NSKeyValueCoding`协议的对象（直接或间接继承NSObject时会为基本方法提供默认实现）通过间接访问其属性的机制，也就是符合键值编码，对象可通过字符串参数来简单而统一的消息对其属性进行寻址，例如`valueForKey: `和 `setValue:forKey:`。在一些特殊情况下，KVC还可以简化代码。[官方文档直通车](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/KeyValueCoding/index.html#//apple_ref/doc/uid/10000107-SW1)
<!--more-->

#### KVC底层原理
举例：用键值编码实现数据源方法
```
- (id)tableView:(NSTableView *)tableview objectValueForTableColumn:(id)column row:(NSInteger)row
{
    return [[self.people objectAtIndex:row] valueForKey:[column identifier]];
}
```
或者我们常用的设置textFiled的placeholder颜色
```
[self.textFiled setValue:[UIColor greenColor] forKeyPath:@"_placeholderLabel.textColor"];
```
> 当调用协议的 getter（比如`valueForKey:`）， 默认实现根据`Accessor Search Patterns`中描述的规则确定为指定键提供值的特定访问器方法或实例变量. 如果返回值不是对象，getter会使用这个值初始一个NSNumber对象或NSValue（对于结构体）对象替代。同样setter（比如`setValue:forKey:`）通过特定键或访问方法或实例变量时确定的数据类型，如果数据类型不是对象，setter首先会向传入的值对象发送适当的 <type>Value 消息(intValue)提取基础数据并存储该数据。仅限__Objective-C__，因为__Swift__的所有属性都是对象。

自动包装和解包不仅限于 NSPoint，NSRange，NSRect，和 NSSize. 也可以是 NSValue 对象。举例：
```
typedef struct {
    float x, y, z;
} ThreeFloats;
 
@interface MyClass
@property (nonatomic) ThreeFloats threeFloats;
@end

```
使用KVC获取myClass的`threeFloats`：默认调用threeFloats的`getter`，然后将返回值包装在NSValue中返回。
```
NSValue* result = [myClass valueForKey:@"threeFloats"];
```
当然，我们也可以使用KVC来设置threeFloats的值
```
ThreeFloats floats = {1., 2., 3.};
NSValue* value = [NSValue valueWithBytes:&floats objCType:@encode(ThreeFloats)];
[myClass setValue:value forKey:@"threeFloats"];
```

##### KVC取值和赋值的过程

首先，我们先根据官方文档找到`Accessor Search Patterns`来查看根据输入`key`的搜索流程。下面是官方文档的部分说明，具体参考官方文档

![官方文档](https://upload-images.jianshu.io/upload_images/5741330-1e29125d421d21c0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

说的就是在getter的时候，先按照`get<Key>, <key>, is<Key>, or _<key>`的顺序来查找，找到了就直接调用，然后判断收到的属性值是一个对象指针直接返回，该值是NSNumber支持的标量类型则用包装NSNumber返回，否则包装成 NSValue 对象返回。如果没有找到则在实例中搜索`countOf<Key>, objectIn<Key>AtIndex:（对应于NSArray基本方法）和<key>AtIndexes:（对应于NSArray的objectsAtIndexes）`，如果找到则创建一个响应所有NSArray方法的集合代理对象并返回，如果还是没有找到则查找`countOf<Key>, enumeratorOf<Key>和memberOf<Key>:（对应于 NSSet 类的基本方法）`，如果三个方法都找到，会创建一个响应所有NSSet方法的集合代理对象并返回该方法。如果还没找到并且接收者对象的`accessInstanceVariablesDirectly`（是否开启间接访问）返回 YES，则会顺序查找`_<key>, _is<key>, <key>, is<key>`，如果找到了则返回相应的值（对象指针、NSNumber或者NSValue），如果最后还是没找到则会执行`valueForUndefindKey:`，默认抛出异常，NSObject子类可以重写。

在setter的时候也是顺序查找访问方法名` set<key>, _set<key>`，找到了就则使用输入值执行，没找到且`accessInstanceVariablesDirectly `返回YES，则按照顺序查找`_<key>, _is<key>, <key>, is<key>`，如果找到了就执行没找到则执行`setValue:forUndefinedKey:`抛出未定义key的异常，同样NSObject子类可以重写。

当没有对异常进行处理的话，不出意外会崩溃。

> 在赋值和取值过程中，当value为空的时候，会执行`setNilValueForKey `，如果Key值不存在则执行`setValue:forUndefinedKey`。
KVC也可以用于验证key或key-path的方法，我们也可以为属性提供验证方法。在响应`validateValue:forKey:error:`方法的时候，会查找`valudate<Key>:error:`是否实现，如果实现了则根据实现方法的自定义逻辑返回YES或者NO，如果没实现则系统默认返回YES，NSError用来返回error信息。

##### KVC异常处理
那在实际运用的时候万一出现意外，要怎么规避呢？可以在NSObject的分类做下相应的处理（OC的对象几乎都可以追溯到NSObject）。
```
// .h文件代码
- (void)k_setValue:(nullable id)value forKey:(NSString *)key;
- (nullable id)k_valueForKey:(NSString *)key;
// ------------------华丽的分割线----------------------
// .m文件代码
- (void)k_setValue:(id)value forKey:(NSString *)key {
    NSError  *error;
    BOOL validate = [self validateValue:&value forKey:key error:&error];
    NSArray *arr = [self getIvarListName];
    if (validate) {
        if ([arr containsObject:key]) {
            [self setValue:value forKey:key];
        }else{
            NSLog(@"%@ 不存在变量",key);
        }
    }
}

- (nullable id)k_valueForKey:(NSString *)key {
    if (key == nil || key.length == 0) {
        NSLog(@"key为nil或者空值");
        return nil;
    }
    NSArray *arr = [self getIvarListName];
    if ([arr containsObject:key]) {
        return [self valueForKey:key];
    }
    NSLog(@"%@ 不存在变量",key);
    return nil;
}

- (BOOL)validateValue:(inout id  _Nullable __autoreleasing *)ioValue forKey:(NSString *)inKey error:(out NSError * _Nullable __autoreleasing *)outError {
    if (*ioValue == nil || inKey == nil || inKey.length == 0) {
        NSLog(@"value 可能为nil  或者key为nil或者空值");
        return NO;
    }
    return YES;
}

- (NSMutableArray *)getIvarListName {
    NSMutableArray *mulArr = [NSMutableArray arrayWithCapacity:1];
    unsigned int count = 0;
    Ivar *ivars = class_copyIvarList([self class], &count);
    for (int i = 0; i<count; i++) {
        Ivar ivar = ivars[i];
        const char *ivarNameChar = ivar_getName(ivar);
        NSString *ivarName = [NSString stringWithUTF8String:ivarNameChar];
        NSLog(@"ivarName == %@",ivarName);
        [mulArr addObject:ivarName];
    }
    free(ivars);
    return mulArr;
}
```
记得添加`#import <objc/runtime.h>`，在调用`k_setValue forKey `或者`k_valueForKey`的时候会自动检测value和key是否有效值，如果不是有效值会抛出

##### KVC使用
-   字典的使用
```
NSDictionary* dict = @{
                           @"oneString":@"one",
                           @"num":@123,
                           @"list":@[@1, @2, @3]
                           };
    Person *p = [[Person alloc] init];
    // 字典转模型
    [p setValuesForKeysWithDictionary:dict];
    NSLog(@"%@--%d---%@",p.oneString, p.num, p.list);
    // 键数组转模型到字典
    NSArray *array = @[@"oneString",@"num"];
    NSDictionary *dic = [p dictionaryWithValuesForKeys:array];
    NSLog(@"%@",dic);
```
打印结果如下：
```
one--123---(
    1,
    2,
    3
)

{
    num = 123;
    oneString = one;
}
```

- KVC消息传递
```
    NSArray *array = @[@"One",@"Two",@"Three",@"HH"];
    NSArray *lenStr= [array valueForKeyPath:@"length"];
    NSLog(@"%@",lenStr);// 消息从array传递给了string
    NSArray *lowStr= [array valueForKeyPath:@"lowercaseString"];
    NSLog(@"%@",lowStr);
    // 也支持聚合操作符
    int count = [[array valueForKeyPath:@"@count.length"] intValue];
    NSLog(@"%d", count);
```
打印结果：
```
(
    3,
    3,
    5,
    2
)

(
    one,
    two,
    three,
    hh
)

 4
```
- 补充
当然了，聚合操作符还有很多，列举一些可能用到的：
__@avg：__通过right-key-path读取集合中每个元素读取属性，并转换为double类型(nil用0代替)，计算他们的平均值，然后返回NSNumber。
__@ sum：__原理同 '@avg'，返回所有元素的和。
__@ count：__以NSNumber实例返回集合中所有对象的数量，如果有right-key-path忽略。
__@ max：__通过right-key-path在集合中查找并返回最大的元素。查找会使用由Foundation类(例如 NSNumber) 定义的compare: 方法，所以right-key-path指定的属性必须是一个对这个消息有意义的响应，查找忽略集合中的nil值。
__@min：__原理同 '@max'，返回最小的元素。
下面的也可以对集合进行操作
__@distinctUnionOfObjects：__返回对right-key-path指定属性进行合并操作后的去重数组。
__@ unionOfObjects：__和distinctUnionOfObjects行为相似, 但是不会删除重复对象。
__@ distinctUnionOfSets：__和distinctUnionOfObjects行为相似获取交集。

#### 键值模型--YYModel
> 字典转模型的大概流程是分析对象，获取到对象的所有ivar，并将ivar一一赋值。

首先，我们进入YYModel的字典转模型入口`yy_modelWithDictionary`、`modelWithJSON`方法，部分代码展示：
```
+ (instancetype)yy_modelWithDictionary:(NSDictionary *)dictionary {
    if (!dictionary || dictionary == (id)kCFNull) return nil;
    if (![dictionary isKindOfClass:[NSDictionary class]]) return nil;
    
    Class cls = [self class];
    // 分析cls
    _YYModelMeta *modelMeta = [_YYModelMeta metaWithClass:cls];
    if (modelMeta->_hasCustomClassFromDictionary) {
        cls = [cls modelCustomClassForDictionary:dictionary] ?: cls;
    }
    
    NSObject *one = [cls new];
    if ([one yy_modelSetWithDictionary:dictionary]) return one;
    return nil;
}
```
在`metaWithClass:`方法内部先用Core Foundation的CFMutableDictionaryRef进行缓存已经处理过的key-value，以备下次直接使用，并用dispatch_semaphore_t来确保字典的读取安全。如果没有该缓存或者不需要更新则进行创建`_YYModelMeta`对象，`_YYModelMeta`首先对黑名单和白名单进行处理，再次判断`modelContainerPropertyGenericClass`是否需要对特殊属性进行替换（按照不同的类型进行遍历 ），然后开始遍历自己和父类所有的属性和方法，生成与数据源相对应的字典映射（具体可网上查找），接着处理_YYModelPropertyMeta，判断是NSString、NSArray，一切准备好后开始执行`yy_modelSetWithDictionary:`方法响应`CFArrayApplyFunction`，再次执行`ModelSetWithPropertyMetaArrayFunction`，获取到的value传入`ModelSetValueForProperty`中，通过objc_msgSend发送`meta->_setter`来完成属性值的设置。
```
switch (meta->_nsType) {
                case YYEncodingTypeNSString:
                case YYEncodingTypeNSMutableString: {
                    if ([value isKindOfClass:[NSString class]]) {
                        if (meta->_nsType == YYEncodingTypeNSString) {
                            ((void (*)(id, SEL, id))(void *) objc_msgSend)((id)model, meta->_setter, value);
                        } else {
                            ((void (*)(id, SEL, id))(void *) objc_msgSend)((id)model, meta->_setter, ((NSString *)value).mutableCopy);
                        }
                    } else if ([value isKindOfClass:[NSNumber class]]) {
                        ((void (*)(id, SEL, id))(void *) objc_msgSend)((id)model,
                                                                       meta->_setter,
                                                                       (meta->_nsType == YYEncodingTypeNSString) ?
                                                                       ((NSNumber *)value).stringValue :
                                                                       ((NSNumber *)value).stringValue.mutableCopy);
                    } else if ([value isKindOfClass:[NSData class]]) {
                        NSMutableString *string = [[NSMutableString alloc] initWithData:value encoding:NSUTF8StringEncoding];
                        ((void (*)(id, SEL, id))(void *) objc_msgSend)((id)model, meta->_setter, string);
                    } else if ([value isKindOfClass:[NSURL class]]) {
                        ((void (*)(id, SEL, id))(void *) objc_msgSend)((id)model,
                                                                       meta->_setter,
                                                                       (meta->_nsType == YYEncodingTypeNSString) ?
                                                                       ((NSURL *)value).absoluteString :
                                                                       ((NSURL *)value).absoluteString.mutableCopy);
                    } else if ([value isKindOfClass:[NSAttributedString class]]) {
                        ((void (*)(id, SEL, id))(void *) objc_msgSend)((id)model,
                                                                       meta->_setter,
                                                                       (meta->_nsType == YYEncodingTypeNSString) ?
                                                                       ((NSAttributedString *)value).string :
                                                                       ((NSAttributedString *)value).string.mutableCopy);
                    }
                } break;
```

该文章为记录本人的学习路程，希望能够帮助大家，知识共享，共同成长，共同进步！！！

