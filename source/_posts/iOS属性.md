---
title: iOS中实例变量、成员变量和属性变量的区别
date: 2019-06-23 11:31:35
author: Vincent
categories: 
- 底层原理
tags: 
- 底层原理
---



![实例变量、成员变量和属性变量的区别](https://upload-images.jianshu.io/upload_images/5741330-99ec68ecf0493fad.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


作为iOS开发，会经常听到成员变量、实例变量和属性；那他们有什么区别吗？

#### 实例变量
__实例变量:__ class类进行实例化出来的对象为实例对象；比如:
```
Person *p = [Person new];
```

#### 成员变量
__成员变量:__ 在`{ }`中所声明的变量都是成员变量(实例变量是一种特殊的成员变量)。其中的hell、btn也是实例对象，id是一种特殊的class，是OC特有的对象。成员变量是私有变量，外部不会获取到。
```
@interface Person : NSObject{
    @public
    NSString *myName; //成员
    id hell; // id - > class
    UIButton *btn;
    int age;
}
```
#### 属性变量
__属性变量:__ 属性是与其他对象交互的变量，会生成默认的setter和getter方法。苹果早期的编译器是GCC，后来发展到LLVM，LLVM在没有匹配实例变量的属性时会自动创建一个带下划线的成员变量。__注意：分类中添加的属性是不会自动生成setter和getter方法的，必须要手动添加__。如果已经手动实现了get和set方法的话Xcode不会再自动生成带有下划线的私有成员变量了，因为xCode自动生成成员变量的目的就是为了根据成员变量而生成get/set方法的，但是如果get和set方法缺一个的话都会生成带下划线的变量。

##### 给分类添加属性
.h文件
```
#import <Foundation/Foundation.h>

@interface NSObject (Person)

@property (nonatomic, copy) NSString *name;

@end
```
.m文件
```
#import "NSObject+Person.h"
#import <objc/runtime.h> /*或者 #import <objc/message.h>*/
static NSString *nameKey = @"nameKey"; // name的key

@interface NSObject ()

@end

@implementation NSObject (Person)

/**
 setter方法
 */
- (void)setName:(NSString *)name {
    objc_setAssociatedObject(self, &nameKey, name, OBJC_ASSOCIATION_COPY);
}

/**
 getter方法
 */
- (NSString *)name {
    return objc_getAssociatedObject(self, &nameKey);
}
@end
```
使用：
```
- (void)viewDidLoad {
    NSObject *objc = [[NSObject alloc] init];
    objc.name = @"Vincent";
    NSLog(@"%@", objc.name);
}
```

##### `@property`和`@synthesize`
`@synthesize`让编译器自动生成setter和getter，可以制定属性对应的成员变量。

在Xcode4.4版本之前`@property`和`@synthesize`的功能是独立分工的：
1. `@property`的作用是：自动的生成成员变量set/get方法的声明如代码：
     ```
        @property int age; // 它的作用和下面两行代码的作用一致
        - (void)setAge:(int)age;
        - (int)age;
      ```
   __注意：属性名称不要加前缀下划线，否则生成的get/set方法中也会有下划线___ 
2. `@synthesize`的作用是实现`@property`定义的方法代码如：
 
      ```
        @synthesize age
      ```

      将`@property`中定义的属性自动生成get/set的实现方法而且默认访问成员变量age，如果指定访问成员变量_age的话代码如：
      ```       
       @synthesize age = _age；
      ```
      把@property中声明的age成员变量生成get/set实现方法，并且在实现方法内部访问_age这个成员变量，也就意味着给成员_age赋值。
 
 > 注意：访问成员变量 _age 如果在.h文件中没有定义_age成员变量的话，就会在.m文件中自动生成@private类型的成员变量_age


该文章为记录本人的学习路程，希望能够帮助大家！！！

