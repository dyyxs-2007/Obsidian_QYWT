> 🍎 先贴一个Apple官方文档[RunLoop](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Multithreading/RunLoopManagement/RunLoopManagement.html#//apple_ref/doc/uid/10000057i-CH16-SW23)，以及开源的CoreFoundation下载地址https://github.com/opensource-apple/CF/tree/master，里面包含RunLoop的实现

# 概念

## Loop机制

```objc
int main(int argc, char * argv[]) {
    @autoreleasepool {
        return UIApplicationMain(argc, argv, nil, NSStringFromClass([AppDelegate class]));
    }
}
```

这是一个iOS程序的main函数

一般来说，程序执行完毕就会关闭，但是由于app需要我们手动交互，处理各种用户与其交互的事件，所以我们需要让程序暂时无法退出，并且随时能响应事件，这样的功能通常叫做 [Event Loop](http://en.wikipedia.org/wiki/Event_loop)，很多框架也使用了这种模型，RunLoop就是基于这种思路搭建的

> EventLoop

```objc
loop() {
    do {
        message = getNextMessage();
        processMessage(message);
    } while (message != quit);
}
```

所以RunLoop核心就是一个do-While循环，在循环中一次次处理交互事件

- 通常我们说的RunLoop指的是NSRunLoop或者CFRunLoopRef，后者是纯C的函数，而NSRunLoop只是对其的OC封装，不提供额外的功能，因此我们主要分析的就是CFRunLoopRef

事实上，CFRunLoopRef是一个结构体指针，指向**__CFRunloop**结构体，按照OC对象的思路，RunLoop的运行就是这个指针的运行，运行的核心方法是**__CFRunloopRun()**，这是一个c函数，调用之后就会启动RunLoop，这里放的伪代码是对源代码解析后的结果

```objc
int32_t __CFRunLoopRun() {
    // 通知即将进入runloop
    __CFRunLoopDoObservers(KCFRunLoopEntry);
    do
    {
        // 通知将要处理timer和source
        __CFRunLoopDoObservers(kCFRunLoopBeforeTimers);
        __CFRunLoopDoObservers(kCFRunLoopBeforeSources);
        
        // 处理非延迟的主线程调用
        __CFRunLoopDoBlocks();
        // 处理Source0事件
        __CFRunLoopDoSource0();
        
        if (sourceHandledThisLoop) {
            __CFRunLoopDoBlocks();
	     }
        /// 如果有 Source1 (基于port) 处于 ready 状态，直接处理这个 Source1 然后跳转去处理消息。
        if (__Source0DidDispatchPortLastTime) {
            Boolean hasMsg = __CFRunLoopServiceMachPort();
            if (hasMsg) goto handle_msg;
        }
            
        /// 通知 Observers: RunLoop 的线程即将进入休眠(sleep)。
        if (!sourceHandledThisLoop) {
            __CFRunLoopDoObservers(runloop, currentMode, kCFRunLoopBeforeWaiting);
        }
            
        // GCD dispatch main queue
        CheckIfExistMessagesInMainDispatchQueue();
        
        // 即将进入休眠
        __CFRunLoopDoObservers(kCFRunLoopBeforeWaiting);
        
        // 等待内核mach_msg事件
        mach_port_t wakeUpPort = SleepAndWaitForWakingUpPorts();
        
        // 等待。。。
        
        // 从等待中醒来
        __CFRunLoopDoObservers(kCFRunLoopAfterWaiting);
        
        // 处理因timer的唤醒
        if (wakeUpPort == timerPort)
            __CFRunLoopDoTimers();
        
        // 处理异步方法唤醒,如dispatch_async
        else if (wakeUpPort == mainDispatchQueuePort)
            __CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__()
            
        // 处理Source1
        else
            __CFRunLoopDoSource1();
        
        // 再次确保是否有同步的方法需要调用
        __CFRunLoopDoBlocks();
        
    } while (!stop && !timeout);
    
    // 通知即将退出runloop
    __CFRunLoopDoObservers(CFRunLoopExit);
}
```

看起来很神秘，但暂时无需逐行阅读，我们需要先搞清楚其中的概念和组成

## CFRunLoopRef组成

**CFRunLoopRef**主要由这些内容组成：

1. Source-输入源
2. Timer-时间源
3. Observer-观察者
4. Mode-模式

输入源会将硬件接收到的信号传入RunLoop待响应，时间源和我们所熟悉的Timer有关，而在RunLoop每次处理时间，休眠这些时间节点会给观察者发送消息，最后就是模式，有默认模式的RunLoop也有在拖动时特定的RunLoop

> 🍎 以上这些内容的用途功能会在下面进行详细解释，学习完这些之后就能拼凑起来看懂上面的代码

- 在这之前，我们需要先理解CFRunLoopRef的创建过程以及RunLoop与线程之间的关系

# RunLoop与线程的关系

苹果不允许我们直接创建一个RunLoop，但是开放了两个获取CFRunLoopRef的函数，分别是**CFRunLoopGetMain()， CFRunLoopGetCurrent()**，这两个函数的内部逻辑大概是这样：

```objc
/// 全局的Dictionary，key 是 pthread_t， value 是 CFRunLoopRef
static CFMutableDictionaryRef loopsDic;
/// 访问 loopsDic 时的锁
static CFSpinLock_t loopsLock;
 
/// 获取一个 pthread 对应的 RunLoop。
CFRunLoopRef _CFRunLoopGet(pthread_t thread) {
    OSSpinLockLock(&loopsLock);
    
    if (!loopsDic) {
        // 第一次进入时，初始化全局Dic，并先为主线程创建一个 RunLoop。
        loopsDic = CFDictionaryCreateMutable();
        CFRunLoopRef mainLoop = _CFRunLoopCreate();
        CFDictionarySetValue(loopsDic, pthread_main_thread_np(), mainLoop);
    }
    
    /// 直接从 Dictionary 里获取。
    CFRunLoopRef loop = CFDictionaryGetValue(loopsDic, thread));
    
    if (!loop) {
        /// 取不到时，创建一个
        loop = _CFRunLoopCreate();
        CFDictionarySetValue(loopsDic, thread, loop);
        /// 注册一个回调，当线程销毁时，顺便也销毁其对应的 RunLoop。
        _CFSetTSD(..., thread, loop, __CFFinalizeRunLoop);
    }
    
    OSSpinLockUnLock(&loopsLock);
    return loop;
}
 
CFRunLoopRef CFRunLoopGetMain() {
    return _CFRunLoopGet(pthread_main_thread_np());
}
 
CFRunLoopRef CFRunLoopGetCurrent() {
    return _CFRunLoopGet(pthread_self());
}
```

- 从上面我们可以看出，RunLoop与线程是一一对应式的绑定的，但是并不是每一个线程都有自己的RunLoop，为什么这样说呢？

首先，这个方法的初衷是什么，是获取一个RunLoop，但是一个线程天生的不会有RunLoop，如果每当创建一个线程的时候就默认创建出来一个对应的RunLoop，这样做很浪费性能，所以苹果在这里采用了懒加载

并且使用线程作为字典的key，这样假如在某一个线程调用这个获取函数，就会创建出来属于这个线程的RunLoop(假如是第一次调用这个函数)

- 注意看第一个if语句，这里是判断全局字典有没有被创建，事实上，这个方法有可能会在任何一个线程使用，**但是第一次一定是在主线程，这是因为在app启动的时候，开头的main函数中的`UIApplicationMain()`方法会为主线程调用一次此函数**
- 所以当主线程第一次调用的时候会先初始化全局字典，然后给主线程配对一个RunLoop，也就是mainRunLoop

接着我们可以再思考一下，为什么必定要给主线程先创建一个RunLoop？原因其实就是我们一开始的需求，让程序在无人操作的时候休息，需要让它干活的时候又能立马响应，即让主线程一直在RunLoop中运行，如下图所示

![pasted-image.png · 123](https://img.cdn1.vip/i/6a995b980afc5_1788435352.webp)

> 简述：对一个非主线程而言，本身不具有RunLoop，除非手动调用CFRunLoopGetMain()， CFRunLoopGetCurrent()函数，会创建并返回该线程专属的CFRunLoopRef

- 理解这一层之后，我们就可以学习RunLoop各个组成部分的作用

# RunLoop对外接口

在CoreFoundation里面关于RunLoop有五个类：

- [ ] CFRunLoopRef
- [ ] CFRunLoopModeRef
- [ ] CFRunLoopSourceRef
- [ ] CFRunLoopTimerRef
- [ ] CFRunLoopObserverRef

其中 CFRunLoopModeRef 类不对外暴露，只是通过 CFRunLoopRef 的接口进行了封装。他们的关系如下:

![pasted-image.png · 56](https://img.cdn1.vip/i/6a995f58b589a_1788436312.webp)

一个RunLoop包含了若干个Mode(本图仅作示例，事实上一个RunLoop可以有很多个mode)，而每个Mode中包含了若干个Source/Timer/Observer。每次调用RunLoop的主函数的时候，只能选择一个mode进入，这个mode就是CurrentMode，代表当前模式，如果需要切换mode，就需要先退出Loop再重新指定mode进入，这样做的目的是隔离不同mode下的观察者、事件源等内容，防止不同模式下的消息互相影响

## ModeItem的作用

### CFRunLoopTimerRef

这是一个基于时间的触发器，包含一个时间长度和一个函数指针，当它被加入到RunLoop的时候，RunLoop会注册对应的时间点，时间点一到，RunLoop就会被唤醒以执行回调

可以使用NSTimer来设置，不过除了 `scheduledTimerWithTimeInterval`开头的方法创建的 Timer 都需要手动添加到当前 RunLoop 中。（`scheduledTimerWithTimeInterval` 创建的 Timer 会自动以 Default Mode 加载到当前 RunLoop中

> 这也可以解释我们在UI学习中涉及到的另一个问题：当拖动屏幕的时候，以NSTimer为定时器的轮播图不滚动
>
> 这个问题的原因就是我们直接使用的NSTimer，默认作为TimerSource加入到了DefaultMode中，当手指拖动的时候，由于要切换模式，从DefaultMode换成TrackingMode，所以原本的DefaultMode中的TimerSource没有被识别到，视觉上就是计时器失效  轮播图静止

### CFRunLoopSourceRef

这是事件产生的地方，Source有两个版本，分别是Source0和Source1

Source0和Source1的区别就是能否唤醒线程，底层的区别就是是否包含mach_port

- Source0：仅包含一个回调(函数指针)，使用的时候由于它不能唤醒线程，所以需要手动调用CFRunLoopWakeUp(runloop) 来唤醒 RunLoop，让线程处理
- Source1：包含一个mach_port和一个回调，主要来源是内核和其他线程，可以主动唤醒RunLoop

### CFRunLoopObserverRef

这 是观察者，每个 Observer 都包含了一个回调（函数指针），当 RunLoop 的状态发生变化时，观察者就能通过回调接受到这个变化

- 以上三个内容被统称为ModeItem，也就是mode中的内容，一个 item 可以被同时加入多个 mode。但一个 item 被重复加入同一个 mode 时是不会有效果的。如果一个 mode 中一个 item 都没有，则 RunLoop 会直接退出，不进入循环。

![pasted-image.png · 84](https://img.cdn1.vip/i/6a996b6b794c3_1788439403.webp)

## RunLoop的Mode

- RunLoop和Mode的定义如下

```objc
struct __CFRunLoop {
	    CFRuntimeBase _base;
	    pthread_mutex_t _lock;			/* locked for accessing mode list */
	    __CFPort _wakeUpPort;			// used for CFRunLoopWakeUp 
	    Boolean _unused;
	    volatile _per_run_data *_perRunData;              // reset for runs of the run loop
	    pthread_t _pthread;
	    uint32_t _winthread;
	    CFMutableSetRef _commonModes;
	    CFMutableSetRef _commonModeItems;
	    CFRunLoopModeRef _currentMode;   //<---CurrentMode
	    CFMutableSetRef _modes;
	    struct _block_item *_blocks_head;
	    struct _block_item *_blocks_tail;
	    CFAbsoluteTime _runTime;
	    CFAbsoluteTime _sleepTime;
	    CFTypeRef _counterpart;
	};
	
	struct __CFRunLoopMode {
	    CFRuntimeBase _base;
	    pthread_mutex_t _lock;	/* must have the run loop locked before locking this */
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

系统默认定义了以下几个 mode：

- kCFRunLoopDefaultMode：App：App的默认 Mode，通常主线程是在这个 Mode 下运行的
- UITrackingRunLoopMode：界面跟踪 Mode，用于 ScrollView 追踪触摸滑动，保证界面滑动时不受其他 Mode 影响
- UIInitializationRunLoopMode：在刚启动 App 时第进入的第一个 Mode，启动完成后就不再使用（私有）
- GSEventReceiveRunLoopMode：接受系统事件的内部 Mode，通常用不到
- kCFRunLoopCommonModes：这是一个占位的 Mode，包含上面的 kCFRunLoopDefaultMode 以及 UITrackingRunLoopMode

但注意，**不是说Runloop会运行在**kCFRunLoopCommonModes**这种模式下，而是相当于分别注册了** NSDefaultRunLoopMode**和** UITrackingRunLoopMode，每当Mode内容发生变化的时候，RunLoop都会自动同步到Common标记下的所有Mode里面

# RunLoop流程

了解以上的内容之后就可以看懂一开始的代码的大致作用了

> 这里再放一次方便看，会以注释的形式将解读插入其中

```objc
int32_t __CFRunLoopRun() {
    // 通知即将进入runloop
    __CFRunLoopDoObservers(KCFRunLoopEntry);//这里就是给Observer发消息
    do
    {
        // 通知将要处理timer和source
        __CFRunLoopDoObservers(kCFRunLoopBeforeTimers);//Observer发消息+1
        __CFRunLoopDoObservers(kCFRunLoopBeforeSources);//Observer发消息+1
        
        // 处理非延迟的主线程调用
        __CFRunLoopDoBlocks();//先处理一次，如GCD提交的async代码
        // 处理Source0事件
        __CFRunLoopDoSource0();//source0是不会主动唤醒线程的source，这里直接进行处理
        
        if (sourceHandledThisLoop) {
            __CFRunLoopDoBlocks();//这里再调用一次的原因是source0里面可能会有和UI更新有关的GCDBlock，source0被处理之后block提交到链表后面，这里再进行一次block的处理
	     }
        /// 如果有 Source1 (基于port) 处于 ready 状态，直接处理这个 Source1 然后跳转去处理消息。
        if (__Source0DidDispatchPortLastTime) {
            Boolean hasMsg = __CFRunLoopServiceMachPort();
            if (hasMsg) goto handle_msg;
        }//虽说source1能唤醒线程，但是通俗的说能在线程醒着的时候直接处理source1是非常好的不需要唤醒的开销
            
        /// 通知 Observers: RunLoop 的线程即将进入休眠(sleep)。
        if (!sourceHandledThisLoop) {
            __CFRunLoopDoObservers(runloop, currentMode, kCFRunLoopBeforeWaiting);//Observer发消息+1
        }
            
        // GCD dispatch main queue
        CheckIfExistMessagesInMainDispatchQueue();//这里实际上是判断的GCD维护的main queue，GCD尚未提交所以并不归RunLoop管，但是看一眼的作用就是通过有没有将要提交的任务决定RunLoop要不要进行深度休眠，这样也可以节省唤醒线程的开销
        
        // 即将进入休眠
        __CFRunLoopDoObservers(kCFRunLoopBeforeWaiting);//Observer发消息+1
        
        // 等待内核mach_msg事件
        mach_port_t wakeUpPort = SleepAndWaitForWakingUpPorts();//mach_port机制，可以理解为内核与线程、线程与线程之间的消息传递站点
        
        // 等待。。。
        
        // 从等待中醒来
        __CFRunLoopDoObservers(kCFRunLoopAfterWaiting);//Observer发消息+1
        
        // 处理因timer的唤醒
        if (wakeUpPort == timerPort)
            __CFRunLoopDoTimers();//时间源唤醒线程后处理Timer的回调
        
        // 处理异步方法唤醒,如dispatch_async
        else if (wakeUpPort == mainDispatchQueuePort)
            __CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__()//GCD提交任务唤醒线程处理提交的block
            
        // 处理Source1
        else
            __CFRunLoopDoSource1();//内核/线程之间的回调
        
        // 再次确保是否有同步的方法需要调用
        __CFRunLoopDoBlocks();//这一步也是处理上面几个回调中有可能产生的block
        
    } while (!stop && !timeout);//循环结束，开始下一轮，这时候没有被完成的block将会在下一轮中再次被处理
    
    // 通知即将退出runloop
    __CFRunLoopDoObservers(CFRunLoopExit);//Observer发消息+1
}
```

上面我对一开始的代码做了详尽的注释，可以看到每次在RunLoop要进行操作的时候会先给Observer报备一下自己在干嘛，然后就开始干活，在每次循环的开始都会趁醒着处理事件，包括不能唤醒线程的source0以及突然的source1

值得注意的是处理Block待办的时候，足足写了三次，第一次是处理正常提交的block

第二次是在处理了source0之后，**这是因为source0叫作软件事件，UIKit的触摸，滚动，手势事件会在App层封装成source0，由source0提交给RunLoop**，所以source0一般里面还会嵌套一个async_dispatch来更新UI等操作，趁着RunLoop还没睡直接进行更新

第三次是在处理了被唤醒事件之后，这里也是一样的道理，处理完唤醒线程的回调，也可能会产生block，这里再统一运行，假如再产生的block里面又嵌套了一层block，那就只能等下一个循环来处理了

![pasted-image.png · 196](https://img.cdn1.vip/i/6a9ac3d0de12d_1788527568.webp)

## 触摸事件的调用流程

**或许你会疑惑**：刚刚讲到了source0是软件事件，处理的是UI等事件内容，然而前面我们又知道，source0没有搭载能唤醒线程的port，所以按这样想，当我点击屏幕等触发手势，用source0传给RunLoop，但是RunLoop可能在睡觉，**这时候source0不会被处理，除非有一个port把它叫醒，也就是说，UI并不会第一时间被更新**

对吗？

显然这样的设计是有问题的，所以在实际的实现中，触摸屏幕事件被这样传递：

1. 硬件检测到手势信息，上报给系统内核
2. 系统内核(IOKit)接收到硬件的信息，打包成一个 IOHIDEvent事件 用mach_port发送给主线程的port
3. 由于这里使用了port发送属于内核/线程之间的消息，会唤醒睡着的RunLoop，于是RunLoop会先处理这个传递过来的事件(source1)

>  但是不是说source0才是处理UI时间的source吗？

4. 这是因为这里source1回调的内容就是把触摸事件投递成一个source0事件，然后提交到RunLoop的source0列表里面，所以说source0是处理UI更新的source
5. 这时候RunLoop处于醒着的状态，接着处理source0，就完成了UI的更新

# RunLoop的应用层

## AutoreleasePool

自动释放池想必大家并不陌生，其实它的机制和RunLoop也有关系，首先我们需要介绍这两个函数

`_objc_autoreleasePoolPush()`

`_objc_autoreleasePoolPop() `

在ARC下，clang会把`@autoreleasepool`(也就是平时用的自动释放池)编译成这样：

```objc
void *pool = _objc_autoreleasePoolPush();
// 原本括号中的代码
_objc_autoreleasePoolPop(pool);
```

> 当然这两个函数再深究的话会创建哨兵以及分页等具体释放，这里就不深入了

当App启动之后，主线程会被注册两个Observer，他们的回调都是`_wrapRunLoopWithAutoreleasePoolHandler()`

第一个Observer监听的是RunLoop刚启动的那个Entry，回调中，会调用`_objc_autoreleasePoolPush()`创建一个自动释放池，也就是上面展示编译后代码的第一行，使用order限定了其回调优先级最高，发生在所有回调以前

第二个Observer监听了两个事件，第一个是BeforeWaiting准备进入休眠以及Exit退出RunLoop，回调就会调用`_objc_autoreleasePoolPop() `，也就是编译后代码的第三行，这样一开一关就把本身的代码放在了push和pop之间，恰好达到了自动释放池的格式要求，并且对应的，它的order保证其优先级最低，发生在所有回调之后

- 在主线程执行的代码，通常会写在各种回调里面，所以天然的会被我们上面生产的自动释放池代码包裹住，所以不会出现内存泄漏，开发者也不用自己创建pool了

## Timer定时器

NSTimer 其实就是 CFRunLoopTimerRef，当你把一个NSTimer注册到RunLoop里面的时候，RunLoop会在时间重复的节点上提前注册好事件，例如10:00, 10:10, 10:20 这几个时间点

但是Timer有一个Tolerance (宽容度)的属性，这个属性告诉Timer到了规定时间点后能等多久，一旦等的太久了就下一次规定时间再运行，就像是等公交车，一趟不行就只能等下一趟

如果在运行Timer事件以前恰好在进行一个长时间任务，这时候已经超过了Tolerance，这次的Timer回调就不会被运行，就像是这次的公交车已经过站，直到下一次准时到达

但是一旦在Tolerance允许范围内另一个事件刚好结束了，这时候还是会运行已经偏离准确时间的Timer回调，这也就是我们常说Timer不准确的原因
