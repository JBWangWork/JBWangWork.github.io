---
title: 多线程原理--了解GCD
date: 2019-10-17 20:57:55
author: Vincent
categories: 
- 底层原理
- 源码分析
tags: 
- GCD
- 底层原理
---


![多线程原理--了解GCD](https://upload-images.jianshu.io/upload_images/5741330-7fde16d065552b2e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


#### GCD 简介
在iOS 4版本之后引用GCD。GCD全称是 Grand Central Dispatch，纯 C 语言，提供了非常多强大的函数。GCD是苹果公司为多核的并行运算提出的解决方案，会自动利用更多的CPU内核（比如双核、四核），GCD 会自动管理线程的生命周期（创建线程、调度任务、销毁线程），iOS开发人员只需要告诉 GCD 想要执行什么任务，不需要编写任何线程管理代码，在多线程中GCD使用特别简单，这也是GCD 的优势。

GCD的任务使用没有参数也没有返回值的`block`封装，执行任务的函数：异步 `dispatch_async`（具备开启新线程的能力来执行`block`的任务，不用等待当前语句执行完毕，就可以执行下一条语句）；同步 `dispatch_sync`（不会开启线程，必须等待当前语句执行完毕，才会执行下一条语句）。先简单来个例子：把任务添加到队列，并指定函数

```
- (void)syncTest{
    dispatch_block_t block = ^{
        NSLog(@"hello GCD");
    };
    //串行队列
    dispatch_queue_t queue = dispatch_queue_create("com.gcdTest", NULL);
    // 同步执行任务
//    dispatch_sync(queue, block);
//    dispatch_sync(queue, ^{
//        // 同步执行任务代码
//        NSLog(@"hello GCD");
//    });

    // 异步执行任务
//    dispatch_async(queue, block);
    dispatch_async(queue, ^{
    // 异步执行任务代码
        NSLog(@"hello GCD");
    });
}
```

 #### 任务队列
使用`dispatch_queue_create`来创建队列，需要传入两个参数，第一个参数表示队列的唯一标识符，推荐使用`AppId`这种逆序域名，可用于DEBUG；第二个参数用来识别是串行队列还是并发队列。用`DISPATCH_QUEUE_SERIAL` 表示串行队列，`DISPATCH_QUEUE_CONCURRENT` 表示并发队列，还有两种特殊队列：全局并发队列、主队列。

##### 队列

- 串行队列（Serial Dispatch Queue）：
  每次只有一个任务被执行。让任务一个接着一个地执行（只开启一个线程，一个任务执行完毕后，再执行下一个任务）。
```
dispatch_queue_t queue = 
dispatch_queue_create("com.gcdTest", DISPATCH_QUEUE_SERIAL);
```
- 并发队列（Concurrent Dispatch Queue）：
  可以让多个任务并发（同时）执行（可以开启多个线程，并且同时执行任务。）。只有在异步`dispatch_async`函数下才有效。
```
dispatch_queue_t queue = 
dispatch_queue_create("com.gcdTest", DISPATCH_QUEUE_CONCURRENT);
```
- 全局并发队列（Global Dispatch Queue）：
GCD默认提供的全局并发队列，可以使用dispatch_get_global_queue来获取，需要传入两个参数。第一个参数表示队列优先级，一般用DISPATCH_QUEUE_PRIORITY_DEFAULT。第二个参数暂时没用，用0即可：
```
/**
* arg1:队列优先级
* arg2:保留字段备用，一般为0
*/
dispatch_queue_t queue = 
dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
```
- 主队列（Main Dispatch Queue）：
GCD提供的一种特殊的串行队列，所有放在主队列中的任务，都会放到主线程中执行。我们可以使用dispatch_get_main_queue()获得主队列：
```
dispatch_queue_t queue = dispatch_get_main_queue();
```

其中全局并发队列可以作为普通并发队列使用，这样我们在创建任务的时候，可以选择6种不同的任务执行方式，同步执行 + 并发队列；异步执行 + 并发队列；同步执行 + 串行队列；异步执行 + 串行队列；同步执行 + 主队列；异步执行 + 主队列；6种组合方式的区别：
区别|并发队列|串行队列|主队列
|:---:|:---:|:---:|:---:|
同步(sync-) | 没有开启新的线程 | 串行执行任务 | 没有开启新线程，串行执行任务
| 异步(async) | 开启新线程，并发执行任务 | 有开启新线程()，串行执行任务 | 没有开启新线程，串行执行任务

#### GCD的基本使用
**并发队列异步函数内部再次执行异步函数：**
```
dispatch_queue_t queue = dispatch_queue_create("com.gcdTest", DISPATCH_QUEUE_CONCURRENT);
    NSLog(@"1");
    // 耗时
    dispatch_async(queue, ^{
        NSLog(@"2");
        dispatch_async(queue, ^{
            NSLog(@"3");
        });
        NSLog(@"4");
    });
    NSLog(@"5");
```
执行结果为1-5-2-4-3。首先打印1，之后进入耗时操作，打印5，进入第一个异步函数打印2，再次进入耗时操作，打印4，再打印异步函数内的3。


**串行队列异步函数内部再次执行同步函数：**
```
dispatch_queue_t queue = dispatch_queue_create("com.gcdTest", DISPATCH_QUEUE_SERIAL);
    NSLog(@"1");
    // 耗时
    dispatch_async(queue, ^{
        NSLog(@"2");
        dispatch_sync(queue, ^{
            NSLog(@"3");
        });
        NSLog(@"4");
    });
    NSLog(@"5");
```
以下串行队列：1-5-2 之后崩溃。首先打印1，之后进入耗时操作，打印5，进入第一个异步函数打印2，此时进入串行队列；正常逻辑是需要先执行block，之后打印4，block再执行3，因为串行队列是FIFO的，当任务执行到block的时候，block等待3执行完毕后，继续打印4；而3则是需要等待4执行完毕后才执行；而4需要等待block执行完毕后才执行，这样就形成了死锁，所以打印2后奔溃。

**以下代码会产生什么问题？**
```
    int a = 0;
    while (a<10) {
        dispatch_async(dispatch_get_global_queue(0, 0), ^{
            a++;
    });
    NSLog(@"主线程%d",a);
```
**产生的问题：**首先`a`会报错，我们需要添加`__block`修饰，以`a`的指针和值的struct形式从栈区copy到堆区的新的`A`；其次在while循环内是异步并发，需要开辟新线程，是个耗时操作，可能在上一个线程没有执行完毕的时候又创建一个新的线程继续执行`a++`操作，结果就是可能多条线程内的`a`的值都是相同的，这样导致最后`a`的结果可能大于等于10，即使已经打印了`a`的值后可能还会有很多线程在执行，导致最后`a`的真正的值会很大。

**解决问题：**我们知道最后会创建出大于等于10条线程，我们可以利用锁的方式来保证最后只创建10条线程，以NSLock为例：
```
    int a = 0;
    NSLock *lock = [NSLock new];
    while (a<10) {
        dispatch_async(dispatch_get_global_queue(0, 0), ^{
            a++;
            [lock unlock];
    });
    [lock lock];
    NSLog(@"主线程%d",a);
```

------
 **开发中我们经常会碰到一些有依赖关系的任务，比如在A请求后再进行B、C请求这样的操作。解决办法很多，比如直接先同步执行A，再异步执行B、C，或者可以把A、B和C放在一个task任务中再进行相同的执行顺序，这样虽然解决了问题但是会堵塞线程，影响其他操作，我们也可以用栅栏函数来解决。**

#### 栅栏函数

> TIP :
栅栏函数可以保证顺序执行，也可以保证线程安全，但一定要是自定义并发队列。正因为栅栏函数只能控制同一自定义并发队列，所以不利于封装。

```
void
dispatch_barrier_async(dispatch_queue_t queue, dispatch_block_t block);

void
dispatch_barrier_sync(dispatch_queue_t queue,
		DISPATCH_NOESCAPE dispatch_block_t block);
```

`dispatch_barrier_async`和`dispatch_barrier_sync`的区别就是是否阻塞当前线程，很明显`dispatch_barrier_async`不会阻塞当前线程：
```
dispatch_queue_t concurrentQueue = dispatch_queue_create("concurrentQueue", DISPATCH_QUEUE_CONCURRENT);
    /* 1.异步函数 */
    dispatch_async(concurrentQueue, ^{
        for (NSUInteger i = 0; i < 5; i++) {
            NSLog(@"加载1-%zd-%@",i,[NSThread currentThread]);
        }
    });
    
    dispatch_async(concurrentQueue, ^{
        for (NSUInteger i = 0; i < 5; i++) {
            NSLog(@"加载2-%zd-%@",i,[NSThread currentThread]);
        }
    });
    
    /* 2. 栅栏函数 */
    dispatch_barrier_async(concurrentQueue, ^{
        NSLog(@"---------------------%@------------------------",[NSThread currentThread]);
    });
    NSLog(@"**********加载完毕!!!**********");
    /* 3. 异步函数 */
    dispatch_async(concurrentQueue, ^{
        for (NSUInteger i = 0; i < 5; i++) {
            NSLog(@"处理结果3-%zd-%@",i,[NSThread currentThread]);
        }
    });
    NSLog(@"**********继续!!!**********");
    
    dispatch_async(concurrentQueue, ^{
        for (NSUInteger i = 0; i < 5; i++) {
            NSLog(@"处理结果4-%zd-%@",i,[NSThread currentThread]);
        }
    });
```

用栅栏函数`dispatch_barrier_async `打印效果：
```
**********加载完毕!!!**********
加载2-0-<NSThread: 0x600000246100>{number = 4, name = (null)}
加载1-0-<NSThread: 0x6000002797c0>{number = 5, name = (null)}
加载2-1-<NSThread: 0x600000246100>{number = 4, name = (null)}
加载1-1-<NSThread: 0x6000002797c0>{number = 5, name = (null)}
**********继续!!!**********
加载2-2-<NSThread: 0x600000246100>{number = 4, name = (null)}
加载1-2-<NSThread: 0x6000002797c0>{number = 5, name = (null)}
加载2-3-<NSThread: 0x600000246100>{number = 4, name = (null)}
加载1-3-<NSThread: 0x6000002797c0>{number = 5, name = (null)}
加载2-4-<NSThread: 0x600000246100>{number = 4, name = (null)}
加载1-4-<NSThread: 0x6000002797c0>{number = 5, name = (null)}
---------------------<NSThread: 0x6000002797c0>{number = 5, name = (null)}------------------------
处理结果3-0-<NSThread: 0x6000002797c0>{number = 5, name = (null)}
处理结果4-0-<NSThread: 0x600000246100>{number = 4, name = (null)}
处理结果3-1-<NSThread: 0x6000002797c0>{number = 5, name = (null)}
处理结果4-1-<NSThread: 0x600000246100>{number = 4, name = (null)}
处理结果3-2-<NSThread: 0x6000002797c0>{number = 5, name = (null)}
处理结果4-2-<NSThread: 0x600000246100>{number = 4, name = (null)}
处理结果3-3-<NSThread: 0x6000002797c0>{number = 5, name = (null)}
处理结果4-3-<NSThread: 0x600000246100>{number = 4, name = (null)}
处理结果3-4-<NSThread: 0x6000002797c0>{number = 5, name = (null)}
处理结果4-4-<NSThread: 0x600000246100>{number = 4, name = (null)}
```

同样代码，如果用栅栏函数`dispatch_barrier_sync`，打印结果为：

```
加载2-0-<NSThread: 0x600000246100>{number = 4, name = (null)}
加载1-0-<NSThread: 0x6000002797c0>{number = 5, name = (null)}
加载2-1-<NSThread: 0x600000246100>{number = 4, name = (null)}
加载1-1-<NSThread: 0x6000002797c0>{number = 5, name = (null)}
加载2-2-<NSThread: 0x600000246100>{number = 4, name = (null)}
加载1-2-<NSThread: 0x6000002797c0>{number = 5, name = (null)}
加载2-3-<NSThread: 0x600000246100>{number = 4, name = (null)}
加载1-3-<NSThread: 0x6000002797c0>{number = 5, name = (null)}
加载2-4-<NSThread: 0x600000246100>{number = 4, name = (null)}
加载1-4-<NSThread: 0x6000002797c0>{number = 5, name = (null)}
---------------------<NSThread: 0x6000002797c0>{number = 5, name = (null)}------------------------
**********加载完毕!!!**********
**********继续!!!**********
处理结果3-0-<NSThread: 0x6000002797c0>{number = 5, name = (null)}
处理结果4-0-<NSThread: 0x600000246100>{number = 4, name = (null)}
处理结果3-1-<NSThread: 0x6000002797c0>{number = 5, name = (null)}
处理结果4-1-<NSThread: 0x600000246100>{number = 4, name = (null)}
处理结果3-2-<NSThread: 0x6000002797c0>{number = 5, name = (null)}
处理结果4-2-<NSThread: 0x600000246100>{number = 4, name = (null)}
处理结果3-3-<NSThread: 0x6000002797c0>{number = 5, name = (null)}
处理结果4-3-<NSThread: 0x600000246100>{number = 4, name = (null)}
处理结果3-4-<NSThread: 0x6000002797c0>{number = 5, name = (null)}
处理结果4-4-<NSThread: 0x600000246100>{number = 4, name = (null)}
```

#### 调度组

虽然栅栏函数很好用，但是还不是很完美，可以利用调度组来解决栅栏函数的不足。
TIP：`dispatch_group_create `：创建调度组，`dispatch_group_async `：异步提交任务到调度组中，`dispatch_group_notify`：监听调度组任务是否执行完毕。
```
    //创建调度组
    dispatch_group_t group = dispatch_group_create();
    dispatch_queue_t queue = dispatch_get_global_queue(0, 0);

    // SIGNAL
    dispatch_group_async(group, queue, ^{
        NSLog(@"第一个走完了");
    });
    
    dispatch_group_async(group, queue, ^{
        NSLog(@"第二个走完了");
    });
    
    dispatch_group_notify(group, dispatch_get_main_queue(), ^{
        NSLog(@"所有任务完成,可以更新UI");
    });
```

当我们不使用`dispatch_group_async`来提交任务的时候，我们也可以使用`dispatch_group_enter和`dispatch_group_leave`来实现：

```
    dispatch_queue_t queue = dispatch_get_global_queue(0, 0);
    dispatch_group_t group = dispatch_group_create();
    
    dispatch_group_enter(group);
    dispatch_async(queue, ^{
        NSLog(@"第一个走完了");
        dispatch_group_leave(group);
    });
    
    dispatch_group_enter(group);
    dispatch_async(queue, ^{
        NSLog(@"第二个走完了");
        dispatch_group_leave(group);
    });
    
    dispatch_group_notify(group, dispatch_get_main_queue(), ^{
        NSLog(@"所有任务完成,可以更新UI");
    });
```
> TIP：`dispatch_group_enter`和`dispatch_group_leave`必须配合使用。

两种写法结果都是先执行前面两个任务，最后执行notify内部的block任务。
#### 信号量

GCD中的信号量是指`Dispatch Semaphore`，是持有计数的信号。除了调度组外，还可以使用信号量来解决问题。

GCD信号量机制主要提供了以下三个函数：
```
dispatch_semaphore_create(long value); // 创建信号量
dispatch_semaphore_signal(dispatch_semaphore_t deem); // 发送信号量
dispatch_semaphore_wait(dispatch_semaphore_t dsema, dispatch_time_t timeout); // 等待信号量
```
`dispatch_semaphore_create`：创建一个`dispatch_semaphore_t`类型的信号量，并设定信号量的大小。
`dispatch_semaphore_wait`：等待信号量，对信号量的值进行减1操作，如果信号量值为0，那么该函数就会一直等待，相当于阻塞当前线程，直到该函数等待的信号量的值大于等于1。
`dispatch_semaphore_signal`：对信号量的值进行加1操作。

一般等待信号量和发送信号量的函数是成对出现的。在并发执行任务时候，在当前任务执行之前，用`dispatch_semaphore_wait`函数对信号量的值减1操作（信号量为0时进行等待），执行当前任务后再通过`dispatch_semaphore_signal`函数对信号量的值加1操作来发送信号量，通知执行下一个任务。这样GCD就可以使用信号量来控制并发数，也可以保持线程同步，保证线程安全，当锁来使用。

举个例子，以下代码由于`dispatch_semaphore_create`创建2个信号量，所以先执行任务1和任务2，等任务1和任务2完成后再执行任务3。
```
    dispatch_semaphore_t semaphore = dispatch_semaphore_create(2);
    
    //任务1
    dispatch_async(queue, ^{
        dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
        NSLog(@"执行任务1");
        sleep(1);
        NSLog(@"任务1完成");
        dispatch_semaphore_signal(semaphore);
    });
    
    //任务2
    dispatch_async(queue, ^{
        dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
        NSLog(@"执行任务2");
        sleep(1);
        NSLog(@"任务2完成");
        dispatch_semaphore_signal(semaphore);
    });
    
    //任务3
    dispatch_async(queue, ^{
        dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
        NSLog(@"执行任务3");
        sleep(1);
        NSLog(@"任务3完成");
        dispatch_semaphore_signal(semaphore);
    });
```
#### 延迟执行

GCD中的延迟执行`dispatch_after`是延时将任务追加到对应队列中，执行block块中的任务，具体延迟多少时间并不一定，对于一些模糊的延迟任务来说还是很有效的，比如在主线程中基本不会用`sleep`来延迟方法的调用，所以用`dispatch_after`是最合适的。
```
//NSEC_PER_SEC : 1000000000ull 纳秒每秒 0.0000001
    dispatch_time_t time = dispatch_time(DISPATCH_TIME_NOW, (int64_t)(1 * NSEC_PER_SEC));
    dispatch_queue_t queue = dispatch_queue_create("com.lg.cn", DISPATCH_QUEUE_CONCURRENT);
    dispatch_after(time, queue, ^{
        NSLog(@"延迟执行");
    });
    NSLog(@"继续执行");
```

#### 调度资源

GCD中的调度资源`dispatch_source`是一种用于处理事件的数据类型，这些被处理的事件为操作系统中的底层级别。常用于处理跟系统有关的事件，协调处理指定的低级别的系统事件，在系统处理请求时应用程序可以继续处理自己的事情。 

**优势：**
- 其CPU负荷非常小，尽量不占用资源。
- 联结的优势
当你配置一个dispatch source时，你指定要监测的事件、dispatch queue、以及处理事件的代码(block或函数)。当事件发生时（在任一线程上调用它的函数`dispatch_source_merge_data`），dispatch source会提交你的block或函数到指定的queue去执行和手工提交到queue的任务不同，dispatch source为应用提供连续的事件源。除非你显式地取消，dispatch source会一直保留与dispatch queue的关联。


**使用：**

- 创建源`dispatch_source_create`：
```
/**
*  arg1：用于标识Dispatch Source要监听的事件类型，共有11个类型。
*  arg2：取决于要监听的事件类型，如果是监听Mach端口相关的事件，那么该参数就是mach_port_t类型的Mach端口号，如果是监听事件变量数据类型的事件那么该参数就不需要，设置为0就可以了。
* arg3：取决于要监听的事件类型，如果是监听文件属性更改的事件，那么该参数就标识文件的哪个属性，比如DISPATCH_VNODE_RENAME。Apple的API介绍说，使用DISPATCH_TIMER_STRICT，会引起电量消耗加剧，毕竟要求精确时间，所以一般传0即可，视业务情况而定。
* arg4：设置回调函数所在的队列，可以传Null，默认为全局队列。
*/
dispatch_source_t
dispatch_source_create(dispatch_source_type_t type,
	                   uintptr_t handle,
	                   unsigned long mask,
	                   dispatch_queue_t _Nullable queue);
```

- 设置源事件回调`dispatch_source_set_event_handler`(block形式)或者`dispatch_source_set_event_handler_f`(函数形式)
以`dispatch_source_set_event_handler`为例:
```
void
dispatch_source_set_event_handler(dispatch_source_t source,
	dispatch_block_t _Nullable handler);
```

- 源事件设置数据`dispatch_source_merge_data`：
```
void
dispatch_source_merge_data(dispatch_source_t source, unsigned long value);
```

- 获取源事件数据`dispatch_source_get_data`：
```
unsigned long
dispatch_source_get_data(dispatch_source_t source);
```

- 继续`dispatch_resume`：
```
void
dispatch_resume(dispatch_object_t object);
```

- 挂起`dispatch_suspend`：
```
void
dispatch_suspend(dispatch_object_t object);
```


模拟下载进度部分业务代码：
```
    self.source = dispatch_source_create(DISPATCH_SOURCE_TYPE_DATA_ADD, 0, 0, dispatch_get_main_queue());
    // 封装我们需要回调的触发函数 -- 响应
    dispatch_source_set_event_handler(self.source, ^{
        
        NSUInteger value = dispatch_source_get_data(self.source); // 取回来值 1 响应式
        self.totalComplete += value;
        NSLog(@"进度：%.2f", self.totalComplete/100.0);
        self.progressView.progress = self.totalComplete/100.0;
    });
    dispatch_resume(self.source);
    self.isRunning     = YES;
```

触发函数：
```
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event{

    NSLog(@"点击开始加载");
    
    for (NSUInteger index = 0; index < 100; index++) {
        dispatch_async(self.queue, ^{
            if (!self.isRunning) {
                NSLog(@"暂停下载");
                return ;
            }
            sleep(2);

            dispatch_source_merge_data(self.source, 1); // source 值响应
        });
    }
}
```

具体代码：[Github直通车-->https://github.com/JBWangWork/DispatchSourceTest](https://github.com/JBWangWork/DispatchSourceTest)

该文章为记录本人的学习路程，也希望能够帮助大家，知识共享，共同成长，共同进步！！！

