---
title: 多线程原理--NSOperation、NSOperationQueue
date: 2020-01-06 15:33:55
author: Vincent
categories: 
- 底层原理
- 源码分析
tags: 
- NSOperation
- 源码分析
- 底层原理
---


![多线程原理--NSOperation、NSOperationQueue](https://upload-images.jianshu.io/upload_images/5741330-d8c4046683af6dd4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


> NSOperation类是iOS2.0推出的，通过NSThread实现的,但是效率一般。从iOS4推出GCD时又重写了NSOperation和NSOperationQueue，NSOperation和NSOperationQueue分别对应GCD的任务和队列（[了解GCD直通车：https://www.jianshu.com/p/acc6e7bd6f10](https://www.jianshu.com/p/acc6e7bd6f10)），所以NSOPeration和NSOperationQueue是基于GCD更高一层的封装，而且完全地面向对象。但是比GCD更简单易用、代码可读性也更高。NSOperation和NSOperationQueue对比GCD会带来一点额外的系统开销，但是可以在多个操作Operation中添加附属。

#### 优点

- 可添加完成的代码块，在操作完成后执行。
- 添加操作之间的依赖关系，方便的控制执行顺序。
- 设定操作执行的优先级。
- 可以很方便的取消一个操作的执行。
- 使用 KVO 观察对操作执行状态的更改：isExecuteing、isFinished、isCancelled。

#### NSOperation、NSOperationQueue
NSOperation是一个和任务相关的抽象类，不具备封装操作的能力，必须使用其子类：NSInvocationOperation或者NSBlockOperation，当然也可以自定义子类,实现内部相应的⽅法，NSOperation实例在多线程上执行是线程安全的不需要添加额外的锁，不  必管理线程生命周期和同步等问题。NSInvocationOperation 和NSBlockOperation子类不同的是，因为NSInvocationOperation没有额外添加任务的方法，所以使用NSInvocationOperation创建的对象只会有一个任务，其次使用NSBlockOperation来执行任务切任务大于1的时候，系统可能会开辟新线程来异步执行任务。

NSOperationQueue有主队列和自定义队列（串行和并发），将NSOperation对象添加NSOperationQueue中，该NSOperationQueue对象从线程中拿取操作、以及分配到对应线程的工作都是由**系统处理**的。一般操作对象添加到NSOperationQueue之后，如果不存在依赖或者整个队列被暂停情况下通常短时间就开始运行。NSOperationQueue可以通过对象属性`suspended`来决定是否让队列暂时停止对任务的调度，或者`cancel、cancelAllOperations`方法来取消操作对象，不过使用这两个方法后操作对象无法恢复，操作时只会停止调度队列中操作对象（**注意：正在执行的操作依然会执行，无法取消。** ）， 且取消不可恢复。

首先创建一个NSOperation的子类（以NSInvocationOperation为例），再创建队列NSOperationQueue，最后将操作加入队列。这样我们就完成了多线程的操作。可以直接执行`start`方法，但不会开辟新线程去执行操作，而是在当前线程同步执行任务。这里注意，如果将操作添加到队列的同时再次执行`start`，这时会抛出异常，因为线程在创建后，开始进入Runnable就绪的状态，如果此时再次执行`start`会重复初始化操作。

```
    NSInvocationOperation *op = [[NSInvocationOperation alloc] initWithTarget:self selector:@selector(handleInvocation:) object:@"123"];

    // 将op加入到队列中
    NSOperationQueue *queue = [[NSOperationQueue alloc] init];
    [queue addOperation:op];
    [[NSOperationQueue mainQueue] addOperation:op];

    // 或者直接start
    [op start];
```

因为基于GCD更高一层的封装，NSOperation同样也具备线程之间的通讯以及控制并发数，举个简单的例子：
```
    NSOperationQueue *queue = [[NSOperationQueue alloc] init];
    queue.maxConcurrentOperationCount = 2;
    for (int i = 0; i<10; i++) {
        [queue addOperationWithBlock:^{
            [NSThread sleepForTimeInterval:2];
            NSLog(@"%d-%@",i,[NSThread currentThread]);
            [[NSOperationQueue mainQueue] addOperationWithBlock:^{
                NSLog(@"%d-%@--%@",i,[NSOperationQueue currentQueue],[NSThread currentThread]);
            }];
        }];
    }
```

由于设置了最大并发数`maxConcurrentOperationCount`为2，所以每2秒输出四个任务。

```
0-<NSThread: 0x600003f08180>{number = 4, name = (null)}
1-<NSThread: 0x600003f17bc0>{number = 5, name = (null)}
<NSOperationQueue: 0x7fce0de147f0>{name = 'NSOperationQueue Main Queue'} --<NSThread: 0x600003f5acc0>{number = 1, name = main}
<NSOperationQueue: 0x7fce0de147f0>{name = 'NSOperationQueue Main Queue'} --<NSThread: 0x600003f5acc0>{number = 1, name = main}
2-<NSThread: 0x600003f31e00>{number = 7, name = (null)}
3-<NSThread: 0x600003f08180>{number = 4, name = (null)}
<NSOperationQueue: 0x7fce0de147f0>{name = 'NSOperationQueue Main Queue'} --<NSThread: 0x600003f5acc0>{number = 1, name = main}
<NSOperationQueue: 0x7fce0de147f0>{name = 'NSOperationQueue Main Queue'} --<NSThread: 0x600003f5acc0>{number = 1, name = main}
... // 省略部分相似打印信息
9-<NSThread: 0x600003f08180>{number = 4, name = (null)}
8-<NSThread: 0x600003f17bc0>{number = 5, name = (null)}
<NSOperationQueue: 0x7fce0de147f0>{name = 'NSOperationQueue Main Queue'} --<NSThread: 0x600003f5acc0>{number = 1, name = main}
<NSOperationQueue: 0x7fce0de147f0>{name = 'NSOperationQueue Main Queue'} --<NSThread: 0x600003f5acc0>{number = 1, name = main}
```

当然我们也可以自定义子类，可能会添加到串行和并发队列的不同情况，需要重写不同的方法。TIP：并发操作时，需要自己创建自动释放池，因为异步操作无法访问主线程的自动释放池。经常通过cancelled属性检查方法是否取消，并且对取消做出响应。

#### 操作依赖

NSOperation、NSOperationQueue 最吸引人的地方是它能添加操作之间的依赖关系，可以使用依赖关系来控制操作间的启动顺序。当一个操作对象添加了依赖，被依赖的操作对象就会先执行，当被依赖的操作对象执行完才会当前的操作对象，通过操作依赖可以很方便的按照特定顺序控制操作之间的执行先后顺序。操作对象可以通过`addDependency`添加和`removeDependency `移除依赖。在添加线程对象NSOperationQueue之前进行依赖设置，操作对象会管理自己的依赖，因此在不相同队列中的操作对象可以建立依赖关系。

举例：现在有任务1、2、3，当任务1执行完毕后再执行任务2，任务2执行完毕后再执行任务3。

```
    NSBlockOperation *bo1 = [NSBlockOperation blockOperationWithBlock:^{
        [NSThread sleepForTimeInterval:0.5];
        NSLog(@"任务----1");
    }];
    
    NSBlockOperation *bo2 = [NSBlockOperation blockOperationWithBlock:^{
        [NSThread sleepForTimeInterval:0.5];
        NSLog(@"任务----2");
    }];
    
    NSBlockOperation *bo3 = [NSBlockOperation blockOperationWithBlock:^{
        [NSThread sleepForTimeInterval:0.5];
        NSLog(@"任务----3"); 
    }];
    
    // 建立依赖
    [bo2 addDependency:bo1];
    [bo3 addDependency:bo2];

    [self.queue addOperations:@[bo1,bo2,bo3] waitUntilFinished:YES];
    
    NSLog(@"执行结束");
```

这里`waitUntilFinished`如果设置为YES，则会堵塞当前线程，直到该操作结束。
最终打印效果总是：**任务1->任务2->任务3**。

```
 任务----1
 任务----2
 任务----3
 执行结束
```

#### 优先级、服务质量

> NSOperation 提供了queuePriority（优先级）属性，queuePriority属性适用于同一操作队列中的操作，不适用于不同操作队列中的操作。默认情况下，所有新创建的操作对象优先级都是NSOperationQueuePriorityNormal。但是我们可以通过`setQueuePriority`方法来改变当前操作在同一队列中的执行优先级。在iOS 8.0后，推出了服务质量，通过设置服务质量让系统优先处理某一个操作。

NSOperation优先级的枚举和Quality of Service枚举：
```
// NSOperation.h
typedef NS_ENUM(NSInteger, NSOperationQueuePriority) {
	NSOperationQueuePriorityVeryLow = -8L,
	NSOperationQueuePriorityLow = -4L,
	NSOperationQueuePriorityNormal = 0,
	NSOperationQueuePriorityHigh = 4,
	NSOperationQueuePriorityVeryHigh = 8
};

// ----------------------------
// NSObjCRuntime.h
typedef NS_ENUM(NSInteger, NSQualityOfService) {
//与用户交互的任务，这些任务通常跟UI级别的刷新相关，比如动画，这些任务需要在一瞬间完成.
    NSQualityOfServiceUserInteractive = 0x21,
// 由用户发起的并且需要立即得到结果的任务，比如滑动scroll view时去加载数据用于后续cell的显示，这些任务通常跟后续的用户交互相关，在几秒或者更短的时间内完成
    NSQualityOfServiceUserInitiated = 0x19,
// 一些可能需要花点时间的任务，这些任务不需要马上返回结果，比如下载的任务，这些任务可能花费几秒或者几分钟的时间
    NSQualityOfServiceUtility = 0x11,
// 一些可能需要花点时间的任务，这些任务不需要马上返回结果
    NSQualityOfServiceBackground = 0x09,
// 一些可能需要花点时间的任务，这些任务不需要马上返回结果
    NSQualityOfServiceDefault = -1
} API_AVAILABLE(macos(10.10), ios(8.0), watchos(2.0), tvos(9.0));
```
Utility 及以下的优先级会受到 iOS9 中低电量模式的控制，另外在没有用户操作时，90% 任务的优先级都应该在 Utility 之下。

一般对于添加到队列中的任务，当这个任务的所有依赖都已经完成时，任务通常会进入准备就绪状态来等待执行，这时该任务的开始执行顺序由任务的优先级决定。如果一个队列中既包含高优先级就绪状态的任务，又包含低优先级就绪状态的任务，高优先级就绪状态的任务会优先被执行；

服务质量则根据CPU，网络和磁盘的分配来创建一个操作的系统优先级。一个高质量的服务意味着可以提供更多的资源来更快的完成操作，涉及到CPU调度的优先级、IO优先级、任务运行所在的线程以及运行的顺序等等。正确的指定线程或任务优先级可以让系统更加智能的管理队列的资源分配，以便于提高执行效率和控制电量等。

#### 自定义NSOperation子类

当NSInvocationOperation或者NSBlockOperation无法满足我们日常需求，我们也可以定义串行和并发的2种类型的NSOperation子类，注意需要创建自动释放池，异步操作无法访问主线程的自动释放池。在自定义串行NSOperation子类时要重写`main`方法并且最好添加一个init方法用于初始化数据，在自定义并行NSOperation子类是需要重写`start、isFinished、isAsynchronous`和`isExecuting`方法。经常通过cancelled属性检查方法是否取消，并且对取消的做出响应，定期调用对象的isCancelled方法，如果返回“YES”，则立即返回，不再执行任务。

如果想进一步了解自定义NSOperation子类的具体实现，接下来将利用自定义NSOperation子类，同时借鉴了`AFNetworking、SDWebImage、YYKit`的部分思想来实现具有缓存支持的异步图像下载器。

该文章为记录本人的学习路程，也希望能够帮助大家，知识共享，共同成长，共同进步！！！

