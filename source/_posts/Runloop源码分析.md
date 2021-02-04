---
title: Runloop底层原理--源码分析
date: 2019-06-23 19:58:22
author: Vincent
categories: 
- 底层原理
- 源码分析
tags: 
- Runtime
- 源码分析
- 底层原理
---

![Runloop底层原理](https://upload-images.jianshu.io/upload_images/5741330-7686a91193dd3b8a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


#### 什么是Runloop？
Runloop不仅仅是一个运行循环（do-while循环），也是提供了一个入口函数的对象，消息机制处理模式。运行循环从两种不同类型的源接收事件。
输入源提供异步事件，通常是来自另一个线程或来自不同应用程序的消息。定时器源提供同步事件，发生在预定时间或重复间隔。
两种类型的源都使用特定于应用程序的处理程序例程来处理事件。除了处理输入源之外，Runloop还会生成有关Runloop行为的通知。
已注册的运行循环观察器可以接收这些通知并使用它们在线程上执行其他处理。

![run loop的结构及来源](https://upload-images.jianshu.io/upload_images/5741330-00d3a78c266ae9a5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

执行代码查看下主运行循环的部分信息：
```
    CFRunLoopRef mainRunloop = CFRunLoopGetMain();
    NSLog(@"%@", mainRunloop);
```
打印结果，里面有port源、modes、items等，items有很多实体（CFRunLoopSource，CFRunLoopObserver等），打印省略N行
```
<CFRunLoop 0x6000014c8300 [0x1034faae8]>{wakeup port = 0x2207, stopped = false, ignoreWakeUps = false, 
current mode = kCFRunLoopDefaultMode,
common modes = <CFBasicHash 0x60000268cb40 [0x1034faae8]>{type = mutable set, count = 2,
entries =>
	0 : <CFString 0x1068ca070 [0x1034faae8]>{contents = "UITrackingRunLoopMode"}
	2 : <CFString 0x10350ced8 [0x1034faae8]>{contents = "kCFRunLoopDefaultMode"}
}
,
common mode items = <CFBasicHash 0x600002680ab0 [0x1034faae8]>{type = mutable set, count = 13,
entries =>
	0 : <CFRunLoopSource 0x600001dc4a80 [0x1034faae8]>{signalled = No, valid = Yes, order = -1, context = <CFRunLoopSource context>{version = 0, info = 0x0, callout = PurpleEventSignalCallback (0x10b77e2bb)}}
	3 : <CFRunLoopSource 0x600001dc8e40 [0x1034faae8]>{signalled = No, valid = Yes, order = -2, context = <CFRunLoopSource context>{version = 0, info = 0x600002680840, callout = __handleHIDEventFetcherDrain (0x1060e0842)}}
... // 省略N行
```
##### Runloop对象
```
Foundation:NSRunLoop
[NSRunLoop currentRunLoop]; // 获得当前线程的RunLoop对象
[NSRunLoop mainRunLoop]; // 获得主线程的RunLoop对象

Core Foundation:CFRunLoopRef
CFRunLoopGetCurrent(); // 获得当前线程的RunLoop对象
CFRunLoopGetMain(); // 获得主线程的RunLoop对象
```
#### Runloop的作用：
1. Runloop可以保持程序的持续运行；
2. 处理APP中的各种事件（比如触摸，定时器，performSelector）；
3. 节省cup资源、提供程序的性能（需要执行事务就执行，不需要就休眠）；
#### Runloop的应用：
1. block应用：__CFRUNLOOP_IS_CALLING_OUT_TO_A_BLOCK__
2. 调用timer：__CFRUNLOOP_IS_CALLING_OUT_TO_A_TIMER_CALLBACK_FUNCTION__
3. 响应source0：__CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__
4. 响应source1： __CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE1_PERFORM_FUNCTION__
5. GCD主队列：__CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__
6. observer源：__CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__

#### Runloop与线程的关系

- Runloop与线程是一一对应的；一个Runloop对应一个核心的线程，Runloop是可以嵌套的，但是核心的只能有一个，他们的关系保存在一个全局的字典里。
- Runloop是用来管理线程的；当线程的Runloop被开启后，线程会在执行完任务后进入休眠状态，有了任务就会被唤醒去执行任务。
- Runloop在第一次获取时被创建，在线程结束时被销毁。
- 对于主线程来说，Runloop在程序一启动就默认创建好了。
- 对于子线程来说，Runloop是懒加载的。只有当我们使用的时候才会创建，所以在子线程用定时器要注意，确保子线程的Runloop被创建，不让定时器不会回调。

##### Runloop与线程源码分析
可以先去官方下载源码进行分析；通过主线程获取main
```
#if DEPLOYMENT_TARGET_WINDOWS || DEPLOYMENT_TARGET_IPHONESIMULATOR
CF_EXPORT pthread_t _CF_pthread_main_thread_np(void);
#define pthread_main_thread_np() _CF_pthread_main_thread_np()
#endif

CFRunLoopRef CFRunLoopGetMain(void) {
    CHECK_FOR_FORK();
    static CFRunLoopRef __main = NULL; // no retain needed
    if (!__main) __main = _CFRunLoopGet0(pthread_main_thread_np()); // no CAS needed
    return __main;
}
```
_CFRunLoopGet0内部调用：通过一个全局可变字典CFMutableDictionaryRef，`__CFRunLoopCreate(pthread_main_thread_np())`创建mainLoop，对CFMutableDictionaryRef进行setValue，pthread_main_thread_np()线程的指针会指向当前的mainLoop，从这里就可以看出，runLoop是基于线程创建的并且runLoop和线程是以key-value的形式一一对应的。当然CFDictionaryGetValue通过当前的__CFRunLoops，关联pthreadPointer(t)的指针，获取到当前的loop，都可以证明runloop和线程是一一对应的关系。
```
__CFSpinLock(&loopsLock);
    if (!__CFRunLoops) {
        __CFSpinUnlock(&loopsLock);
        CFMutableDictionaryRef dict = CFDictionaryCreateMutable(kCFAllocatorSystemDefault, 0, NULL, &kCFTypeDictionaryValueCallBacks);
        
        CFRunLoopRef mainLoop = __CFRunLoopCreate(pthread_main_thread_np());
        CFDictionarySetValue(dict, pthreadPointer(pthread_main_thread_np()), mainLoop);
        if (!OSAtomicCompareAndSwapPtrBarrier(NULL, dict, (void * volatile *)&__CFRunLoops)) {
            CFRelease(dict);
        }
        CFRelease(mainLoop);
        __CFSpinLock(&loopsLock);
    }
    
    CFRunLoopRef loop = (CFRunLoopRef)CFDictionaryGetValue(__CFRunLoops, pthreadPointer(t));
    __CFSpinUnlock(&loopsLock);
```

##### Runloop与线程代码实现
程序启动创建了一个子线程，在子线程内添加了一个定时器timer，并开启子线程的runLoop，开始打印`hello word`，当点击屏幕时退出子线程停止打印。
```
- (void)viewDidLoad {
    [super viewDidLoad];

    // 主运行循环
//     CFRunLoopRef mainRunloop = CFRunLoopGetMain();
//    NSLog(@"%@", mainRunloop);
    
    self.isStopping = NO;
    NSThread *customThread = [[NSThread alloc] initWithBlock:^{
        NSLog(@"%@---%@",[NSThread currentThread],[[NSThread currentThread] name]);
        [NSTimer scheduledTimerWithTimeInterval:1 repeats:YES block:^(NSTimer * _Nonnull timer) {
            NSLog(@"hello word");     
            if (self.isStopping) {
                [NSThread exit];
            }
        }];
        [[NSRunLoop currentRunLoop] run];
    }];
    customThread.name = @"customThread";
    [customThread start];
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event{
    self.isStopping = YES;
}
```

打印结果：
```
<NSThread: 0x600001fb21c0>{number = 3, name = customThread}---customThread
hello word
hello word
hello word
hello word
hello word
hello word
hello word
```
###### TIP：
项目启动，通过isStopping变量来控制当前线程，线程控制runloop，runloop控制timer。注意子线程runloop默认不开启。timer依赖于runloop。

#### Runloop源码分析
```
    CFRunLoopRef runLoop     = CFRunLoopGetCurrent();
    CFRunLoopMode loopMode  = CFRunLoopCopyCurrentMode(runLoop);
    NSLog(@"mode == %@",loopMode);
    CFArrayRef modeArray= CFRunLoopCopyAllModes(runLoop);
    NSLog(@"modeArray == %@",modeArray);
```
##### CFRunLoopRef源码分析
Runloop是利用线程创建的CFRunLoopRef类型，通过源码定位，看到__CFRunLoop是一个结构体，里面包含了_base、_lock、_wakeUpPort(激活port)、_commonModes、_commonModeItems、_modes等等，默认mode为kCFRunLoopDefaultMode类型


```
typedef CFStringRef CFRunLoopMode CF_EXTENSIBLE_STRING_ENUM;

typedef struct CF_BRIDGED_MUTABLE_TYPE(id) __CFRunLoop * CFRunLoopRef;

typedef struct CF_BRIDGED_MUTABLE_TYPE(id) __CFRunLoopSource * CFRunLoopSourceRef;

typedef struct CF_BRIDGED_MUTABLE_TYPE(id) __CFRunLoopObserver * CFRunLoopObserverRef;

typedef struct CF_BRIDGED_MUTABLE_TYPE(NSTimer) __CFRunLoopTimer * CFRunLoopTimerRef;


struct __CFRunLoop {
    CFRuntimeBase _base;
    pthread_mutex_t _lock;            /* locked for accessing mode list */
    __CFPort _wakeUpPort;            // used for CFRunLoopWakeUp
    Boolean _unused;
    volatile _per_run_data *_perRunData;              // reset for runs of the run loop
    pthread_t _pthread;
    uint32_t _winthread;
    CFMutableSetRef _commonModes;
    CFMutableSetRef _commonModeItems;
    CFRunLoopModeRef _currentMode;
    CFMutableSetRef _modes;
    struct _block_item *_blocks_head;
    struct _block_item *_blocks_tail;
    CFTypeRef _counterpart;
};
```
##### CFRunLoopMode源码分析
一个runLoop可以包含很多种Mode，CFRunLoopMode也是一个结构体，其中包含_sources0、_sources1、_observers、_timers等等

> RunLoop的五种运行模式：
>1. kCFRunLoopDefaultMode：App的默认Mode，通常主线程是在这个Mode下运行
> 2. UITrackingRunLoopMode：界面跟踪 Mode，用于 ScrollView 追踪触摸滑动，保证界面滑动时不受其他 Mode 影响
> 3. UIInitializationRunLoopMode: 在刚启动 App 时第进入的第一个 Mode，启动完成后就不再使用
> 4. GSEventReceiveRunLoopMode: 接受系统事件的内部 Mode，通常用不到
> 5. kCFRunLoopCommonModes: 这是一个占位用的Mode，作为标记kCFRunLoopDefaultMode和UITrackingRunLoopMode用，并不是一种真正的Mode

```
typedef struct __CFRunLoopMode *CFRunLoopModeRef;

struct __CFRunLoopMode {
    CFRuntimeBase _base;
    pthread_mutex_t _lock;    /* must have the run loop locked before locking this */
    CFStringRef _name;
    Boolean _stopped;
    char _padding[3];
    CFMutableSetRef _sources0;
    CFMutableSetRef _sources1;
    CFMutableArrayRef _observers;
    CFMutableArrayRef _timers;
    CFMutableDictionaryRef _portToV1SourceMap;
    __CFPortSet _portSet;
    CFIndex _observerMask;
#if USE_DISPATCH_SOURCE_FOR_TIMERS
    dispatch_source_t _timerSource;
    dispatch_queue_t _queue;
    Boolean _timerFired; // set to true by the source when a timer has fired
    Boolean _dispatchTimerArmed;
#endif
#if USE_MK_TIMER_TOO
    mach_port_t _timerPort;
    Boolean _mkTimerArmed;
#endif
#if DEPLOYMENT_TARGET_WINDOWS
    DWORD _msgQMask;
    void (*_msgPump)(void);
#endif
    uint64_t _timerSoftDeadline; /* TSR */
    uint64_t _timerHardDeadline; /* TSR */
};
```
其中的items通过`CFRunLoopAddSource`、`CFRunLoopAddObserver`、`CFRunLoopAddTimer`来添加CFRunLoopSourceRef、CFRunLoopObserverRef、CFRunLoopTimerRef。

##### RunLoop 结构
经过源码，我们发现，CFRunLoop和线程是一一对应的，一个CFRunLoop对应多个CFRunLoopMode，一个CFRunLoopMode对应多个CFRunLoopSource、CFRunLoopObserver、CFRunLoopTimer。



#### RunLoop和Obsever的关系
Obsever就是观察者，能够监听RunLoop的状态改变，创建这个观察者，再通过`CFRunLoopAddObserver `把观察者添加到runloop中，`runLoopObserverCallBack `来监听状态的变化。
CFRunLoopObserverRef：
```
- (void)cfObseverDemo{
    
    CFRunLoopObserverContext context = {
        0,
        ((__bridge void *)self),
        NULL,
        NULL,
        NULL
    };
    CFRunLoopRef rlp = CFRunLoopGetCurrent();
    /**
     参数一:用于分配对象的内存
     参数二:你关注的事件
          kCFRunLoopEntry=(1<<0),
          kCFRunLoopBeforeTimers=(1<<1),
          kCFRunLoopBeforeSources=(1<<2),
          kCFRunLoopBeforeWaiting=(1<<5),
          kCFRunLoopAfterWaiting=(1<<6),
          kCFRunLoopExit=(1<<7),
          kCFRunLoopAllActivities=0x0FFFFFFFU
     参数三:CFRunLoopObserver是否循环调用
     参数四:CFRunLoopObserver的优先级 当在Runloop同一运行阶段中有多个CFRunLoopObserver 正常情况下使用0
     参数五:回调,比如触发事件,我就会来到这里
     参数六:上下文记录信息
     */
    CFRunLoopObserverRef observerRef = CFRunLoopObserverCreate(kCFAllocatorDefault, kCFRunLoopAllActivities, YES, 0, runLoopObserverCallBack, &context);
    CFRunLoopAddObserver(rlp, observerRef, kCFRunLoopDefaultMode);
}

void runLoopObserverCallBack(CFRunLoopObserverRef observer, CFRunLoopActivity activity, void *info){
    NSLog(@"%lu-%@",activity,info);
}
```

![run loop](https://upload-images.jianshu.io/upload_images/5741330-aa756fcfc4344e8d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


进入源码，runloop就是一个do-while循环，再次进入`CFRunLoopRunSpecific`方法，如果监听到有进入状态或者退出状态改变则执行`__CFRunLoopDoObservers`，其余的进入`__CFRunLoopRun `方法。
```
void CFRunLoopRun(void) {    /* DOES CALLOUT */
    int32_t result;
    do {
        result = CFRunLoopRunSpecific(CFRunLoopGetCurrent(), kCFRunLoopDefaultMode, 1.0e10, false);
        CHECK_FOR_FORK();
    } while (kCFRunLoopRunStopped != result && kCFRunLoopRunFinished != result);
}

// -----------------------------------

SInt32 CFRunLoopRunSpecific(CFRunLoopRef rl, CFStringRef modeName, CFTimeInterval seconds, Boolean returnAfterSourceHandled) {     /* DOES CALLOUT */
    CHECK_FOR_FORK();
    if (__CFRunLoopIsDeallocating(rl)) return kCFRunLoopRunFinished;
    __CFRunLoopLock(rl);
    // 根据modeName找到本次运行的mode
    CFRunLoopModeRef currentMode = __CFRunLoopFindMode(rl, modeName, false);
    // 如果没找到 || mode中没有注册任何事件，则就此停止，不进入循环
    if (NULL == currentMode || __CFRunLoopModeIsEmpty(rl, currentMode, rl->_currentMode)) {
        Boolean did = false;
        if (currentMode) __CFRunLoopModeUnlock(currentMode);
        __CFRunLoopUnlock(rl);
        return did ? kCFRunLoopRunHandledSource : kCFRunLoopRunFinished;
    }
    volatile _per_run_data *previousPerRun = __CFRunLoopPushPerRunData(rl);
    // 取上一次运行的mode
    CFRunLoopModeRef previousMode = rl->_currentMode;
    // 如果本次mode和上次的mode一致
    rl->_currentMode = currentMode;
    // 初始化一个result为kCFRunLoopRunFinished
    int32_t result = kCFRunLoopRunFinished;
    
    if (currentMode->_observerMask & kCFRunLoopEntry ) __CFRunLoopDoObservers(rl, currentMode, kCFRunLoopEntry);
    // 通知 Observers: RunLoop 即将进入 loop
    result = __CFRunLoopRun(rl, currentMode, seconds, returnAfterSourceHandled, previousMode);
    // 通知 Observers: RunLoop 即将退出。
    if (currentMode->_observerMask & kCFRunLoopExit ) __CFRunLoopDoObservers(rl, currentMode, kCFRunLoopExit);
   
    __CFRunLoopModeUnlock(currentMode);
    __CFRunLoopPopPerRunData(rl, previousPerRun);
    rl->_currentMode = previousMode;
    __CFRunLoopUnlock(rl);
    return result;
}
```

进入`__CFRunLoopRun `方法，其内部有Observers监听timer、source0、source1。
```
static int32_t __CFRunLoopRun(CFRunLoopRef rl, CFRunLoopModeRef rlm, CFTimeInterval seconds, Boolean stopAfterHandle, CFRunLoopModeRef previousMode) {
    
    //获取系统启动后的CPU运行时间，用于控制超时时间
    uint64_t startTSR = mach_absolute_time();
    
    // 判断当前runloop的状态是否关闭
    if (__CFRunLoopIsStopped(rl)) {
        __CFRunLoopUnsetStopped(rl);
        return kCFRunLoopRunStopped;
    } else if (rlm->_stopped) {
        return kCFRunLoopRunStopped;
        rlm->_stopped = false;
    }
    
    //mach端口，在内核中，消息在端口之间传递。 初始为0
    mach_port_name_t dispatchPort = MACH_PORT_NULL;
    //判断是否为主线程
    Boolean libdispatchQSafe = pthread_main_np() && ((HANDLE_DISPATCH_ON_BASE_INVOCATION_ONLY && NULL == previousMode) || (!HANDLE_DISPATCH_ON_BASE_INVOCATION_ONLY && 0 == _CFGetTSD(__CFTSDKeyIsInGCDMainQ)));
    //如果在主线程 && runloop是主线程的runloop && 该mode是commonMode，则给mach端口赋值为主线程收发消息的端口
    if (libdispatchQSafe && (CFRunLoopGetMain() == rl) && CFSetContainsValue(rl->_commonModes, rlm->_name)) dispatchPort = _dispatch_get_main_queue_port_4CF();
    
#if USE_DISPATCH_SOURCE_FOR_TIMERS
    mach_port_name_t modeQueuePort = MACH_PORT_NULL;
    if (rlm->_queue) {
        //mode赋值为dispatch端口_dispatch_runloop_root_queue_perform_4CF
        modeQueuePort = _dispatch_runloop_root_queue_get_port_4CF(rlm->_queue);
        if (!modeQueuePort) {
            CRASH("Unable to get port for run loop mode queue (%d)", -1);
        }
    }
#endif
```

`__CFRunLoopRun`部分源码，接下来进入do-while循环，先初始化一个存放内核消息的缓冲池，获取所有需要监听的port，设置RunLoop为可以被唤醒状态，判断是否有timer、source0、source1回调。如果有timer则通知 Observers: RunLoop 即将触发 Timer 回调。如果有source0则通知 Observers: RunLoop 即将触发 Source0 (非port) 回调，执行被加入的block。RunLoop 触发 Source0 (非port) 回调，再执行被加入的block。如果有 Source1 (基于port) 处于 ready 状态，直接处理这个 Source1 然后跳转去处理消息。例如一个Timer 到时间了，触发这个Timer的回调。处理完后再次进入`__CFArmNextTimerInMode`查看是否有其他的timer。如果没有事务需要处理则通知 Observers: RunLoop 的线程即将进入休眠(sleep)，此时会进入一个内循环，线程进入休眠状态mach_msg_trap（比如我们在断点调试的时候），直到收到新消息才跳出该循环，继续执行run loop。比如监听到了事务基于 port 的Source 的事件、Timer 到时间了、RunLoop 自身的超时时间到了或者被其他什么调用者手动唤醒则唤醒。
```
//标志位默认为true
    Boolean didDispatchPortLastTime = true;
    //记录最后runloop状态，用于return
    int32_t retVal = 0;
    do {
        //初始化一个存放内核消息的缓冲池
        uint8_t msg_buffer[3 * 1024];
#if DEPLOYMENT_TARGET_MACOSX || DEPLOYMENT_TARGET_EMBEDDED || DEPLOYMENT_TARGET_EMBEDDED_MINI
        mach_msg_header_t *msg = NULL;
        mach_port_t livePort = MACH_PORT_NULL;
#elif DEPLOYMENT_TARGET_WINDOWS
        HANDLE livePort = NULL;
        Boolean windowsMessageReceived = false;
#endif
        //取所有需要监听的port
        __CFPortSet waitSet = rlm->_portSet;
        
        //设置RunLoop为可以被唤醒状态
        __CFRunLoopUnsetIgnoreWakeUps(rl);
        
        /// 2. 通知 Observers: RunLoop 即将触发 Timer 回调。
        if (rlm->_observerMask & kCFRunLoopBeforeTimers) __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeTimers);
        if (rlm->_observerMask & kCFRunLoopBeforeSources)
            /// 3. 通知 Observers: RunLoop 即将触发 Source0 (非port) 回调。
            __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeSources);
        
        /// 执行被加入的block
        __CFRunLoopDoBlocks(rl, rlm);
        /// 4. RunLoop 触发 Source0 (非port) 回调。
        Boolean sourceHandledThisLoop = __CFRunLoopDoSources0(rl, rlm, stopAfterHandle);
        if (sourceHandledThisLoop) {
            /// 执行被加入的block
            __CFRunLoopDoBlocks(rl, rlm);
        }
        
        //如果没有Sources0事件处理 并且 没有超时，poll为false
        //如果有Sources0事件处理 或者 超时，poll都为true
        Boolean poll = sourceHandledThisLoop || (0ULL == timeout_context->termTSR);
        //第一次do..whil循环不会走该分支，因为didDispatchPortLastTime初始化是true
        if (MACH_PORT_NULL != dispatchPort && !didDispatchPortLastTime) {
#if DEPLOYMENT_TARGET_MACOSX || DEPLOYMENT_TARGET_EMBEDDED || DEPLOYMENT_TARGET_EMBEDDED_MINI
            //从缓冲区读取消息
            msg = (mach_msg_header_t *)msg_buffer;
            /// 5. 如果有 Source1 (基于port) 处于 ready 状态，直接处理这个 Source1 然后跳转去处理消息。
            if (__CFRunLoopServiceMachPort(dispatchPort, &msg, sizeof(msg_buffer), &livePort, 0)) {
                //如果接收到了消息的话，前往第9步开始处理msg
                goto handle_msg;
            }
#elif DEPLOYMENT_TARGET_WINDOWS
            if (__CFRunLoopWaitForMultipleObjects(NULL, &dispatchPort, 0, 0, &livePort, NULL)) {
                goto handle_msg;
            }
#endif
        }
// ...
```


#### RunLoop和Timer的关系
首先timer要加入到runLoop的其中一个mode中，也就是加入到当前mode的items中；在runLoopRun的时候，执行doBlock，然后while循环items，block调用。
NSTimer：
```
    NSTimer *timer = [NSTimer timerWithTimeInterval:1 repeats:YES block:^(NSTimer * _Nonnull timer) {
        NSLog(@"hell timer -- %@",[[NSRunLoop currentRunLoop] currentMode]);
    }];
    [[NSRunLoop currentRunLoop] addTimer:timer forMode:NSRunLoopCommonModes];
```

timer的底层CFRunLoopTimerRef：
```
- (void)cfTimerDemo{
    CFRunLoopTimerContext context = {
        0,
        ((__bridge void *)self),
        NULL,
        NULL,
        NULL
    };
    CFRunLoopRef rlp = CFRunLoopGetCurrent();
    /**
     参数一:用于分配对象的内存
     参数二:在什么是触发 (距离现在)
     参数三:每隔多少时间触发一次
     参数四:未来参数
     参数五:CFRunLoopObserver的优先级 当在Runloop同一运行阶段中有多个CFRunLoopObserver 正常情况下使用0
     参数六:回调,比如触发事件,就会执行
     参数七:上下文记录信息
     */
    CFRunLoopTimerRef timerRef = CFRunLoopTimerCreate(kCFAllocatorDefault, 0, 1, 0, 0, runLoopTimerCallBack, &context);
    CFRunLoopAddTimer(rlp, timerRef, kCFRunLoopDefaultMode);

}

void runLoopTimerCallBack(CFRunLoopTimerRef timer, void *info){
    NSLog(@"%@---%@",timer,info);
}

```

我们再次查找RunLoop的addTimer方法`CFRunLoopAddTimer`（当然也有AddObserver、AddSource等），先判断kCFRunLoopCommonModes是否相同，如果不是则进行查找，其中`CFSetAddValue`把CFRunLoopTimerRef对象保存在items中，`CFSetApplyFunction`再把刚加进来的item储存到commonModes中。
```
void CFRunLoopAddTimer(CFRunLoopRef rl, CFRunLoopTimerRef rlt, CFStringRef modeName) {
    CHECK_FOR_FORK();
    if (__CFRunLoopIsDeallocating(rl)) return;
    if (!__CFIsValid(rlt) || (NULL != rlt->_runLoop && rlt->_runLoop != rl)) return;
    __CFRunLoopLock(rl);
    if (modeName == kCFRunLoopCommonModes) {
        CFSetRef set = rl->_commonModes ? CFSetCreateCopy(kCFAllocatorSystemDefault, rl->_commonModes) : NULL;
        if (NULL == rl->_commonModeItems) {
            rl->_commonModeItems = CFSetCreateMutable(kCFAllocatorSystemDefault, 0, &kCFTypeSetCallBacks);
        }
        CFSetAddValue(rl->_commonModeItems, rlt);
        if (NULL != set) {
            CFTypeRef context[2] = {rl, rlt};
            /* add new item to all common-modes */
            CFSetApplyFunction(set, (__CFRunLoopAddItemToCommonModes), (void *)context);
            CFRelease(set);
        } else {
        CFRunLoopModeRef rlm = __CFRunLoopFindMode(rl, modeName, true);
    // ...省略N行代码
```

把item添加到modes中后，`__CFRunLoopRun `方法有个重要的方法`CFRunLoopDoBlocks`，rl是runLoop，rlm是runLoopMode，把runLoopMode传给runLoop中，检查将执行哪个事务
```
__CFRunLoopDoBlocks(rl, rlm);   

// -----------------------------------

static Boolean __CFRunLoopDoBlocks(CFRunLoopRef rl, CFRunLoopModeRef rlm) { // Call with rl and rlm locked
    if (!rl->_blocks_head) return false;
    if (!rlm || !rlm->_name) return false;
    Boolean did = false;
    struct _block_item *head = rl->_blocks_head;
    struct _block_item *tail = rl->_blocks_tail;
    rl->_blocks_head = NULL;
    rl->_blocks_tail = NULL;
    CFSetRef commonModes = rl->_commonModes;
    CFStringRef curMode = rlm->_name;
    __CFRunLoopModeUnlock(rlm);
    __CFRunLoopUnlock(rl);
    struct _block_item *prev = NULL;
    struct _block_item *item = head;
    while (item) {
        struct _block_item *curr = item;
        item = item->_next;
        Boolean doit = false;
        if (CFStringGetTypeID() == CFGetTypeID(curr->_mode)) {
            doit = CFEqual(curr->_mode, curMode) || (CFEqual(curr->_mode, kCFRunLoopCommonModes) && CFSetContainsValue(commonModes, curMode));
        } else {
            doit = CFSetContainsValue((CFSetRef)curr->_mode, curMode) || (CFSetContainsValue((CFSetRef)curr->_mode, kCFRunLoopCommonModes) && CFSetContainsValue(commonModes, curMode));
        }
        if (!doit) prev = curr;
        if (doit) {
            if (prev) prev->_next = item;
            if (curr == head) head = item;
            if (curr == tail) tail = prev;
            void (^block)(void) = curr->_block;
            CFRelease(curr->_mode);
            free(curr);
            if (doit) {
                __CFRUNLOOP_IS_CALLING_OUT_TO_A_BLOCK__(block);
                did = true;
            }
            Block_release(block); // do this before relocking to prevent deadlocks where some yahoo wants to run the run loop reentrantly from their dealloc
        }
    }
    __CFRunLoopLock(rl);
    __CFRunLoopModeLock(rlm);
    if (head) {
        tail->_next = rl->_blocks_head;
        rl->_blocks_head = head;
        if (!rl->_blocks_tail) rl->_blocks_tail = tail;
    }
    return did;
}
```
__CFRunLoopDoBlocks中通过链表遍历item， 判断当前的runLoopMode和加入的runLoopMode或者CFRunLoopCommonModes是否相同，执行`doit`，进入`__CFRUNLOOP_IS_CALLING_OUT_TO_A_BLOCK__ `,执行block
```
static void __CFRUNLOOP_IS_CALLING_OUT_TO_A_BLOCK__() __attribute__((noinline));
static void __CFRUNLOOP_IS_CALLING_OUT_TO_A_BLOCK__(void (^block)(void)) {
    if (block) {
        block();
    }
    getpid(); // thwart tail-call optimization
}
```
举个例子，把定时器添加到RunLoop中，timer加入的runLoopMode类型`NSDefaultRunLoopMode`，和当前runLoopMode的类型(runLoopMode可以切换，比如默认kCFRunLoopDefaultMode类型，滑动的时候UITrackingRunLoopMode，启动时UIInitializationRunLoopMode)比较，默认情况下执行timer，当页面滑动的时候，当前runLoopMode的类型自动切换到`UITrackingRunLoopMode`，因此timer失效，停止滑动时，当前runLoopMode的类型切换到`NSDefaultRunLoopMode`，timer恢复。当然了，如果我们把timer加入到UITrackingRunLoopMode模式时，那么只有在滑动的时候才执行。如果想在默认情况下和滑动的时候都执行，就要把timer加入到占位模式NSRunLoopCommonModes中，NSRunLoopCommonModes相当于Mode集合，这样就可以在两个模式下都执行了，__这就是为什么定时器不准的原因与解决办法__。



#### RunLoop和Source的关系
__CFRunLoopSource也是一个结构体，其中有一个union属性，它包含了version0和version1，也就是Source0（CFRunLoopSourceContext）和Source1（CFRunLoopSourceContext1）。进入源码
```
struct __CFRunLoopSource {
    CFRuntimeBase _base;
    uint32_t _bits;
    pthread_mutex_t _lock;
    CFIndex _order;            /* immutable */
    CFMutableBagRef _runLoops;
    union {
        CFRunLoopSourceContext version0;    /* immutable, except invalidation */
        CFRunLoopSourceContext1 version1;    /* immutable, except invalidation */
    } _context;
};
```

##### Source0分析
Source0是用来处理APP内部事件、APP自己负责管理，比如UIevent。
__调用底层：__因为source0只包含一个回调（函数指针）它并不能主动触发事件；CFRunLoopSourceSignal（source）将这个事件标记为待处理；CFRunLoopWakeUp来唤醒runloop，让他处理事件。首先创建一个Source0并添加到当前的runLoop中，执行信号，标记待处理CFRunLoopSourceSignal，再唤醒runloop去处理CFRunLoopWakeUp，通过`CFRunLoopRemoveSource`来取消移除源，CFRelease(rlp)。打印结果会显示`准备执行`和`取消了,终止了!!!`，如果注释掉`CFRunLoopRemoveSource`，则会打印`准备执行`和`执行啦`。
```
- (void)source0Demo{
    
    CFRunLoopSourceContext context = {
        0,
        NULL,
        NULL,
        NULL,
        NULL,
        NULL,
        NULL,
        schedule,
        cancel,
        perform,
    };
    /**
     
     参数一:传递NULL或kCFAllocatorDefault以使用当前默认分配器。
     参数二:优先级索引，指示处理运行循环源的顺序。这里传0为了的就是自主回调
     参数三:为运行循环源保存上下文信息的结构
     */
    CFRunLoopSourceRef source0 = CFRunLoopSourceCreate(CFAllocatorGetDefault(), 0, &context);
    CFRunLoopRef rlp = CFRunLoopGetCurrent();
    // source --> runloop 指定了mode  那么此时我们source就进入待绪状态
    CFRunLoopAddSource(rlp, source0, kCFRunLoopDefaultMode);
    // 一个执行信号
    CFRunLoopSourceSignal(source0);
    // 唤醒 run loop 防止沉睡状态
    CFRunLoopWakeUp(rlp);
    // 取消 移除
    CFRunLoopRemoveSource(rlp, source0, kCFRunLoopDefaultMode);
    CFRelease(rlp);
}

void schedule(void *info, CFRunLoopRef rl, CFRunLoopMode mode){
    NSLog(@"准备执行");
}

void perform(void *info){
    NSLog(@"执行啦");
}

void cancel(void *info, CFRunLoopRef rl, CFRunLoopMode mode){
    NSLog(@"取消了,终止了!!!");
}
```

##### Source1分析
Source1被用于通过内核和其他线程相互发送消息。
__调用底层：__Source1包含一个 mach_port和一个回调（函数指针）
当然了，线程间的通讯除了可以通过以下方式：
```
// 主线 -- 子线程
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        NSLog(@"%@", [NSThread currentThread]); // 3

        NSString *str;
        dispatch_async(dispatch_get_main_queue(), ^{
            // 1
            NSLog(@"%@", [NSThread currentThread]);

        });
    });
```

还可以通过更加底层、更加接近内核的NSPort方式，NSPort是source1类型，通过addPort添加到runLoop中去。再添加子线程，子线程中再加入port。
```
- (void)setupPort{
    
    self.mainThreadPort = [NSPort port];
    self.mainThreadPort.delegate = self;
    // port - source1 -- runloop
    [[NSRunLoop currentRunLoop] addPort:self.mainThreadPort forMode:NSDefaultRunLoopMode];

    [self task];
}

- (void) task {
    NSThread *thread = [[NSThread alloc] initWithBlock:^{
        self.subThreadPort = [NSPort port];
        self.subThreadPort.delegate = self;
        
        [[NSRunLoop currentRunLoop] addPort:self.subThreadPort forMode:NSDefaultRunLoopMode];
        [[NSRunLoop currentRunLoop] run];
    }];
    
    [thread start];
}
``` 

子线程给主线程发送消息响应。
```
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    
    NSMutableArray* components = [NSMutableArray array];
    NSData* data = [@"hello" dataUsingEncoding:NSUTF8StringEncoding];
    [components addObject:data];
    
    [self.subThreadPort sendBeforeDate:[NSDate date] components:components from:self.mainThreadPort reserved:0];
}
```
当NSPort对象接收到端口消息时，会调起`handlePortMessage`，__官方文档如下解释：__
> When an `NSPort` object receives a port message, it forwards the message to its delegate in a [`handle<wbr>Mach<wbr>Message:`](apple-reference-documentation://hcZ8Ot65Ji) or [`handle<wbr>Port<wbr>Message:`](apple-reference-documentation://hcoGVv2jBt) message. The delegate should implement only one of these methods to process the incoming message in whatever form desired. [`handle<wbr>Mach<wbr>Message:`](apple-reference-documentation://hcZ8Ot65Ji) provides a message as a raw Mach message beginning with a `msg_header_t` structure. [`handle<wbr>Port<wbr>Message:`](apple-reference-documentation://hcoGVv2jBt) provides a message as an [`NSPort<wbr>Message`](apple-reference-documentation://hcI7iWzk8C) object, which is an object-oriented wrapper for a Mach message. If a delegate has not been set, the `NSPort` object handles the message itself.

端口接收到消息后会打印message内部属性：`localPort`、`components`、`remotePort`等，睡眠一秒后，主线程再向子线程发送消息。
```
- (void)handlePortMessage:(id)message {
    NSLog(@"%@", [NSThread currentThread]); // 3 1

    unsigned int count = 0;
    Ivar *ivars = class_copyIvarList([message class], &count);
    for (int i = 0; i<count; i++) {
        
        NSString *name = [NSString stringWithUTF8String:ivar_getName(ivars[i])];
        NSLog(@"%@",name);
    }
    
    sleep(1);
    if (![[NSThread currentThread] isMainThread]) {

        NSMutableArray* components = [NSMutableArray array];
        NSData* data = [@"woard" dataUsingEncoding:NSUTF8StringEncoding];
        [components addObject:data];

        [self.mainThreadPort sendBeforeDate:[NSDate date] components:components from:self.subThreadPort reserved:0];
    }
}
```
该文章为记录本人的学习路程，希望能够帮助大家！！！

