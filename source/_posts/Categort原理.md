---
title: Category实现原理--源码分析
date: 2019-07-27 15:24:04
author: Vincent
categories: 
- 底层原理
- 源码分析
tags: 
- Categort
- 源码分析
- 底层原理
---


![Category实现原理](https://upload-images.jianshu.io/upload_images/5741330-b9c059d5a3119823.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


> 在Objective-C 2.0中新增的Category可以动态地为已有类添加新的对象方法、类方法、协议、和属性。注：这里的属性只会生成set/get方法的声明，并不会自动生成成员变量（分类是在运行时才去加载，对象的内存布局已经确定，无法在程序运行时将分类的成员变量添加到实例对象的结构体中），可以利用关联对象来实现。

<!--more-->
在Runtime层，Category用结构体category_t表示，`name`：类的名字，`cls`：类，`instanceMethods`：Category中所有给类添加的实例方法的列表，`classMethods`：Category中所有添加的类方法的列表，`protocols`：Category实现的所有协议的列表，`instanceProperties`：Category中添加的所有属性。我们在分类中声明的方法、属性等都会存在对应的字段中，有多少个分类就会有多少的category_t结构体。
```
struct category_t {
    const char *name;    // 类的名字
    classref_t cls;    // 类
    struct method_list_t *instanceMethods;
    struct method_list_t *classMethods;
    struct protocol_list_t *protocols;
    struct property_list_t *instanceProperties;
    // Fields below this point are not always present on disk.
    struct property_list_t *_classProperties;

    method_list_t *methodsForMeta(bool isMeta) {
        if (isMeta) return classMethods;
        else return instanceMethods;
    }

    property_list_t *propertiesForMeta(bool isMeta, struct header_info *hi);
};
```

#### 源码分析Category加载过程
首先从镜像加载开始，`_objc_init`通过`map_images`加载并缓存所有镜像文件，比如类、方法编号等信息加载。`map_images`内部执行`_getObjc2CategoryList`来获取category_t数组，`addUnattachedCategoryForClass`把类和Category做一个关联映射，然后执行`remethodizeClass `。
```
/ Discover categories. 
    for (EACH_HEADER) {
        category_t **catlist = 
            _getObjc2CategoryList(hi, &count);
        bool hasClassProperties = hi->info()->hasCategoryClassProperties();

        for (i = 0; i < count; i++) {
            category_t *cat = catlist[i];
            Class cls = remapClass(cat->cls);

            if (!cls) {
                // Category's target class is missing (probably weak-linked).
                // Disavow any knowledge of this category.
                catlist[i] = nil;
                if (PrintConnecting) {
                    _objc_inform("CLASS: IGNORING category \?\?\?(%s) %p with "
                                 "missing weak-linked target class", 
                                 cat->name, cat);
                }
                continue;
            }

            // Process this category. 
            // First, register the category with its target class. 
            // Then, rebuild the class's method lists (etc) if 
            // the class is realized. 
            bool classExists = NO;
            if (cat->instanceMethods ||  cat->protocols  
                ||  cat->instanceProperties) 
            {
                addUnattachedCategoryForClass(cat, cls, hi);
                if (cls->isRealized()) {
                    remethodizeClass(cls);
                    classExists = YES;
                }
                if (PrintConnecting) {
                    _objc_inform("CLASS: found category -%s(%s) %s", 
                                 cls->nameForLogging(), cat->name, 
                                 classExists ? "on existing class" : "");
                }
            }

            if (cat->classMethods  ||  cat->protocols  
                ||  (hasClassProperties && cat->_classProperties)) 
            {
                addUnattachedCategoryForClass(cat, cls->ISA(), hi);
                if (cls->ISA()->isRealized()) {
                    remethodizeClass(cls->ISA());
                }
                if (PrintConnecting) {
                    _objc_inform("CLASS: found category +%s(%s)", 
                                 cls->nameForLogging(), cat->name);
                }
            }
        }
    }
```

`remethodizeClass `通过`attachCategories `，将方法列表、属性列表和协议列表写入rw中（这里分类和类都是一样的）。
```
// Attach method lists and properties and protocols from categories to a class.
// Assumes the categories in cats are all loaded and sorted by load order, 
// oldest categories first.
static void 
attachCategories(Class cls, category_list *cats, bool flush_caches)
{
    if (!cats) return;
    if (PrintReplacedMethods) printReplacements(cls, cats);

    bool isMeta = cls->isMetaClass();

    // fixme rearrange to remove these intermediate allocations
    method_list_t **mlists = (method_list_t **)
        malloc(cats->count * sizeof(*mlists));
    property_list_t **proplists = (property_list_t **)
        malloc(cats->count * sizeof(*proplists));
    protocol_list_t **protolists = (protocol_list_t **)
        malloc(cats->count * sizeof(*protolists));

    // Count backwards through cats to get newest categories first
    int mcount = 0;
    int propcount = 0;
    int protocount = 0;
    int i = cats->count;
    bool fromBundle = NO;
    while (i--) {
        auto& entry = cats->list[i];

        method_list_t *mlist = entry.cat->methodsForMeta(isMeta);
        if (mlist) {
            mlists[mcount++] = mlist;
            fromBundle |= entry.hi->isBundle();
        }

        property_list_t *proplist = 
            entry.cat->propertiesForMeta(isMeta, entry.hi);
        if (proplist) {
            proplists[propcount++] = proplist;
        }

        protocol_list_t *protolist = entry.cat->protocols;
        if (protolist) {
            protolists[protocount++] = protolist;
        }
    }

    auto rw = cls->data();

    prepareMethodLists(cls, mlists, mcount, NO, fromBundle);
    rw->methods.attachLists(mlists, mcount);
    free(mlists);
    if (flush_caches  &&  mcount > 0) flushCaches(cls);

    rw->properties.attachLists(proplists, propcount);
    free(proplists);

    rw->protocols.attachLists(protolists, protocount);
    free(protolists);
}
```

通过`attachLists `把Category的实例方法列表、协议列表以及属性列表附加到原来类的相应列表中。以方法为例，先判断分类的方法列表hasArray()，然后利用散列表把分类的方法列表和原来类中的方法列表进行合并，如果分类和原来的类中有同名方法，就把分类的方法放在原来的类的前面，合并后copy到新的方法列表，如果原来类没有同名方法则放在方法列表后面，所以__Category并没有覆盖原来类的方法__。举个例子：如果分类和原来的类都有func方法，那么Category附加后，类的方法列表里会有两个func方法并且Category的func方法在前面，原来的类的func方法在后面，运行时在查找方法的时候是顺着方法列表的顺序查找的，当查找func方法时优先返回Category的imp，而不会继续查找下去，这就是__不会执行原来类的同名方法而执行分类的方法的原因__。
```
void attachLists(List* const * addedLists, uint32_t addedCount) {
        if (addedCount == 0) return;

        if (hasArray()) {
            // many lists -> many lists
            uint32_t oldCount = array()->count;
            uint32_t newCount = oldCount + addedCount;
            setArray((array_t *)realloc(array(), array_t::byteSize(newCount)));
            array()->count = newCount;
            memmove(array()->lists + addedCount, array()->lists, 
                    oldCount * sizeof(array()->lists[0]));
            memcpy(array()->lists, addedLists, 
                   addedCount * sizeof(array()->lists[0]));
        }
        else if (!list  &&  addedCount == 1) {
            // 0 lists -> 1 list
            list = addedLists[0];
        } 
        else {
            // 1 list -> many lists
            List* oldList = list;
            uint32_t oldCount = oldList ? 1 : 0;
            uint32_t newCount = oldCount + addedCount;
            setArray((array_t *)malloc(array_t::byteSize(newCount)));
            array()->count = newCount;
            if (oldList) array()->lists[addedCount] = oldList;
            memcpy(array()->lists, addedLists, 
                   addedCount * sizeof(array()->lists[0]));
        }
    }
```

#### Category的load方法调用栈

进入开源objc源码`_objc_init`开始，进入`load_images `，`prepare_load_methods`做好准备工作后，`call_load_methods`开始调用。
```
/***********************************************************************
* load_images
* Process +load in the given images which are being mapped in by dyld.
*
* Locking: write-locks runtimeLock and loadMethodLock
**********************************************************************/
extern bool hasLoadMethods(const headerType *mhdr);
extern void prepare_load_methods(const headerType *mhdr);

void
load_images(const char *path __unused, const struct mach_header *mh)
{
    // Return without taking locks if there are no +load methods here.
    if (!hasLoadMethods((const headerType *)mh)) return;

    recursive_mutex_locker_t lock(loadMethodLock);

    // Discover load methods
    {
        mutex_locker_t lock2(runtimeLock);
        prepare_load_methods((const headerType *)mh);
    }

    // Call +load methods (without runtimeLock - re-entrant)
    call_load_methods();
}
```
##### prepare_load_methods准备阶段
进入`prepare_load_methods`方法内部，`_getObjc2NonlazyClassList`取出所有加载进去的类列表，然后开始遍历执行`schedule_class_load`，`schedule_class_load`内部先递归父类`schedule_class_load(cls->superclass);`，然后`add_class_to_loadable_list`把当前类的load方法加载到list中，这里可以发现，父类永远在子类的前面，所以__在加载类的load方法时先加载父类的load方法，再加载子类的load方法__。类列表加载完执行`_getObjc2NonlazyCategoryList`开始加载分类列表，按照编译顺序取出分类的数据再for循环执行`realizeClass(cls);`和`add_category_to_loadable_list(cat);`加载分类中的load到list。这里可以知道，__分类的load方法加载顺序就是谁先编译的，谁的load方法就被先加载。__可以写个demo看下。
```
/***********************************************************************
* prepare_load_methods
* Schedule +load for classes in this image, any un-+load-ed 
* superclasses in other images, and any categories in this image.
**********************************************************************/
// Recursively schedule +load for cls and any un-+load-ed superclasses.
// cls must already be connected.
static void schedule_class_load(Class cls)
{
    if (!cls) return;
    assert(cls->isRealized());  // _read_images should realize

    if (cls->data()->flags & RW_LOADED) return;

    // Ensure superclass-first ordering
    schedule_class_load(cls->superclass);

    add_class_to_loadable_list(cls);
    cls->setInfo(RW_LOADED); 
}


void prepare_load_methods(const headerType *mhdr)
{
    size_t count, i;

    runtimeLock.assertLocked();

    classref_t *classlist = 
        _getObjc2NonlazyClassList(mhdr, &count);
    for (i = 0; i < count; i++) {
        schedule_class_load(remapClass(classlist[i]));
    }

    category_t **categorylist = _getObjc2NonlazyCategoryList(mhdr, &count);
    for (i = 0; i < count; i++) {
        category_t *cat = categorylist[i];
        Class cls = remapClass(cat->cls);
        if (!cls) continue;  // category for ignored weak-linked class
        realizeClass(cls);
        assert(cls->ISA()->isRealized());
        add_category_to_loadable_list(cat);
    }
}
```
##### Load方法调用阶段

进入`call_load_methods`中，`objc_autoreleasePoolPush`压栈自动释放池，之后在do-while循环中先执行`call_class_loads `加载类的load方法，再执行`call_category_loads`加载分类的load方法，最后`objc_autoreleasePoolPop(pool);`出栈。`call_class_loads `和`call_category_loads`内部都是通过初始化一个指向当前类的load方法的指针来访问load方法。
```
/***********************************************************************
* call_load_methods
* Call all pending class and category +load methods.
* Class +load methods are called superclass-first. 
* Category +load methods are not called until after the parent class's +load.
* 
* This method must be RE-ENTRANT, because a +load could trigger 
* more image mapping. In addition, the superclass-first ordering 
* must be preserved in the face of re-entrant calls. Therefore, 
* only the OUTERMOST call of this function will do anything, and 
* that call will handle all loadable classes, even those generated 
* while it was running.
*
* The sequence below preserves +load ordering in the face of 
* image loading during a +load, and make sure that no 
* +load method is forgotten because it was added during 
* a +load call.
* Sequence:
* 1. Repeatedly call class +loads until there aren't any more
* 2. Call category +loads ONCE.
* 3. Run more +loads if:
*    (a) there are more classes to load, OR
*    (b) there are some potential category +loads that have 
*        still never been attempted.
* Category +loads are only run once to ensure "parent class first" 
* ordering, even if a category +load triggers a new loadable class 
* and a new loadable category attached to that class. 
*
* Locking: loadMethodLock must be held by the caller 
*   All other locks must not be held.
**********************************************************************/
void call_load_methods(void)
{
    static bool loading = NO;
    bool more_categories;

    loadMethodLock.assertLocked();

    // Re-entrant calls do nothing; the outermost call will finish the job.
    if (loading) return;
    loading = YES;

    void *pool = objc_autoreleasePoolPush();

    do {
        // 1. Repeatedly call class +loads until there aren't any more
        while (loadable_classes_used > 0) {
            call_class_loads();
        }

        // 2. Call category +loads ONCE
        more_categories = call_category_loads();

        // 3. Run more +loads if there are classes OR more untried categories
    } while (loadable_classes_used > 0  ||  more_categories);

    objc_autoreleasePoolPop(pool);

    loading = NO;
}
```
这里我们可以知道，__load方法的调用并不是消息发送objc_msgSend机制，而是直接找到类的load方法的地址，接下来调用类的load方法，然后再找到分类的load方法的地址，再去调用它。__

这里插一个题外话，我们平时在load方法内做交换方法的原因，一是load方法在main函数之前调用，执行比较早；二是load方法自动执行，不需要手动执行；三是唯一性，不用担心被紫烈覆盖。当然，这里也有很多坑
- 找到真正的方法归属--NSArray，__NSSArray
- 可能被主动调用--单例原则保证只执行一次
- 子类没有实现父类的方法，导致调用交换，会找父类，但是父类没有swizzling的方法，会崩溃--先尝试给自己添加要交换的方法：personInstanceMethod(SEL)->swiMethod(IMP)，然后再将父类的IMP给swizzle personInstanceMethod(imp)->swizzledSEL
- 交换没有实现的方法--添加一个老方法编号的实现（swiMethod）,把swiMethod的具体实现赋值一个空实现，防止递归。
- 交换类方法--类方法存在元类中。


该文章为记录本人的学习路程，希望能够帮助大家，知识共享，共同成长，共同进步！！！


