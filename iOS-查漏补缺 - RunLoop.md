`RunLoop` 在 0202 年的今天其实已经不是个新鲜的话题了，关于这方面的文章网上有很多大神总结得非常精辟。

作为 `iOS` 查漏补缺系列，这篇文章是笔者探索 `RunLoop` 底层的一些知识点总结，同时也借鉴了网上一些优秀的 `RunLoop` 技术文章的内容。

本文内容如有错误，欢迎指正。
<!-- more -->
# RunLoop 前导知识

## iOS/OS X 系统架构

### iOS 进化史

![iOS Evolutionary history ](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200316102159.jpg)


> Mac OS Classic 拥有伟大的 GUI，但系统设计却非常糟糕，尤其是协作式的多任务系统和低效的内存管理，以今天的标准看非常原始。
> 
> NeXTSTEP 操作系统作为乔布斯回归苹果的嫁妆，连同架构中的 Mach 和 OC 等一起融合进了Mac OS Classic。与其说是 Mac OS Classic 融合了 NeXTSTEP，不如说是后者反客为主替换了前者，这个转变并不是一瞬间发生的。
>
> Mac OS Classic 拥有不错的 GUI 但系统设计糟糕，NeXTSTEP 设计很棒，但 GUI 非常平淡。这两个小众操作系统融合的结果是一个继承二者优点的成功得多的全新操作系统—Mac OS X
> 
> 全新的 Mac OS X 在设计与实现上都同 NeXTSTEP 非常接近，诸如 Cocoa、Mach 、Interface Builder 等核心组件都源自 NeXTSTEP
> 
> iOS 最初称为 iPhone OS,是 Mac OS X 应对移动平台的分支，iOS 拥有和 `Mac OS X`一样的操作系统层次结构以及相同的操作系统核心Dawin。

### iOS 系统架构

![iOS System Architecture](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200316102205.jpg)


> 苹果官方将整个系统大致划分为上述4个层次：
> 
> 应用层包括用户能接触到的图形应用，例如 Spotlight、Aqua(macOS)、SpringBoard(iOS) 等。
> 
> 应用框架层即开发人员接触到的 Cocoa 等框架。
> 
> 核心框架层包括各种核心框架、OpenGL 等内容。
> 
> Darwin层是操作系统核心，包括 XNU 内核(2017 年已开源)、驱动和 UNIX shell

### Darwin 架构

![Darwin Architecture](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200316102208.jpg)


> Darwin 的内核是 XNU，XNU is Not Unix。XNU 是两种技术的混合体，Mach和BSD。BSD 层确保了 Darwin 系统的 UNIX 特性，真正的内核是 Mach，但是对外部隐藏。BSD 以上属于用户态，所有的内容都可以被应用程序访问，而应用程序不能访问内核态。当需要从用户态切换到内核态的时候，需要通过 mach trap 实现切换。
> 
> ---《深入解析Mac OS X & iOS 操作系统》


> 其中，在硬件层上面的三个组成部分：Mach、BSD、IOKit (还包括一些上面没标注的内容)，共同组成了 XNU 内核。
> 
> XNU 内核的内环被称作 Mach，其作为一个微内核，仅提供了诸如处理器调度、IPC (进程间通信)等非常少量的基础服务。
>
> BSD 层可以看作围绕 Mach 层的一个外环，其提供了诸如进程管理、文件系统和网络等功能。
> 
> IOKit 层是为设备驱动提供了一个面向对象(C++)的一个框架。
> 
> Mach 本身提供的 API 非常有限，而且苹果也不鼓励使用 Mach 的 API，但是这些API非常基础，如果没有这些API的话，其他任何工作都无法实施。
> 在 Mach 中，所有的东西都是通过自己的对象实现的，进程、线程和虚拟内存都被称为”对象”。
> 和其他架构不同， Mach 的对象间不能直接调用，只能通过消息传递的方式实现对象间的通信。
> ”消息”是 Mach 中最基础的概念，消息在两个端口 (port) 之间传递，这就是 Mach 的 IPC (进程间通信) 的核心。
> 
> --- 《深入理解RunLoop》ibireme

## mach_msg 函数

`mach_msg` 函数位于 `mach` 内核的 `message.h` 头文件中

![-w525](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200316102155.jpg)

> 这是系统内核在某个 port 收发消息所使用的函数，理解这个函数对于理解 runloop 的运行机制非常重要。
> 
> 详细的说明可参考 [这里](http://web.mit.edu/darwin/src/modules/xnu/osfmk/man/mach_msg.html)。

```c
// mach 消息基础结构体
typedef struct{
  // 消息头部
	mach_msg_header_t       header;
	// 消息体
	mach_msg_body_t         body;
} mach_msg_base_t;

// mach 消息头结构体
typedef struct{
  // 消息头的 bits
	mach_msg_bits_t       msgh_bits;
	// 消息头的大小
	mach_msg_size_t       msgh_size;
	// 远程端口号
	mach_port_t           msgh_remote_port;
	// 本地端口号
	mach_port_t           msgh_local_port;
	// 凭证端口号
	mach_port_name_t      msgh_voucher_port;
	// 消息ID
	mach_msg_id_t         msgh_id;
} mach_msg_header_t;

// mach_msg 函数
mach_msg_return_t mach_msg(
    // 消息头
    mach_msg_header_t *msg,
    // 消息选项，可用来指定是发消息(MACH_SEND_MSG)还是收消息(MACH_RCV_MSG)
    mach_msg_option_t option,
    // 发消息时设置的缓存区大小，无符号整型
    mach_msg_size_t send_size,
    // 收消息时设置的缓存区大小，无符号整型
    mach_msg_size_t rcv_size,
    // 收消息的端口，无符号整型
    mach_port_name_t rcv_name,
    // 消息过期时间
    // 当 option 参数中包含 MACH_SEND_TIMEOUT 或 MACH_RCV_TIMEOUT 的时候，传入以毫秒单位的过期时间
    mach_msg_timeout_t timeout,
    // 接收通知的端口
    // 当 option 参数中包含 MACH_SEND_CANCEL 和 MACH_RCV_NOTIFY 时，设置接收通知的端口；否则，传入 MACH_PORT_NULL。
    mach_port_name_t notify);
```

> 可以简单的将 mach_msg 理解为**多进程之间的一种通信机制**，不同的进程可以使用**同一个消息队列**来交流数据，当使用 mach_msg 从消息队列里读取 msg 时，可以在参数中设置 timeout 值，在 timeout 之前如果没有读到 msg，当前线程会一直处于休眠状态。这也是 runloop 在没有任务可执行的时候，能够进入 sleep 状态的原因。
>
> --- 《解密 Runloop》MrPeak

--

> 为了实现消息的发送和接收，mach_msg() 函数实际上是调用了一个 Mach 陷阱 (trap)，即函数 mach_msg_trap()，陷阱这个概念在 Mach 中等同于系统调用。当你在用户态调用 mach_msg_trap() 时会触发陷阱机制，切换到内核态；内核态中内核实现的 mach_msg() 函数会完成实际的工作。
> 
> --- 《深入理解RunLoop》ibireme

## CoreFoundation 基础

CoreFoundation 是一套基于 `C` 的 `API`，为 `iOS` 提供了基本数据管理和服务功能。因为 CoreFoundation 对象是基于 C 实现的，也有引用计数的概念，而 Foundation 对象是基于 OC 实现的。两种对象在相互转换的时候需要用到三个关键字: __bridge、__bridge_retained、__bridge_transfer，下面进行简单的介绍。

### __bridge

```objectivec
id obj = [[NSObject alloc] init];
void *p = (__bridge void *)obj;
id o = (__bridge id)p;
```

将 Objective-C 的对象类型用 __bridge 转换为 void* 类型和使用 __unsafe_unretained 关键字修饰的变量是一样的。

> __bridge: 只做类型转换，不修改相关对象的引用计数，原来的 Core Foundation 对象在不用时，需要调用 CFRelease 方法

### __bridge_retained

```objectivec
d obj = [[NSObject alloc] init]; 
void *p = (__bridge_retained void *)obj;
```

从名字上我们应该能理解其意义：类型被转换时，其对象的所有权也将被变换后变量所持有。
如果是 MRC 中:
```objectivec
id obj = [[NSObject alloc] init];
void *p = obj;
[(id)p retain];
```
下面的例子验证了出了大括号的范围后，p 仍然指向一个有效的实体。说明她拥有该对象的所有权，该对象没有因为出其定义范围而被销毁。
```objectivec
void *p = 0;
{
 id obj = [[NSObject alloc] init];
 p = (__bridge_retained void *)obj;
}
NSLog(@"class=%@", [(__bridge id)p class]);
```

> __bridge_retained: 类型转换后，将相关对象的引用计数加 1，原来的 Core Foundation 对象在不用时，需要调用 CFRelease 方法

### __bridge_transfer

MRC 中：

```objectivec
// p 变量原先持有对象的所有权
id obj = (id)p;
[obj retain];
[(id)p release];
```

ARC 中:

```objectivec
// p 变量原先持有对象的所有权
id obj = (__bridge_transfer id)p;
```

可以看出来，__bridge_retained 是编译器替我们做了 retain 操作，而 __bridge_transfer 是替我们做了 release。

> __bridge_transfer: 类型转换后，将该对象的引用计数交给 ARC 管理，Core Foundation 对象在不用时，不需要调用 CFRelease 方法

### Toll-Free bridged

CoreFoundation 和 Foundation 之间有一个 `Toll-free bridging` 机制。
在 MRC 时代，可以直接通过类型强转完成两个框架对象的类型转换。但是在 ARC 时代，则需要使用 `Toll-free bridging` 机制提供的方法来操作。上面已经介绍了三种不同的转换机制，而实际上在 Foundation 中，有更简便的方法来操作。如下所示：

```objectivec
// After using a CFBridgingRetain on an NSObject, the caller must take responsibility for calling CFRelease at an appropriate time.
NS_INLINE CF_RETURNS_RETAINED CFTypeRef _Nullable CFBridgingRetain(id _Nullable X) {
    return (__bridge_retained CFTypeRef)X;
}

NS_INLINE id _Nullable CFBridgingRelease(CFTypeRef CF_CONSUMED _Nullable X) {
    return (__bridge_transfer id)X;
}
```

CFBridgingRetain 可以代替 __bridge_retained，CFBridgingRelease 可以代替 __bridge_transfer。

# RunLoop 初探

## 什么是 RunLoop

根据[苹果官方文档](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Multithreading/RunLoopManagement/RunLoopManagement.html#//apple_ref/doc/uid/10000057i-CH16-SW1)的定义

> Run loops are part of the fundamental infrastructure associated with threads.
>
> RunLoop 是与线程相关的基础结构。

> A run loop is an event processing loop that you use to schedule work and coordinate the receipt of incoming events. The purpose of a run loop is to keep your thread busy when there is work to do and put your thread to sleep when there is none.
> 
> RunLoop 是一个调度任务和处理事件的事件循环。其意义在于让线程有事做的时候繁忙起来，没事做的时候休眠。

了解过 `NodeJS` 事件循环机制的同学应该不会对 `事件循环` 这一名词陌生，用伪代码实现如下:

```c
    while(AppIsRunning) {
        id whoWakesMe = SleepForWakingUp();
        id event = GetEvent(whoWakesMe);
        HandleEvent(event);
    }
```

> 这份伪代码来自 [sunnyx](http://blog.sunnyxx.com/) 大神的 RunLoop 技术分享

可以看到，RunLoop 就是这么一个简单的工作原理。但是其内部实现要比上面的代码复杂得多。我们下一节将深入 RunLoop 底层一探究竟。

> OSX/iOS 系统中，提供了两个这样的对象：NSRunLoop 和 CFRunLoopRef。
> 
CFRunLoopRef 是在 CoreFoundation 框架内的，它提供了纯 C 函数的 API，所有这些 API 都是线程安全的。
>
>NSRunLoop 是基于 CFRunLoopRef 的封装，提供了面向对象的 API，但是这些 API 不是线程安全的。

![NSRunLoop And CFRunLoop](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200316102211.jpg)


## RunLoop 底层数据结构

![RunLoop](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200316102217.jpg)

如上图所示，每一个 RunLoop 内部有多个 mode，而每一个 mode 内部有 sources0，source1, observers 和 timers。mode 中的元素统称为 `mode item`。下面我们一起回顾下这五个类的数据结构定义:

* CFRunLoopRef
* CFRunLoopModeRef
* CFRunLoopSourceRef
* CFRunLoopTimerRef
* CFRunLoopObserverRef

### CFRunLoopRef

```c
typedef struct __CFRunLoop * CFRunLoopRef;

struct __CFRunLoop
{
    // CoreFoundation 中的 runtime 基础信息
    CFRuntimeBase _base;
    // 针对获取 mode 列表操作的锁
    pthread_mutex_t _lock; /* locked for accessing mode list */
    // 唤醒端口
    __CFPort _wakeUpPort;  // used for CFRunLoopWakeUp
    // 是否使用过
    Boolean _unused;
    // runloop 运行会重置的一个数据结构
    volatile _per_run_data *_perRunData; // reset for runs of the run loop
    // runloop 所对应线程
    pthread_t _pthread;
    uint32_t _winthread;
    // 存放 common mode 的集合
    CFMutableSetRef _commonModes;
    // 存放 common mode item 的集合
    CFMutableSetRef _commonModeItems;
    // runloop 当前所在 mode
    CFRunLoopModeRef _currentMode;
    // 存放 mode 的集合
    CFMutableSetRef _modes;
    
    // runloop 内部 block 链表表头指针
    struct _block_item *_blocks_head;
    // runloop 内部 block 链表表尾指针
    struct _block_item *_blocks_tail;
    // 运行时间点
    CFAbsoluteTime _runTime;
    // 休眠时间点
    CFAbsoluteTime _sleepTime;
    CFTypeRef _counterpart;
};

// 每次 RunLoop 运行后会重置
typedef struct _per_run_data
{
    uint32_t a;
    uint32_t b;
    uint32_t stopped;   // runloop 是否停止
    uint32_t ignoreWakeUps; // runloop 是否已唤醒
} _per_run_data;

// 链表节点
struct _block_item
{
    // 指向下一个 _block_item
    struct _block_item *_next;
    // 要么是 string 类型，要么是集合类型，也就是说一个 block 可能对应单个或多个 mode
    CFTypeRef _mode; // CFString or CFSet
    // 存放的真正要执行的 block
    void (^_block)(void);
};
```

通过上面的代码可以看出，CFRunLoopRef 其实是一个 `__CFRunLoop` 类型的结构体指针。`__CFRunLoop` 内部以下属性需要重点关注:

* `_pthread`
* `_commonModes`
* `_currentMode`
* `_modes`

### CFRunLoopModeRef

```c
typedef struct __CFRunLoopMode *CFRunLoopModeRef;

struct __CFRunLoopMode
{
    // CoreFoundation 中的 runtime 基础信息
    CFRuntimeBase _base;
    // 互斥锁，加锁前需要 runloop 先加锁
    pthread_mutex_t _lock; /* must have the run loop locked before locking this */
    // mode 的名称
    CFStringRef _name;
    // mode 是否停止
    Boolean _stopped;
    char _padding[3];
    // source0
    CFMutableSetRef _sources0;
    // source1
    CFMutableSetRef _sources1;
    // observers
    CFMutableArrayRef _observers;
    // timers
    CFMutableArrayRef _timers;
    CFMutableDictionaryRef _portToV1SourceMap;
    // port 的集合
    __CFPortSet _portSet;
    // observer 的 mask
    CFIndex _observerMask;
    // 如果定义了 GCD 定时器
#if USE_DISPATCH_SOURCE_FOR_TIMERS
    // GCD 定时器
    dispatch_source_t _timerSource;
    // 队列
    dispatch_queue_t _queue;
    // 当 GCD 定时器触发时设置为 true
    Boolean _timerFired; // set to true by the source when a timer has fired
    Boolean _dispatchTimerArmed;
#endif
// 如果使用 MK_TIMER
#if USE_MK_TIMER_TOO
    // MK_TIMER 的 port
    mach_port_t _timerPort;
    Boolean _mkTimerArmed;
#endif
#if DEPLOYMENT_TARGET_WINDOWS
    DWORD _msgQMask;
    void (*_msgPump)(void);
#endif
    // 定时器软临界点
    uint64_t _timerSoftDeadline; /* TSR */
    // 定时器硬临界点
    uint64_t _timerHardDeadline; /* TSR */
};
```

通过上面的代码可以看出，CFRunLoopModeRef 其实是一个 `__CFRunLoopMode` 类型的结构体指针。
`__CFRunLoopMode` 内部以下属性需要重点关注:

* `_sources0`
* `_sources1`
* `_observers`
* `_timers`

### CFRunLoopSourceRef

```c
typedef struct __CFRunLoopSource * CFRunLoopSourceRef;

struct __CFRunLoopSource
{
    // CoreFoundation 中的 runtime 基础信息
    CFRuntimeBase _base;
    uint32_t _bits;
    // 互斥锁
    pthread_mutex_t _lock;
    // source 的优先级，值为小，优先级越高
    CFIndex _order; /* immutable */
    // runloop 集合
    CFMutableBagRef _runLoops;
    // 一个联合体，说明 source 要么为 source0，要么为 source1
    union {
        CFRunLoopSourceContext version0;  /* immutable, except invalidation */
        CFRunLoopSourceContext1 version1; /* immutable, except invalidation */
    } _context;
};

typedef struct {
    CFIndex version;
    // source 的信息
    void *  info;
    const void *(*retain)(const void *info);
    void    (*release)(const void *info);
    CFStringRef (*copyDescription)(const void *info);
    // 判断 source 相等的函数
    Boolean (*equal)(const void *info1, const void *info2);
    CFHashCode  (*hash)(const void *info);
    void    (*schedule)(void *info, CFRunLoopRef rl, CFStringRef mode);
    void    (*cancel)(void *info, CFRunLoopRef rl, CFStringRef mode);
    // source 要执行的任务块
    void    (*perform)(void *info);
} CFRunLoopSourceContext;
```

通过上面的代码可以看出，CFRunLoopSourceRef 其实是一个 `__CFRunLoopSource` 类型的结构体指针。
`__CFRunLoopSource` 的结构相对来说要简单一点，有几个属性要重点关注：

* `_order`
* `_context`
* `CFRunLoopSourceContext` 下的 `perform` 函数指针

### CFRunLoopTimerRef

```c
typedef struct CF_BRIDGED_MUTABLE_TYPE(NSTimer) __CFRunLoopTimer * CFRunLoopTimerRef;

struct __CFRunLoopTimer
{
    // CoreFoundation 中的 runtime 基础信息
    CFRuntimeBase _base;
    uint16_t _bits;
    // 互斥锁
    pthread_mutex_t _lock;
    // timer 对应的 runloop
    CFRunLoopRef _runLoop;
    // timer 对应的 mode 集合
    CFMutableSetRef _rlModes;
    // 下一次触发时间点
    CFAbsoluteTime _nextFireDate;
    // 定时器每次任务的间隔
    CFTimeInterval _interval;        /* immutable */
    // 定时器所允许的误差
    CFTimeInterval _tolerance;       /* mutable */
    // 触发时间点
    uint64_t _fireTSR;               /* TSR units */
    // 定时器的优先级
    CFIndex _order;                  /* immutable */
    // 定时器所要执行的任务
    CFRunLoopTimerCallBack _callout; /* immutable */
    // 定时器上下文
    CFRunLoopTimerContext _context;  /* immutable, except invalidation */
};

typedef struct {
    CFIndex	version;
    void *	info;
    const void *(*retain)(const void *info);
    void	(*release)(const void *info);
    CFStringRef	(*copyDescription)(const void *info);
} CFRunLoopTimerContext;

typedef void (*CFRunLoopTimerCallBack)(CFRunLoopTimerRef timer, void *info);
```

通过上面的代码可以看出，CFRunLoopTimerRef 其实是一个 `__CFRunLoopTimer` 类型的结构体指针，并且这里 `__CFRunLoopTimer` 在前面还有一个 `CF_BRIDGED_MUTABLE_TYPE(NSTimer)`

```c
#define CF_BRIDGED_TYPE(T)		__attribute__((objc_bridge(T)))
#define CF_BRIDGED_MUTABLE_TYPE(T)	__attribute__((objc_bridge_mutable(T)))
#define CF_RELATED_TYPE(T,C,I)		__attribute__((objc_bridge_related(T,C,I)))
#else
#define CF_BRIDGED_TYPE(T)
#define CF_BRIDGED_MUTABLE_TYPE(T)
#define CF_RELATED_TYPE(T,C,I)
#endif
```

从这里说明 `CFRunLoopTimerRef` 与 `NSTimer` 是 `toll-free bridged` 的。

`__CFRunLoopTimer` 中有几个属性需要重点关注:

* _order
* _callout
* _context

### CFRunLoopObserverRef

```c
typedef struct __CFRunLoopObserver * CFRunLoopObserverRef;

struct __CFRunLoopObserver
{
    // CoreFoundation 中的 runtime 基础信息
    CFRuntimeBase _base;
    // 互斥锁
    pthread_mutex_t _lock;
    // observer 对应的 runloop
    CFRunLoopRef _runLoop;
    // observer 观察了多少个 runloop
    CFIndex _rlCount;
    CFOptionFlags _activities;          /* immutable */
    // observer 优先级
    CFIndex _order;                     /* immutable */
    // observer 回调函数
    CFRunLoopObserverCallBack _callout; /* immutable */
    // observer 上下文
    CFRunLoopObserverContext _context;  /* immutable, except invalidation */
};

typedef struct {
    CFIndex	version;
    void *	info;
    const void *(*retain)(const void *info);
    void	(*release)(const void *info);
    CFStringRef	(*copyDescription)(const void *info);
} CFRunLoopObserverContext;

typedef void (*CFRunLoopObserverCallBack)(CFRunLoopObserverRef observer, CFRunLoopActivity activity, void *info);
```
通过上面的代码可以看出，CFRunLoopObserverRef 其实是一个 `__CFRunLoopObserver` 类型的结构体指针。

`__CFRunLoopObserver` 中有几个属性需要重点关注:

* _order
* _callout
* _context

### CFRunLoopActivity

```c
/* Run Loop Observer Activities */
typedef CF_OPTIONS(CFOptionFlags, CFRunLoopActivity) {
    kCFRunLoopEntry = (1UL << 0),
    kCFRunLoopBeforeTimers = (1UL << 1),
    kCFRunLoopBeforeSources = (1UL << 2),
    kCFRunLoopBeforeWaiting = (1UL << 5),
    kCFRunLoopAfterWaiting = (1UL << 6),
    kCFRunLoopExit = (1UL << 7),
    kCFRunLoopAllActivities = 0x0FFFFFFFU
};
```

通过上面的源码可知，RunLoop 有以下状态

* kCFRunLoopEntry: 即将进入 Loop
* kCFRunLoopBeforeTimers: 即将处理 Timer
* kCFRunLoopBeforeSources: 即将处理 Source
* kCFRunLoopBeforeWaiting: 即将进入休眠
* kCFRunLoopAfterWaiting: 刚从休眠中唤醒
* kCFRunLoopExit: 即将推出 Loop


## RunLoop 与线程关系

要想搞清楚 RunLoop 与线程的关系，我们从 `CFRunLoopGetMain` 和 `CFRunLoopGetCurrent` 函数开始研究:

```c
CFRunLoopRef CFRunLoopGetMain(void)
{
    CHECK_FOR_FORK();
    // 声明一个空的 RunLoop 
    static CFRunLoopRef __main = NULL; // no retain needed
    // 如果 __main 为空，则调用 _CFRunLoopGet0 函数来获取 RunLoop
    if (!__main)
        __main = _CFRunLoopGet0(pthread_main_thread_np()); // no CAS needed
    return __main;
}

CFRunLoopRef CFRunLoopGetCurrent(void)
{
    CHECK_FOR_FORK();
    // 从 线程的 TSD(Thread Specific Data) 中获取 runloop
    CFRunLoopRef rl = (CFRunLoopRef)_CFGetTSD(__CFTSDKeyRunLoop);
    // 如果拿到了就返回
    if (rl)
        return rl;
    // 如果在 TSD 中没有，那么就调用 _CFRunLoopGet0 函数
    return _CFRunLoopGet0(pthread_self());
}
```

> `pthread_main_thread_np()` 获取的是主线程，其内部实现如下

> ```c
>#define pthread_main_thread_np() _CF_pthread_main_thread_np()

>CF_EXPORT pthread_t _CF_pthread_main_thread_np(void);
>pthread_t _CF_pthread_main_thread_np(void) {
>    return _CFMainPThread;
>}

>void __CFInitialize(void) {
 >   ...
    
 >   _CFMainPThread = pthread_self();
    
 >  ...
>}
```

我们接着探索 `_CFRunLoopGet0` 函数，其实现如下

```c
// 存储了 runloop 与线程的字典
static CFMutableDictionaryRef __CFRunLoops = NULL;
static CFLock_t loopsLock = CFLockInit;

// should only be called by Foundation
// t==0 is a synonym for "main thread" that always works
// t为0的话，表示是要获取 主线程对应的 RunLoop
CF_EXPORT CFRunLoopRef _CFRunLoopGet0(pthread_t t)
{
    // 如果传入的现场为空，则再通过 pthread_main_thread_np() 获取到主线程
    if (pthread_equal(t, kNilPthreadT))
    {
        t = pthread_main_thread_np();
    }
    // 加锁
    __CFLock(&loopsLock);
    // 如果 __CFRunLoops 字典为空，则初始化字典，并把 mainLoop 加入字典
    if (!__CFRunLoops)
    {
        // 解锁
        __CFUnlock(&loopsLock);
        // 初始化一个 可变字典
        CFMutableDictionaryRef dict = CFDictionaryCreateMutable(kCFAllocatorSystemDefault, 0, NULL, &kCFTypeDictionaryValueCallBacks);
        // 拿到主线程对应的 RunLoop
        CFRunLoopRef mainLoop = __CFRunLoopCreate(pthread_main_thread_np());
        // 将 runloop 和 线程存入字典中，线程为 key，runloop 为 value
        CFDictionarySetValue(dict, pthreadPointer(pthread_main_thread_np()), mainLoop);
        if (!OSAtomicCompareAndSwapPtrBarrier(NULL, dict, (void *volatile *)&__CFRunLoops))
        {
            // 释放 字典
            CFRelease(dict);
        }
        // 释放 mainLoop
        CFRelease(mainLoop);
        // 加锁
        __CFLock(&loopsLock);
    }
    // 根据传入的 线程 t 从字典中获取 loop 对象
    CFRunLoopRef loop = (CFRunLoopRef)CFDictionaryGetValue(__CFRunLoops, pthreadPointer(t));
    // 对 loops 字典解锁
    __CFUnlock(&loopsLock);
    // 如果 loop 为空
    if (!loop)
    {
        // 创建一个新的 loop
        CFRunLoopRef newLoop = __CFRunLoopCreate(t);
        // 对 loops 字典加锁
        __CFLock(&loopsLock);
        // 从 loops 字典中获取 loop
        loop = (CFRunLoopRef)CFDictionaryGetValue(__CFRunLoops, pthreadPointer(t));
        if (!loop)
        {
            // 如果在字典中没有找到 loop，把新的 loop 存入 loops 字典中
            CFDictionarySetValue(__CFRunLoops, pthreadPointer(t), newLoop);
            loop = newLoop;
        }
        // don't release run loops inside the loopsLock, because CFRunLoopDeallocate may end up taking it
        // 不要在 字典锁作用域里面释放 runloop 对象，因为 CFRunLoopDeallocate 会释放掉 runloop
        __CFUnlock(&loopsLock);
        // 释放 newLoop
        CFRelease(newLoop);
    }
    // 判断 t 是否是当前线程
    if (pthread_equal(t, pthread_self()))
    {
        // 设置 thread specific data，简称 TSD，这也是一个字典
        // 以 __CFTSDKeyRunLoop 为 key, loop 对象为 value 存入 TSD 中
        /**
         // Set some thread specific data in a pre-assigned slot. Don't pick a random value.
         Make sure you're using a slot that is unique.
         Pass in a destructor to free this data, or NULL if none is needed. Unlike pthread TSD, the destructor is per-thread.
         // 和 pthread 的 TSD 不一样的时，第三个参数 析构函数指针是每个线程都拥有的
         CF_EXPORT void *_CFSetTSD(uint32_t slot, void *newVal, void (*destructor)(void *));
         */
        _CFSetTSD(__CFTSDKeyRunLoop, (void *)loop, NULL);
        if (0 == _CFGetTSD(__CFTSDKeyRunLoopCntr))
        {
            _CFSetTSD(__CFTSDKeyRunLoopCntr, (void *)(PTHREAD_DESTRUCTOR_ITERATIONS - 1), (void (*)(void *))__CFFinalizeRunLoop);
        }
    }
    return loop;
}
```

通过上面的源码，我们明确了在底层有一个字典的数据结构。
这个字典存储方式的是以 `pthread_t` 线程为 key，CFRunLoopRef 为 value。
我们可以有以下的结论：

*  每条线程都有唯一的一个与之对应的RunLoop对象*  RunLoop 保存在一个全局的 Dictionary 里，线程作为 key，RunLoop 作为 value*  线程刚创建时并没有 RunLoop 对象，RunLoop 会在第一次获取它时创建
*  RunLoop 会在线程结束时销毁
*  主线程的 RunLoop 已经自动获取（创建），子线程默认没有开启RunLoop
# Runloop 底层

## RunLoop 启动方式

```c
void CFRunLoopRun(void)
{ /* DOES CALLOUT */
    int32_t result;
    do
    {
        result = CFRunLoopRunSpecific(CFRunLoopGetCurrent(), kCFRunLoopDefaultMode, 1.0e10, false);
        CHECK_FOR_FORK();
    } while (kCFRunLoopRunStopped != result && kCFRunLoopRunFinished != result);
}

SInt32 CFRunLoopRunInMode(CFStringRef modeName, CFTimeInterval seconds, Boolean returnAfterSourceHandled)
{ /* DOES CALLOUT */
    CHECK_FOR_FORK();
    return CFRunLoopRunSpecific(CFRunLoopGetCurrent(), modeName, seconds, returnAfterSourceHandled);
}
```
由上面的源码可知：

* 默认启动方式，底层是通过 `CFRunLoopRun` 来实现，超时时间传入的是 `1.0e10`，这是一个很大的数字，所以可以理解为不超时。
* 自定义启动方式，可以配置超时时间以及 mode 等其它参数。

## RunLoop 核心逻辑

我们接着进入 `CFRunLoopRunSpecific`:

```c
SInt32 CFRunLoopRunSpecific(CFRunLoopRef rl, CFStringRef modeName, CFTimeInterval seconds, Boolean returnAfterSourceHandled)
{ /* DOES CALLOUT */
    CHECK_FOR_FORK();
    // 如果 runloop 正在回收中，直接返回 kCFRunLoopRunFinished ，表示 runloop 已经完成
    if (__CFRunLoopIsDeallocating(rl))
        return kCFRunLoopRunFinished;
    // 对 runloop 加锁
    __CFRunLoopLock(rl);
    // 从 runloop 中查找给定的 mode
    CFRunLoopModeRef currentMode = __CFRunLoopFindMode(rl, modeName, false);
    // 如果找不到 mode，且当前 runloop 的 currentMode 也为空，进入 if 逻辑
    // __CFRunLoopModeIsEmpty 函数结果为空的话，说明 runloop 已经处理完所有任务
    if (NULL == currentMode || __CFRunLoopModeIsEmpty(rl, currentMode, rl->_currentMode))
    {
        Boolean did = false;
        // 如果 currentMode 不为空
        if (currentMode)
            // 对 currentMode 解锁
            __CFRunLoopModeUnlock(currentMode);
        // 对 runloop 解锁
        __CFRunLoopUnlock(rl);
        return did ? kCFRunLoopRunHandledSource : kCFRunLoopRunFinished;
    }
    // 暂时取出 runloop 的 per_run_data
    volatile _per_run_data *previousPerRun = __CFRunLoopPushPerRunData(rl);
    // 取出 runloop 的当前 mode
    CFRunLoopModeRef previousMode = rl->_currentMode;
    // 将查找到的 mode 赋值到 runloop 的 _curentMode，也就是说在这 runloop 完成了 mode 的切换
    rl->_currentMode = currentMode;
    // 初始化返回结果 result
    int32_t result = kCFRunLoopRunFinished;

    // 如果注册了 observer 监听 kCFRunLoopEntry 状态(即将进入 loop)，则通知 observer
    if (currentMode->_observerMask & kCFRunLoopEntry)
        __CFRunLoopDoObservers(rl, currentMode, kCFRunLoopEntry);
    // runloop 核心函数 __CFRunLoopRun
    result = __CFRunLoopRun(rl, currentMode, seconds, returnAfterSourceHandled, previousMode);
    // 如果注册了 observer 监听 kCFRunLoopExit 状态(即将推出 loop)，则通知 observer
    if (currentMode->_observerMask & kCFRunLoopExit)
        __CFRunLoopDoObservers(rl, currentMode, kCFRunLoopExit);
    // 对 currentMode 解锁
    __CFRunLoopModeUnlock(currentMode);
    // 还原原来的 previousPerRun
    __CFRunLoopPopPerRunData(rl, previousPerRun);
    // 还原原来的 mode
    rl->_currentMode = previousMode;
    // 对 runloop 解锁
    __CFRunLoopUnlock(rl);
    return result;
}
```

这块的代码精简一下之后:

```c
SInt32 CFRunLoopRunSpecific(CFRunLoopRef rl, CFStringRef modeName, CFTimeInterval seconds, Boolean returnAfterSourceHandled)
{ /* DOES CALLOUT */
    // 通知 Observers 进入 loop
    __CFRunLoopDoObservers(rl, currentMode, kCFRunLoopEntry);
    
    // 具体要做的事情
    result = __CFRunLoopRun(rl, currentMode, seconds, returnAfterSourceHandled, previousMode);
    
    // 通知 Observers 退出 loop
    __CFRunLoopDoObservers(rl, currentMode, kCFRunLoopExit);

    return result;
}
```

功夫不负有心人，终于来到了最核心的流程 `__CFRunLoopRun` 函数，说白了，一次运行循环就是一次 `__CFRunLoopRun` 的运行。

```c
static int32_t __CFRunLoopRun(CFRunLoopRef rl, CFRunLoopModeRef rlm, CFTimeInterval seconds, Boolean stopAfterHandle, CFRunLoopModeRef previousMode)
```

`__CFRunLoopRun` 参数有 5 个，分别表示:

* `rl`: CFRunLoopRef 对象
* `rlm`: mode 的名称
* `seconds`: 超时时间
* `stopAfterHandle`: 处理完 source 后是否直接返回
* `previousMode`: 前一次运行循环的 mode

> 下面提供 `__CFRunLoopRun` 的完整代码和精简代码，读者可根据自身需要选择阅读。

## __CFRunLoopRun 完整代码

```c
static int32_t __CFRunLoopRun(CFRunLoopRef rl, CFRunLoopModeRef rlm, CFTimeInterval seconds, Boolean stopAfterHandle, CFRunLoopModeRef previousMode)
{
    // 获取开始时间
    uint64_t startTSR = mach_absolute_time();

    if (__CFRunLoopIsStopped(rl))
    {
        // Runloop 处于停止状态，修改runloop底层数据结构 停止状态 flag
        __CFRunLoopUnsetStopped(rl);
        // 返回 kCFRunLoopRunStopped 状态
        return kCFRunLoopRunStopped;
    }
    else if (rlm->_stopped)
    {
        // Mode 处于停止状态，修改 mode 的成员变量 _stopped 为 false
        rlm->_stopped = false;
        // 返回 kCFRunLoopRunStopped 状态
        return kCFRunLoopRunStopped;
    }

    // 判断是否有主队列的 Mach Port
    mach_port_name_t dispatchPort = MACH_PORT_NULL;
    Boolean libdispatchQSafe = pthread_main_np() && ((HANDLE_DISPATCH_ON_BASE_INVOCATION_ONLY && NULL == previousMode) || (!HANDLE_DISPATCH_ON_BASE_INVOCATION_ONLY && 0 == _CFGetTSD(__CFTSDKeyIsInGCDMainQ)));
    if (libdispatchQSafe && (CFRunLoopGetMain() == rl) && CFSetContainsValue(rl->_commonModes, rlm->_name))
        dispatchPort = _dispatch_get_main_queue_port_4CF();

#if USE_DISPATCH_SOURCE_FOR_TIMERS
    // 从 mode 的 成员变量 _queue 中取出对应的 Mach Port
    mach_port_name_t modeQueuePort = MACH_PORT_NULL;
    if (rlm->_queue)
    {
        modeQueuePort = _dispatch_runloop_root_queue_get_port_4CF(rlm->_queue);
        if (!modeQueuePort)
        {
            CRASH("Unable to get port for run loop mode queue (%d)", -1);
        }
    }
#endif

    // 声明一个空的 GCD 定时器
    dispatch_source_t timeout_timer = NULL;
    // 初始化一个 「超时上下文」 结构体指针对象
    struct __timeout_context *timeout_context = (struct __timeout_context *)malloc(sizeof(*timeout_context));
    // 如果 runloop 超时时间小于等于0
    if (seconds <= 0.0)
    { // instant timeout
        // 将 seconds 置为 0s
        seconds = 0.0;
        // 将 「超时上下文」的 termTSR 属性设置为 0，0ULL 后面的 ULL 表示 Unsigned Long Long 即无符号 long long 整型
        timeout_context->termTSR = 0ULL;
    }
    else if (seconds <= TIMER_INTERVAL_LIMIT)
    {
        // 如果 runloop 超时时间小于等于 TIMER_INTERVAL_LIMIT 常量
        // 如果主线程存在，则获取主线程的主队列
        // 如果主线程不存在，则获取全局队列
        dispatch_queue_t queue = pthread_main_np() ? __CFDispatchQueueGetGenericMatchingMain() : __CFDispatchQueueGetGenericBackground();
        // 将队列传入并初始化一个 GCD 对象，对象类型为定时器
        timeout_timer = dispatch_source_create(DISPATCH_SOURCE_TYPE_TIMER, 0, 0, queue);
        // 对 GCD 对象进行 retain
        dispatch_retain(timeout_timer);
        // 将 GCD 对象赋值于 「超时上下文」 结构体的 ds 属性
        timeout_context->ds = timeout_timer;
        // 对 runloop 对象进行 retain，然后赋值于「超时上下文」 结构体的 rl 属性
        timeout_context->rl = (CFRunLoopRef)CFRetain(rl);
        // 将本次运行循环的开始时间加上要运行的时间得到runloop的生命周期时间
        // 然后赋值于「超时上下文」 结构体的 termTSR 属性
        timeout_context->termTSR = startTSR + __CFTimeIntervalToTSR(seconds);
        
        // 设置 GCD 对象的上下文为 自定义的 「超时上下文」 结构体
        dispatch_set_context(timeout_timer, timeout_context); // source gets ownership of context
        
        // 设置 GCD 对象的回调 可以理解为要执行的任务
        /**
         static void __CFRunLoopTimeout(void *arg)
         {
             struct __timeout_context *context = (struct __timeout_context *)arg;
             context->termTSR = 0ULL;
             CFRUNLOOP_WAKEUP_FOR_TIMEOUT();
             CFRunLoopWakeUp(context->rl);
             // The interval is DISPATCH_TIME_FOREVER, so this won't fire again
         }
         __CFRunLoopTimeout 函数内部
         1.将 「超时上下文」 结构体的 termTSR 属性赋值为0
         2.CFRUNLOOP_WAKEUP_FOR_TIMEOUT() 宏内部是一个 do-while(0) 的无用操作，可以忽略
         3.然后通过 CFRunLoopWakeUp 函数来唤醒 runloop
         */
        dispatch_source_set_event_handler_f(timeout_timer, __CFRunLoopTimeout);
        // 设置 GCD 对象的取消回调 可以理解为取消任务时的回调
        dispatch_source_set_cancel_handler_f(timeout_timer, __CFRunLoopTimeoutCancel);
        
        // 将开始时间加上给定的 runloop 过期时间，因为此时结果为 秒，需要转换为 纳秒，所以再乘以 10 ^ 9
        uint64_t ns_at = (uint64_t)((__CFTSRToTimeInterval(startTSR) + seconds) * 1000000000ULL);
        
        // 设置 GCD 定时器
        // 第二个参数表示定时器开始时间 也就是说，如果是 CFRunLoopRun 进来的话
        // 因为传入的值是 1.0e10，所以这个定时器永远不会触发，也就意味着 runloop 永远不会超时
        // 第三个参数表示 定时器 纳秒级的 间隔，这里传入的是 DISPATCH_TIME_FOREVER，说明定时器只会触发一次，不会触发多次
        // 第四个参数表示 定时器 纳秒级 的容错时间
        dispatch_source_set_timer(timeout_timer, dispatch_time(1, ns_at), DISPATCH_TIME_FOREVER, 1000ULL);
        // 开启 GCD 定时器
        dispatch_resume(timeout_timer);
    }
    else
    { // infinite timeout
        // 设置超时时间为 9999999999
        // 然后设置 自定义的 「超时上下文」 结构体的 termTSR 属性值为 UNIT64_MAX
        seconds = 9999999999.0;
        timeout_context->termTSR = UINT64_MAX;
    }

    // 初始化一个布尔值 上一次是否通过 dispatch port
    Boolean didDispatchPortLastTime = true;
    int32_t retVal = 0;
    do
    {
#if DEPLOYMENT_TARGET_MACOSX || DEPLOYMENT_TARGET_EMBEDDED || DEPLOYMENT_TARGET_EMBEDDED_MINI
        voucher_mach_msg_state_t voucherState = VOUCHER_MACH_MSG_STATE_UNCHANGED;
        voucher_t voucherCopy = NULL;
#endif
        // 初始化一个 无符号 int8 整型类型的 消息缓冲区，缓冲区大小为 3KB
        uint8_t msg_buffer[3 * 1024];
#if DEPLOYMENT_TARGET_MACOSX || DEPLOYMENT_TARGET_EMBEDDED || DEPLOYMENT_TARGET_EMBEDDED_MINI
        // 初始化一个空的 mach_msg_header_t 类型的 msg 结构体指针变量
        mach_msg_header_t *msg = NULL;
        // 初始化一个值为 MACH_PORT_NULL 的 port
        mach_port_t livePort = MACH_PORT_NULL;

        // 将 mode 的 _portSet 取出
        __CFPortSet waitSet = rlm->_portSet;

        // 设置 runloop 底层数据结构的 ignoreWakeUps 值为 0
        __CFRunLoopUnsetIgnoreWakeUps(rl);

        // 判断如果 mode 的 observer 的 mask 与上 kCFRunLoopBeforeTimers 枚举值是否为1
        // 如果成功，通知 Observers 即将处理 Timers
        if (rlm->_observerMask & kCFRunLoopBeforeTimers)
            __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeTimers);
        
        // 判断如果 mode 的 observer 的 mask 与上 kCFRunLoopBeforeSources 枚举值是否为1
        // 如果成功，通知 Observers 即将处理 Sources
        if (rlm->_observerMask & kCFRunLoopBeforeSources)
            __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeSources);

        // 处理 Blocks，传入 runloop 和 mode
        __CFRunLoopDoBlocks(rl, rlm);

        // 处理 Source0，传入 runloop 和 mode，以及 stopAfterHandle
        Boolean sourceHandledThisLoop = __CFRunLoopDoSources0(rl, rlm, stopAfterHandle);
        // 如果处理 Sources0 成功，再次处理 Blocks
        if (sourceHandledThisLoop)
        {
            __CFRunLoopDoBlocks(rl, rlm);
        }
        
        // 如果本次 loop 执行 source0 成功 或 超时时间为0，设置 poll 为 true
        Boolean poll = sourceHandledThisLoop || (0ULL == timeout_context->termTSR);

        // 如果主队列 port 不为空且上一次 loop 没有主队列 port
        if (MACH_PORT_NULL != dispatchPort && !didDispatchPortLastTime)
        {
#if DEPLOYMENT_TARGET_MACOSX || DEPLOYMENT_TARGET_EMBEDDED || DEPLOYMENT_TARGET_EMBEDDED_MINI
            // 初始化一个 mach_msg_header_t 结构的 msg
            msg = (mach_msg_header_t *)msg_buffer;
            // 判断主队列的port上是否有消息，如果有，进入 handle_msg 流程
            if (__CFRunLoopServiceMachPort(dispatchPort, &msg, sizeof(msg_buffer), &livePort, 0, &voucherState, NULL))
            {
                // 说明主队列的port上有消息过来，需要唤醒 runloop
                goto handle_msg;
            }
#endif
        }
        
        // 将上一次 loop 主队列 值为空
        didDispatchPortLastTime = false;

        // 说明没有从 port 拉取消息，并且 mode 的 _observerMask 与上 kCFRunLoopBeforeWaiting 为真，
        // 则通知 Observers 即将休眠
        if (!poll && (rlm->_observerMask & kCFRunLoopBeforeWaiting))
            __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeWaiting);
        
        
        /**
         CF_INLINE void __CFRunLoopSetSleeping(CFRunLoopRef rl)
         {
             __CFBitfieldSetValue(((CFRuntimeBase *)rl)->_cfinfo[CF_INFO_BITS], 1, 1, 1);
         }
         */
        // 设置 runloop sleeping 的flag
        __CFRunLoopSetSleeping(rl);
        // do not do any user callouts after this point (after notifying of sleeping)
        // 在 runloop 已经睡眠之后，不要做任何用户调用

        // Must push the local-to-this-activation ports in on every loop
        // iteration, as this mode could be run re-entrantly and we don't
        // want these ports to get serviced.
        
        /**
         //必须在每个循环中都将 local-to-this-activation 端口推入
         //迭代，因为此模式可以重新运行，我们不
         //希望这些端口可以被服务。
         */
        // 将主队列 port 加入 waitSet 中
        __CFPortSetInsert(dispatchPort, waitSet);

        // 解锁 mode
        __CFRunLoopModeUnlock(rlm);
        // 解锁 runloop
        __CFRunLoopUnlock(rl);

        // 如果 poll 为真，sleepStart 为 0，否则为当前时间
        CFAbsoluteTime sleepStart = poll ? 0.0 : CFAbsoluteTimeGetCurrent();

#if DEPLOYMENT_TARGET_MACOSX || DEPLOYMENT_TARGET_EMBEDDED || DEPLOYMENT_TARGET_EMBEDDED_MINI
#if USE_DISPATCH_SOURCE_FOR_TIMERS
        // 这个 do - while 循环主要针对于 GCD 定时器
        do
        {
            if (kCFUseCollectableAllocator)
            {
                // objc_clear_stack(0);
                // <rdar://problem/16393959>
                // 将 msg_buffer 缓冲区 置为0
                memset(msg_buffer, 0, sizeof(msg_buffer));
            }
            msg = (mach_msg_header_t *)msg_buffer;
            // 判断 waitSet 的 port 是否有消息
            __CFRunLoopServiceMachPort(waitSet, &msg, sizeof(msg_buffer), &livePort, poll ? 0 : TIMEOUT_INFINITY, &voucherState, &voucherCopy);

            // 判断 modeQueuePort 这个 port 不为空，且 livePort 为 modeQueuePort
            if (modeQueuePort != MACH_PORT_NULL && livePort == modeQueuePort)
            {
                // Drain the internal queue. If one of the callout blocks sets the timerFired flag, break out and service the timer.
                /**
                 清空内部队列。 如果其中一个标注块设置了 timerFired 标志，请中断并为计时器提供服务。
                 */
                while (_dispatch_runloop_root_queue_perform_4CF(rlm->_queue))
                    ;
                // 如果 mode 的 _timerFired 为 真，则置为 false，并退出 do-while 循环
                if (rlm->_timerFired)
                {
                    // Leave livePort as the queue port, and service timers below
                    rlm->_timerFired = false;
                    break;
                }
                else
                {
                    if (msg && msg != (mach_msg_header_t *)msg_buffer)
                        free(msg);
                }
            }
            else
            {
                // Go ahead and leave the inner loop.
                break;
            }
        } while (1);
#else
        if (kCFUseCollectableAllocator)
        {
            // objc_clear_stack(0);
            // <rdar://problem/16393959>
            // 将 msg_buffer 缓冲区 置为0
            memset(msg_buffer, 0, sizeof(msg_buffer));
        }
        msg = (mach_msg_header_t *)msg_buffer;
        // 判断 waitSet 的 port 是否有消息
        __CFRunLoopServiceMachPort(waitSet, &msg, sizeof(msg_buffer), &livePort, poll ? 0 : TIMEOUT_INFINITY, &voucherState, &voucherCopy);
#endif

#endif
        // 对 runloop 加锁
        __CFRunLoopLock(rl);
        // 对 mode 加锁
        __CFRunLoopModeLock(rlm);

        // 如果 poll 为真，说明主队了唤醒了 runloop，反之则获得 runloop 的睡眠时间
        rl->_sleepTime += (poll ? 0.0 : (CFAbsoluteTimeGetCurrent() - sleepStart));

        // Must remove the local-to-this-activation ports in on every loop
        // iteration, as this mode could be run re-entrantly and we don't
        // want these ports to get serviced. Also, we don't want them left
        // in there if this function returns.
        
        
        /**
         
         CF_INLINE kern_return_t __CFPortSetRemove(__CFPort port, __CFPortSet portSet)
         {
             if (MACH_PORT_NULL == port)
             {
                 return -1;
             }
             return mach_port_extract_member(mach_task_self(), port, portSet);
         }
         
         */
        // 将 dispatchPort 从 waitSet 中移除，因为每次 runloop 迭代中 dispatchPort 都会被加入 waitSet
        __CFPortSetRemove(dispatchPort, waitSet);
        
        /**
         CF_INLINE void __CFRunLoopSetIgnoreWakeUps(CFRunLoopRef rl)
         {
             rl->_perRunData->ignoreWakeUps = 0x57414B45; // 'WAKE'
         }
         唤醒 runloop
         */
        __CFRunLoopSetIgnoreWakeUps(rl);

        // user callouts now OK again
        // 设置 runloop 的 sleeping flag 为 0
        __CFRunLoopUnsetSleeping(rl);
        
        // 通知 Observers 结束休眠
        if (!poll && (rlm->_observerMask & kCFRunLoopAfterWaiting))
            __CFRunLoopDoObservers(rl, rlm, kCFRunLoopAfterWaiting);

    handle_msg:;
        /**
         CF_INLINE void __CFRunLoopSetIgnoreWakeUps(CFRunLoopRef rl)
         {
             rl->_perRunData->ignoreWakeUps = 0x57414B45; // 'WAKE'
         }
         表示收到了 port 消息，需要处理
         */
        __CFRunLoopSetIgnoreWakeUps(rl);

        // 如果唤醒的端口为空，啥事也不干
        if (MACH_PORT_NULL == livePort)
        {
            CFRUNLOOP_WAKEUP_FOR_NOTHING();
            // handle nothing
        }
        // 如果唤醒端口是 runloop 的 _wakeUpPort，则执行 CFRUNLOOP_WAKEUP_FOR_WAKEUP() 函数
        // CFRUNLOOP_WAKEUP_FOR_WAKEUP() 函数其实啥也没干
        else if (livePort == rl->_wakeUpPort)
        {
            CFRUNLOOP_WAKEUP_FOR_WAKEUP();
            // do nothing on Mac OS
        }
#if USE_DISPATCH_SOURCE_FOR_TIMERS
        // 如果唤醒端口是 modeQueuePort，说明是 定时器唤醒了 runloop
        else if (modeQueuePort != MACH_PORT_NULL && livePort == modeQueuePort)
        {
            // 调用 CFRUNLOOP_WAKEUP_FOR_TIMER 函数，
            CFRUNLOOP_WAKEUP_FOR_TIMER();
            // 执行 timer
            if (!__CFRunLoopDoTimers(rl, rlm, mach_absolute_time()))
            {
                // Re-arm the next timer, because we apparently fired early
                __CFArmNextTimerInMode(rlm, rl);
            }
        }
#endif
#if USE_MK_TIMER_TOO
        // 如果唤醒端口是 mode 的 _timerPort
        else if (rlm->_timerPort != MACH_PORT_NULL && livePort == rlm->_timerPort)
        {
            CFRUNLOOP_WAKEUP_FOR_TIMER();
            // On Windows, we have observed an issue where the timer port is set before the time which we requested it to be set. For example, we set the fire time to be TSR 167646765860, but it is actually observed firing at TSR 167646764145, which is 1715 ticks early. The result is that, when __CFRunLoopDoTimers checks to see if any of the run loop timers should be firing, it appears to be 'too early' for the next timer, and no timers are handled.
            // In this case, the timer port has been automatically reset (since it was returned from MsgWaitForMultipleObjectsEx), and if we do not re-arm it, then no timers will ever be serviced again unless something adjusts the timer list (e.g. adding or removing timers). The fix for the issue is to reset the timer here if CFRunLoopDoTimers did not handle a timer itself. 9308754
            // 执行 timer
            if (!__CFRunLoopDoTimers(rl, rlm, mach_absolute_time()))
            {
                // Re-arm the next timer
                __CFArmNextTimerInMode(rlm, rl);
            }
        }
#endif
        // 如果唤醒端口是 dispatchPort，说明是主队列唤醒了 Runloop
        else if (livePort == dispatchPort)
        {
            // 执行 CFRUNLOOP_WAKEUP_FOR_DISPATCH() 函数
            CFRUNLOOP_WAKEUP_FOR_DISPATCH();
            // 解锁 mode
            __CFRunLoopModeUnlock(rlm);
            // 解锁 runloop
            __CFRunLoopUnlock(rl);
            // 设置 TSD，以 __CFTSDKeyIsInGCDMainQ 为 key， 6 为值
            _CFSetTSD(__CFTSDKeyIsInGCDMainQ, (void *)6, NULL);
            // 执行主队列任务
            __CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__(msg);
            // 设置 TSD，以 __CFTSDKeyIsInGCDMainQ 为 key， 0 为值
            _CFSetTSD(__CFTSDKeyIsInGCDMainQ, (void *)0, NULL);
            // 对 runloop 加锁
            __CFRunLoopLock(rl);
            // 对 mode 加锁
            __CFRunLoopModeLock(rlm);
            // 标记本次 loop 处理了主队列
            sourceHandledThisLoop = true;
            didDispatchPortLastTime = true;
        }
        else
        {
            // 来到这，说明是被 source1 唤醒
            CFRUNLOOP_WAKEUP_FOR_SOURCE();

            // If we received a voucher from this mach_msg, then put a copy of the new voucher into TSD. CFMachPortBoost will look in the TSD for the voucher. By using the value in the TSD we tie the CFMachPortBoost to this received mach_msg explicitly without a chance for anything in between the two pieces of code to set the voucher again.
            voucher_t previousVoucher = _CFSetTSD(__CFTSDKeyMachMessageHasVoucher, (void *)voucherCopy, os_release);

            // Despite the name, this works for windows handles as well
            // 从端口中取出 source1
            CFRunLoopSourceRef rls = __CFRunLoopModeFindSourceForMachPort(rl, rlm, livePort);
            if (rls)
            {
#if DEPLOYMENT_TARGET_MACOSX || DEPLOYMENT_TARGET_EMBEDDED || DEPLOYMENT_TARGET_EMBEDDED_MINI
                mach_msg_header_t *reply = NULL;
                // 执行 source1 z任务
                sourceHandledThisLoop = __CFRunLoopDoSource1(rl, rlm, rls, msg, msg->msgh_size, &reply) || sourceHandledThisLoop;
                if (NULL != reply)
                {
                    (void)mach_msg(reply, MACH_SEND_MSG, reply->msgh_size, 0, MACH_PORT_NULL, 0, MACH_PORT_NULL);
                    CFAllocatorDeallocate(kCFAllocatorSystemDefault, reply);
                }
            }

            // Restore the previous voucher
            _CFSetTSD(__CFTSDKeyMachMessageHasVoucher, previousVoucher, os_release);
        }
#if DEPLOYMENT_TARGET_MACOSX || DEPLOYMENT_TARGET_EMBEDDED || DEPLOYMENT_TARGET_EMBEDDED_MINI
        if (msg && msg != (mach_msg_header_t *)msg_buffer)
            free(msg);
#endif
        // 处理 Blocks，传入 runloop 和 mode
        __CFRunLoopDoBlocks(rl, rlm);

        /**
         CFRunLoopRunInMode() 函数返回的原因，有四种
         // Reasons for CFRunLoopRunInMode() to Return
         enum {
             kCFRunLoopRunFinished = 1, // 完成了 loop
             kCFRunLoopRunStopped = 2,  // loop 被终止
             kCFRunLoopRunTimedOut = 3, // loop 超时
             kCFRunLoopRunHandledSource = 4 // loop 执行了 主队列任务
         };
         */
        // 如果执行了主队列任务并且 stopAfterHandle 为真，则退出 do-while 循环
        if (sourceHandledThisLoop && stopAfterHandle)
        {
            retVal = kCFRunLoopRunHandledSource;
        }
        // 如果超时，则退出 do-while 循环
        else if (timeout_context->termTSR < mach_absolute_time())
        {
            retVal = kCFRunLoopRunTimedOut;
        }
        // 如果 runloop 已经停止，则退出 do-while 循环
        else if (__CFRunLoopIsStopped(rl))
        {
            __CFRunLoopUnsetStopped(rl);
            retVal = kCFRunLoopRunStopped;
        }
        // 如果 mode 的 _stopped 属性为真，则退出 do-while 循环
        else if (rlm->_stopped)
        {
            rlm->_stopped = false;
            retVal = kCFRunLoopRunStopped;
        }
        // 如果 mode 此时为空，则退出 do-while 循环
        else if (__CFRunLoopModeIsEmpty(rl, rlm, previousMode))
        {
            retVal = kCFRunLoopRunFinished;
        }

#if DEPLOYMENT_TARGET_MACOSX || DEPLOYMENT_TARGET_EMBEDDED || DEPLOYMENT_TARGET_EMBEDDED_MINI
        voucher_mach_msg_revert(voucherState);
        os_release(voucherCopy);
#endif

    } while (0 == retVal);

    // 如果开启了 GCD 定时器
    if (timeout_timer)
    {
        // 停掉定时器并释放资源
        dispatch_source_cancel(timeout_timer);
        dispatch_release(timeout_timer);
    }
    else
    {
        // 释放 超时上下文 结构体指针
        free(timeout_context);
    }

    return retVal;
}
```

## __CFRunLoopRun 精简代码

```c
static int32_t __CFRunLoopRun(CFRunLoopRef rl, CFRunLoopModeRef rlm, CFTimeInterval seconds, Boolean stopAfterHandle, CFRunLoopModeRef previousMode)
{
    int32_t retVal = 0;
    do
    {
        // 通知 Observers 即将处理 Timers
        __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeTimers);
        
        // 通知 Observers 即将处理 Sources
        __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeSources);

        // 处理 Blocks
        __CFRunLoopDoBlocks(rl, rlm);

        // 处理 Source0
        if (__CFRunLoopDoSources0(rl, rlm, stopAfterHandle))
        {
            // 处理 Blocks
            __CFRunLoopDoBlocks(rl, rlm);
        }

        Boolean poll = sourceHandledThisLoop || (0ULL == timeout_context->termTSR);

        // 判断有无 Source1
        if (__CFRunLoopServiceMachPort(dispatchPort, &msg, sizeof(msg_buffer), &livePort, 0, &voucherState, NULL))
        {
            // 如果有 Source1，就跳转到 handle_msg
            goto handle_msg;
        }

            
        didDispatchPortLastTime = false;

        // 通知 Observers 即将休眠
        __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeWaiting);
            
        __CFRunLoopSetSleeping(rl);

        CFAbsoluteTime sleepStart = poll ? 0.0 : CFAbsoluteTimeGetCurrent();
        
        do
        {
            if (kCFUseCollectableAllocator)
            {
                // objc_clear_stack(0);
                // <rdar://problem/16393959>
                memset(msg_buffer, 0, sizeof(msg_buffer));
            }
            msg = (mach_msg_header_t *)msg_buffer;

            // 等待别的消息来唤醒当前线程
            __CFRunLoopServiceMachPort(waitSet, &msg, sizeof(msg_buffer), &livePort, poll ? 0 : TIMEOUT_INFINITY, &voucherState, &voucherCopy);

            if (modeQueuePort != MACH_PORT_NULL && livePort == modeQueuePort)
            {
                // Drain the internal queue. If one of the callout blocks sets the timerFired flag, break out and service the timer.
                while (_dispatch_runloop_root_queue_perform_4CF(rlm->_queue))
                    ;
                if (rlm->_timerFired)
                {
                    // Leave livePort as the queue port, and service timers below
                    rlm->_timerFired = false;
                    break;
                }
                else
                {
                    if (msg && msg != (mach_msg_header_t *)msg_buffer)
                        free(msg);
                }
            }
            else
            {
                // Go ahead and leave the inner loop.
                break;
            }
        } while (1);

        // user callouts now OK again
        __CFRunLoopUnsetSleeping(rl);
        
        // 通知 Observers 结束休眠
        __CFRunLoopDoObservers(rl, rlm, kCFRunLoopAfterWaiting);

    handle_msg:
        if (被 timer 唤醒)
        {
            // 处理 timers
            __CFRunLoopDoTimers(rl, rlm, mach_absolute_time());
        }

        else if (被 GCD 唤醒)
        {
            // 处理 GCD 主队列
            __CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__(msg);
        }
        else
        {
            // 被 Source1 唤醒
            __CFRunLoopDoSource1(rl, rlm, rls, msg, msg->msgh_size, &reply) || sourceHandledThisLoop;
        }

        // 处理 Blocks
        __CFRunLoopDoBlocks(rl, rlm);

        if (sourceHandledThisLoop && stopAfterHandle)
        {
            retVal = kCFRunLoopRunHandledSource;
        }
        else if (timeout_context->termTSR < mach_absolute_time())
        {
            retVal = kCFRunLoopRunTimedOut;
        }
        else if (__CFRunLoopIsStopped(rl))
        {
            __CFRunLoopUnsetStopped(rl);
            retVal = kCFRunLoopRunStopped;
        }
        else if (rlm->_stopped)
        {
            rlm->_stopped = false;
            retVal = kCFRunLoopRunStopped;
        }
        else if (__CFRunLoopModeIsEmpty(rl, rlm, previousMode))
        {
            retVal = kCFRunLoopRunFinished;
        }

        voucher_mach_msg_revert(voucherState);
        os_release(voucherCopy);

    } while (0 == retVal);

    return retVal;
}
```

## RunLoop 执行的任务

通过 `__CFRunLoopRun`，我们知道了 `RunLoop` 执行的任务有以下几个:

* `__CFRunLoopDoObservers`
* `__CFRunLoopDoBlocks`
* `__CFRunLoopDoSources0`
* `__CFRunLoopDoSource1`
* `__CFRunLoopDoTimers`
* `__CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__`

下面，我们将一个一个进行分析:

### __CFRunLoopDoObservers

```c
/* rl is locked, rlm is locked on entrance and exit */
static void __CFRunLoopDoObservers() __attribute__((noinline));
static void __CFRunLoopDoObservers(CFRunLoopRef rl, CFRunLoopModeRef rlm, CFRunLoopActivity activity)
{ /* DOES CALLOUT */
    CHECK_FOR_FORK();

    // 判断传入的 mode 对应的 observer 数量是否为 0
    CFIndex cnt = rlm->_observers ? CFArrayGetCount(rlm->_observers) : 0;
    if (cnt < 1)
        return;

    /* Fire the observers */
    
    // #define STACK_BUFFER_DECL(T, N, C) T N[C]
    // 这里可以分为两种情况
    // 如果 mode 中的 observer 数量小于等于 1024 个，那么相当于 CFRunLoopObserverRef buffer[cnt]
    // 如果 mode 中的 observer 数量大于 1024 个，那么相当于 CFRunLoopObserverRef buffer[1]
    // 所以这里通过 observer 数量来分配不同大小的缓存区
    STACK_BUFFER_DECL(CFRunLoopObserverRef, buffer, (cnt <= 1024) ? cnt : 1);
    
    
    // 判断如果 observer 数量小于 1024 个，则把上一步得到的 buffer 缓存区赋值给 collectedObservers
    // 如果 observer 数量大于 1024 个，则调用 malloc 函数来分配空间, 最终空间大小为 =  observer数目 * CFRunLoopObserverRef 类型占用内存大小
    // 最终获得了一个 类型为 CFRunLoopObserverRef 的数组指针变量 collectedObservers
    CFRunLoopObserverRef *collectedObservers = (cnt <= 1024) ? buffer : (CFRunLoopObserverRef *)malloc(cnt * sizeof(CFRunLoopObserverRef));
    
    CFIndex obs_cnt = 0;
    // 循环遍历传入的 runloop mode 中所有的 observers
    for (CFIndex idx = 0; idx < cnt; idx++)
    {
        // 从 observers 集合中取出一个 observer
        CFRunLoopObserverRef rlo = (CFRunLoopObserverRef)CFArrayGetValueAtIndex(rlm->_observers, idx);
        
        // 满足下列三个条件后进入 if 内部的逻辑
        // 1.判断 observer 当前的状态是否与传入的状态匹配，这里是通过 与 运算来完成，如果相等，结果应为 1，所以这里判断是不等于 0
        // 2.判断 observer 是否是有效的
        /**
         // Bit 3 in the base reserved bits is used for invalid state in run loop objects

         CF_INLINE Boolean __CFIsValid(const void *cf)
         {
             return (Boolean)__CFBitfieldGetValue(((const CFRuntimeBase *)cf)->_cfinfo[CF_INFO_BITS], 3, 3);
         }

         CF_INLINE void __CFSetValid(void *cf)
         {
             __CFBitfieldSetValue(((CFRuntimeBase *)cf)->_cfinfo[CF_INFO_BITS], 3, 3, 1);
         }

         CF_INLINE void __CFUnsetValid(void *cf)
         {
             __CFBitfieldSetValue(((CFRuntimeBase *)cf)->_cfinfo[CF_INFO_BITS], 3, 3, 0);
         }
         
         */
        // 3.判断 observer 是否已经 fire 了
        /**
         Bit 0 of the base reserved bits is used for firing state
         Bit 1 of the base reserved bits is used for repeats state
         CF_INLINE Boolean __CFRunLoopObserverIsFiring(CFRunLoopObserverRef rlo)
         {
             return (Boolean)__CFBitfieldGetValue(((const CFRuntimeBase *)rlo)->_cfinfo[CF_INFO_BITS], 0, 0);
         }
         */
        
        /**
         探索第二步和第三步操作之后，可以发现 CFRuntimeBase 结构中的
         Bit 0 是用来表示 启动状态
         Bit 1 是用来表示 重复状态
         Bit 3 是用来表示 runloop 对象的可用状态
         */
        
        if (0 != (rlo->_activities & activity) && __CFIsValid(rlo) && !__CFRunLoopObserverIsFiring(rlo))
        {
            // 对 observer 进行 retain，然后存入 collectedObservers 集合中
            // CoreFoundation 框架中 需要手动MRC，也就是对接收到的对象需要调用 CFRetain，与此同时，需要有配对的 CFRelease 操作防止内存泄漏
            collectedObservers[obs_cnt++] = (CFRunLoopObserverRef)CFRetain(rlo);
        }
    }
    // 对传入的 mode 解锁
    // 底层实现是一个互斥锁
    /**
     CF_INLINE void __CFRunLoopModeUnlock(CFRunLoopModeRef rlm)
     {
         //CFLog(6, CFSTR("__CFRunLoopModeLock unlocking %p"), rlm);
         pthread_mutex_unlock(&(rlm->_lock));
     }
     */
    __CFRunLoopModeUnlock(rlm);
    // 对传入的 runloop 解锁
    // 底层实现是一个互斥锁
    /**
     CF_INLINE void __CFRunLoopUnlock(CFRunLoopRef rl)
     {
         //    CFLog(6, CFSTR("__CFRunLoopLock unlocking %p"), rl);
         pthread_mutex_unlock(&(((CFRunLoopRef)rl)->_lock));
     }
     */
    __CFRunLoopUnlock(rl);
    // 根据上面 for 循环内部自增的变量 obs_cnt ，也就是有效的 observer 的个数进行遍历
    for (CFIndex idx = 0; idx < obs_cnt; idx++)
    {
        // 取出一个 observer
        CFRunLoopObserverRef rlo = collectedObservers[idx];
        
        // 对 observer 进行加锁
        // 底层实现是一把互斥锁
        /**
         CF_INLINE void __CFRunLoopObserverLock(CFRunLoopObserverRef rlo)
         {
             pthread_mutex_lock(&(rlo->_lock));
             //    CFLog(6, CFSTR("__CFRunLoopObserverLock locked %p"), rlo);
         }
         */
        __CFRunLoopObserverLock(rlo);
        
        // 再次判断 observer 是否已失效
        if (__CFIsValid(rlo))
        {
            // 从 observer 中取出 重复状态，然后取反，赋值给 doInvalidate 布尔变量
            // 这个布尔变量在后面用作判断，即如果 observer 的 repeat 为真，那么 doInvalidate 就为假，
            // 如果 observer 的 repeat 为假，那么 doInvalidate 就为真，
            // 就会调用 CFRunLoopObserverInvalidate 来将 observer 从所属 runloop 中剔除，然后 observer 被释放
            Boolean doInvalidate = !__CFRunLoopObserverRepeats(rlo);
            
            // 设置 observer 的 runtimebase 里面的 bit 0 值为1 ，即标记 observer 已经启动
            /**
             CF_INLINE void __CFRunLoopObserverSetFiring(CFRunLoopObserverRef rlo)
             {
                 __CFBitfieldSetValue(((CFRuntimeBase *)rlo)->_cfinfo[CF_INFO_BITS], 0, 0, 1);
             }
             */
            __CFRunLoopObserverSetFiring(rlo);
            
            // 对 observer 解锁
            __CFRunLoopObserverUnlock(rlo);
            
            // 调用 __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__ 函数，传入 observer 对应的 callout，以及 observer 自己，runloop 当前状态，以及 observer 上下文信息
            __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(rlo->_callout, rlo, activity, rlo->_context.info);
            
            if (doInvalidate)
            {
                CFRunLoopObserverInvalidate(rlo);
            }
            // 取消 observer 的 firing 状态
            __CFRunLoopObserverUnsetFiring(rlo);
        }
        else
        {
            // 说明 observer 已经失效，对 observer 解锁
            __CFRunLoopObserverUnlock(rlo);
        }
        // 因为此时 observer 要么已经触发了回调，要么由于已经失效啥也没干，所以需要释放释放掉 observer，然后开启下一次循环过程
        CFRelease(rlo);
    }
    // 对 runloop 加锁
    __CFRunLoopLock(rl);
    // 对 mode 加锁
    __CFRunLoopModeLock(rlm);

    // 判断集合如果不等于 buffer ，说明是使用的 malloc 函数初始化而来，需要调用 free 函数释放
    if (collectedObservers != buffer)
        free(collectedObservers);
}
```

可以看到最终执行回调任务的是 `__CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__`，我们来到它的内部:

```c
static void __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__() __attribute__((noinline));
static void __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(CFRunLoopObserverCallBack func, CFRunLoopObserverRef observer, CFRunLoopActivity activity, void *info)
{
    if (func)
    {
        func(observer, activity, info);
    }
    asm __volatile__(""); // thwart tail-call optimization
}
```


### __CFRunLoopDoBlocks

```c
/**
 从 runloop 的 _block_item 链表里面匹配 mode，匹配上了的就执行 block 任务
 */
static Boolean __CFRunLoopDoBlocks(CFRunLoopRef rl, CFRunLoopModeRef rlm)
{ // Call with rl and rlm locked
    /*
     因为 runloop 内部有这样的结构
     
     struct _block_item *_blocks_head;  // 链表的头指针
     struct _block_item *_blocks_tail;  // 链表的尾指针
     
     struct _block_item
     {
         struct _block_item *_next;     // 指向下一个 _block_item
         CFTypeRef _mode; // CFString or CFSet // mode 可能是字符串，也可能是集合
         void (^_block)(void);          // block
     };
     
     所以 runloop 底层是以单链表的方式存储着 _block_item 结构体指针对象
     */
    // 如果 block 链表头指针为空，说明当前 runloop 没有要执行的 block，直接返回 false
    if (!rl->_blocks_head)
        return false;
    // 如果 mode 为空或者 mode 的名称为空，也直接返回 false
    if (!rlm || !rlm->_name)
        return false;

    // 是否执行了 block
    Boolean did = false;
    // 取出 block 链表头指针
    struct _block_item *head = rl->_blocks_head;
    // 取出 block 链表尾指针
    struct _block_item *tail = rl->_blocks_tail;
    
    // 将 runloop 上的 block 链表头指针置空
    rl->_blocks_head = NULL;
    // 将 runloop 上的 block 链表尾指针置空
    rl->_blocks_tail = NULL;
    // 获取 runloop 的 commonModes
    CFSetRef commonModes = rl->_commonModes;
    // 获取传入的 mode 的名称
    CFStringRef curMode = rlm->_name;
    // 对 mode 解锁
    __CFRunLoopModeUnlock(rlm);
    // 对 runloop 解锁
    __CFRunLoopUnlock(rl);
    
    // 初始化一个空的指针
    struct _block_item *prev = NULL;
    // 初始化一个指向链表头结点的指针
    struct _block_item *item = head;
    // 从取得的 block 链表头指针开始遍历整个 block 链表
    while (item)
    {
        // 当前拿到的 _block_item 结构体变量
        struct _block_item *curr = item;
        // 指针往前移动一个位置，用于下一次循环
        item = item->_next;
        // 初始化 doit 布尔值
        Boolean doit = false;
        // 判断 _block_item 的 mode 是字符串还是集合，即这个 block_item 是对应一个 mode 还是多个 mode
        if (CFStringGetTypeID() == CFGetTypeID(curr->_mode))
        {
            // 说明是一个 mode
            // 判断两种情况
            // 1.传入的 mode 是否与 block_item 内部的 mode 相等
            // 2.判断 block_item 内部的 mode 是否为 kCFRunLoopCommonModes，同时判断 commonModes 里面是否有传入的 mode
            // 以上情况有一个满足，则 doit 为 true，否则 doit 为 false
            doit = CFEqual(curr->_mode, curMode) || (CFEqual(curr->_mode, kCFRunLoopCommonModes) && CFSetContainsValue(commonModes, curMode));
        }
        else
        {
            // 说明是多个 mode
            // 也是判断两种情况
            // 1.判断 _block_item 内部的 mode 集合是否包含传入的 mode
            // 2.判断 _block_item 内部的 mode 集合是否包含 kCFRunLoopCommonModes，以及 commonModes 里面是否有传入的 mode
            // 以上情况有一个满足，则 doit 为 true，否则 doit 为 false
            doit = CFSetContainsValue((CFSetRef)curr->_mode, curMode) || (CFSetContainsValue((CFSetRef)curr->_mode, kCFRunLoopCommonModes) && CFSetContainsValue(commonModes, curMode));
        }
        // 说白了只要 _block_item 节点满足 mode 匹配关系，就要执行其内部的 block 任务
        
        // 如果 doit 为 false，说明要执行 block 任务的不是当前的 _block_item,则将 prev 指针指向当前的 _block_item
        // 然后因为在前面已经做过 item = item->_next 的操作了，此时的 item 已经指向下一个 _block_item 了
        if (!doit)
            prev = curr;
        
        // 如果 doit 为 true，说明要执行 block 任务的就是当前的 _block_item
        if (doit)
        {
            // 这里其实可以分为三种情况
            // 一.目标节点在头尾之间的任一节点
            // 二.目标节点在头节点
            // 三.目标节点在尾节点
            // 因为已经执行完了 block 任务，需要删除目标节点
            
            // 判断 prev 指针是否为空，如果不为空，则将 prev 指针的 _next 指向 item，注意，此时 item 为下一次要遍历的 _block_item
            // 这里的作用就是情况一
            if (prev)
                prev->_next = item;
            
            // 下面分别判断了两种边界值情况，即要执行 block 的任务为头节点或尾节点
            
            // 判断如果当前正在遍历的 _block_time 是否等于 head 指针
            // 因为 head 在进入 while 循环之前指向的链表的头结点，所以能够进入 if 内部逻辑的条件
            // 说白了也就是刚好头结点就是要执行 block 任务的那个节点，此时将 head 指针指向下一节点，在循环结束后有一个判断，
            // 如果 head 不为空，然后里面的有一步操作就是重新让 rl->_blocks_head 指向 head，也就是说 执行了 block 任务且刚好为头节点的节点被删除了。
            
            // 这里的作用就是情况二
            if (curr == head)
                head = item;
            
            // 判断如果当前正在遍历的 _block_time 是否等于 tail 指针
            // 因为 tail 指针在 while 循环之前是指向的链表的尾节点，所有能够进入 if 内部逻辑条件
            // 说白了就是刚好尾节点就是要执行 block 任务的那个节点，此时将 tail 指针指向 prev 指针指向的节点，
            // 而 prev 指针的指向，我们知道，显然就是尾节点的前一个节点，说白了就是把执行了 block 任务且刚好为尾节点的节点被删除了
            
            // 这里的作用就是情况三
            if (curr == tail)
                tail = prev;
            
            // 取出如果当前正在遍历的 _block_item 中的 block
            void (^block)(void) = curr->_block;
            // 释放如果当前正在遍历的 _block_item 的 mode
            CFRelease(curr->_mode);
            // 释放如果当前正在遍历的 _block_item
            free(curr);
            
            // 再次判断 doit
            if (doit)
            {
                // 传入 block，执行任务
                __CFRUNLOOP_IS_CALLING_OUT_TO_A_BLOCK__(block);
                // 将 did 置为 true
                did = true;
            }
            // 释放 block
            Block_release(block); // do this before relocking to prevent deadlocks where some yahoo wants to run the run loop reentrantly from their dealloc
            // 在重新加锁之前释放 blcok 可以防止程序员在 dealloc 方法里面重新让 runloop 运行起来而导致死锁的情况
        }
    }
    // 对 runloop 加锁
    __CFRunLoopLock(rl);
    // 对  mode 加锁
    __CFRunLoopModeLock(rlm);
    // 如果 head 指针不为空，这里 if 的逻辑也是必进
    if (head)
    {
        // 因为在前面已经对 rl->_blocks_head 指针进行了置空操作，这里等价于 tail->_next = NULL
        tail->_next = rl->_blocks_head;
        // 让 rl->_blocks_head 指向 head 指针指向的节点
        // 如果在上面的 while 循环中，没找到要执行的 block，那么 head 其实就是原来 rl->_blocks_head 的值
        // 如果在上面的 while 循环中，找到了要执行的 block，那么 head 是指向的
        rl->_blocks_head = head;
        // 因为在前面已经对 rl->_blocks_tail 指针进行了置空操作,所以这里 if 的逻辑必进
        // 所以这里就是让 rl->_blocks_tail 指向 tail 指针指向的节点
        if (!rl->_blocks_tail)
            rl->_blocks_tail = tail;
    }
    return did;
}
```

可以看到真正执行回调函数的是 __CFRUNLOOP_IS_CALLING_OUT_TO_A__BLOCK__ 函数，其内部实现如下

```c
static void __CFRUNLOOP_IS_CALLING_OUT_TO_A_BLOCK__() __attribute__((noinline));
static void __CFRUNLOOP_IS_CALLING_OUT_TO_A_BLOCK__(void (^block)(void))
{
    if (block)
    {
        block();
    }
    asm __volatile__(""); // thwart tail-call optimization
}
```

<a name="ti2qJ"></a>
### __CFRunLoopDoSources0

```c
/* rl is locked, rlm is locked on entrance and exit */
static Boolean __CFRunLoopDoSources0(CFRunLoopRef rl, CFRunLoopModeRef rlm, Boolean stopAfterHandle) __attribute__((noinline));
static Boolean __CFRunLoopDoSources0(CFRunLoopRef rl, CFRunLoopModeRef rlm, Boolean stopAfterHandle)
{ /* DOES CALLOUT */
    CHECK_FOR_FORK();
    // 初始化 sources ，并置空
    CFTypeRef sources = NULL;
    // 初始化返回值 sourceHandled，并置为 false ，意为是否已经处理了 source0
    Boolean sourceHandled = false;

    /* Fire the version 0 sources */
    if (NULL != rlm->_sources0 && 0 < CFSetGetCount(rlm->_sources0))
    {
        // 判断 mode 的 _source0 不为空且大小大于0
        // CFSetApplyFunction (Calls a function once for each value in the set.)
        // 这个函数的作用是对传入的 set 里面的每个元素执行 传入的函数指针
        // 第一个参数是 set
        // 第二个参数是 要对set每个元素执行一次的函数指针
        // 第三个参数作为传入的函数指针的第二个参数
        CFSetApplyFunction(rlm->_sources0, (__CFRunLoopCollectSources0), &sources);
    }
    // 此时 sources 里面已经有全部的 source0 了
    if (NULL != sources)
    {
        // 对 mode 解锁
        __CFRunLoopModeUnlock(rlm);
        // 对 runloop 解锁
        __CFRunLoopUnlock(rl);
        // sources 有可能是一个 runloop source 也有可能是一个装的 runloop source 的数组
        // sources is either a single (retained) CFRunLoopSourceRef or an array of (retained) CFRunLoopSourceRef
        if (CFGetTypeID(sources) == CFRunLoopSourceGetTypeID())
        {
            // sources 是单个 runloop source
            // 类型强转一下
            CFRunLoopSourceRef rls = (CFRunLoopSourceRef)sources;
            // 对 rls 加锁
            __CFRunLoopSourceLock(rls);
            if (__CFRunLoopSourceIsSignaled(rls))
            {
                // 来到这，说明 rls 被标记为 signaled
                // 取消 signaled 的标志
                __CFRunLoopSourceUnsetSignaled(rls);
                // 判断 rls 是否有效
                if (__CFIsValid(rls))
                {
                    // 对 rls 进行解锁
                    __CFRunLoopSourceUnlock(rls);
                    // 执行 source0 回调
                    /**
                     static void __CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__() __attribute__((noinline));
                     static void __CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__(void (*perform)(void *), void *info)
                     {
                         if (perform)
                         {
                             perform(info);
                         }
                         asm __volatile__(""); // thwart tail-call optimization
                     }
                     */
                    __CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__(rls->_context.version0.perform, rls->_context.version0.info);
                    CHECK_FOR_FORK();
                    // 将处理 source0 的结果置为 true
                    sourceHandled = true;
                }
                else
                {
                    __CFRunLoopSourceUnlock(rls);
                }
            }
            else
            {
                // 对 rls 解锁，来到这，说明 rls 没有被标记为 signaled
                __CFRunLoopSourceUnlock(rls);
            }
        }
        else
        {
            // sources 是数组
            // 取出 source0 的个数
            CFIndex cnt = CFArrayGetCount((CFArrayRef)sources);
            // 对 sources 进行排序，按照 _order 进行升序排序
            /**
             static CFComparisonResult __CFRunLoopSourceComparator(const void *val1, const void *val2, void *context)
             {
                 CFRunLoopSourceRef o1 = (CFRunLoopSourceRef)val1;
                 CFRunLoopSourceRef o2 = (CFRunLoopSourceRef)val2;
                 if (o1->_order < o2->_order)
                     return kCFCompareLessThan;
                 if (o2->_order < o1->_order)
                     return kCFCompareGreaterThan;
                 return kCFCompareEqualTo;
             }
             */
            CFArraySortValues((CFMutableArrayRef)sources, CFRangeMake(0, cnt), (__CFRunLoopSourceComparator), NULL);
            // 遍历 sources 数组
            for (CFIndex idx = 0; idx < cnt; idx++)
            {
                // 取出一个 source0 rls
                CFRunLoopSourceRef rls = (CFRunLoopSourceRef)CFArrayGetValueAtIndex((CFArrayRef)sources, idx);
                // 对 rls 加锁
                __CFRunLoopSourceLock(rls);
                if (__CFRunLoopSourceIsSignaled(rls))
                {
                    // 来到这，说明 rls 被标记为 signaled
                    // 取消 signaled 的标志
                    __CFRunLoopSourceUnsetSignaled(rls);
                    // 判断 rls 是否有效
                    if (__CFIsValid(rls))
                    {
                        // 对 rls 进行解锁
                        __CFRunLoopSourceUnlock(rls);
                        // 执行 source0 回调
                        /**
                         static void __CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__() __attribute__((noinline));
                         static void __CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__(void (*perform)(void *), void *info)
                         {
                             if (perform)
                             {
                                 perform(info);
                             }
                             asm __volatile__(""); // thwart tail-call optimization
                         }
                         */
                        __CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__(rls->_context.version0.perform, rls->_context.version0.info);
                        CHECK_FOR_FORK();
                        // 将处理 source0 的结果置为 true
                        sourceHandled = true;
                    }
                    else
                    {
                        __CFRunLoopSourceUnlock(rls);
                    }
                }
                else
                {
                    // 对 rls 解锁，来到这，说明 rls 没有被标记为 signaled
                    __CFRunLoopSourceUnlock(rls);
                }
                // 如果有一个 source0 处理完成且 传入的 stopAfterHandle 为 true，则跳出循环
                if (stopAfterHandle && sourceHandled)
                {
                    break;
                }
            }
        }
        // 释放 sources
        CFRelease(sources);
        // 对 runloop 加锁
        __CFRunLoopLock(rl);
        // 对 mode 解锁
        __CFRunLoopModeLock(rlm);
    }
    // 返回处理 source0 结果
    return sourceHandled;
}
```

而真正执行回调的是 `__CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__` 函数，其内部实现如下

```c
static void __CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__() __attribute__((noinline));
static void __CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__(void (*perform)(void *), void *info)
{
    if (perform)
    {
        perform(info);
    }
    asm __volatile__(""); // thwart tail-call optimization
}
```

<a name="Xn8lf"></a>
### __CFRunLoopDoSource1

```c
// msg, size and reply are unused on Windows
static Boolean __CFRunLoopDoSource1() __attribute__((noinline));
static Boolean __CFRunLoopDoSource1(CFRunLoopRef rl, CFRunLoopModeRef rlm, CFRunLoopSourceRef rls
#if DEPLOYMENT_TARGET_MACOSX || DEPLOYMENT_TARGET_EMBEDDED || DEPLOYMENT_TARGET_EMBEDDED_MINI
                                    ,
                                    mach_msg_header_t *msg, CFIndex size, mach_msg_header_t **reply
#endif
)
{ /* DOES CALLOUT */
    CHECK_FOR_FORK();
    // 初始化 是否处理成功 source1 结果
    Boolean sourceHandled = false;

    /* Fire a version 1 source */
    // 对 source1 进行 retian
    CFRetain(rls);
    // 对 mode 解锁
    __CFRunLoopModeUnlock(rlm);
    // 对 runloop 解锁
    __CFRunLoopUnlock(rl);
    // 对 source1 加锁
    __CFRunLoopSourceLock(rls);
    // 如果 source1 有效
    if (__CFIsValid(rls))
    {
        /**
         // Bit 1 of the base reserved bits is used for signalled state

         CF_INLINE Boolean __CFRunLoopSourceIsSignaled(CFRunLoopSourceRef rls)
         {
             return (Boolean)__CFBitfieldGetValue(rls->_bits, 1, 1);
         }

         CF_INLINE void __CFRunLoopSourceSetSignaled(CFRunLoopSourceRef rls)
         {
             __CFBitfieldSetValue(rls->_bits, 1, 1, 1);
         }

         CF_INLINE void __CFRunLoopSourceUnsetSignaled(CFRunLoopSourceRef rls)
         {
             __CFBitfieldSetValue(rls->_bits, 1, 1, 0);
         }
         */
        // 设置 source1 状态为未激活
        __CFRunLoopSourceUnsetSignaled(rls);
        // 解锁 source1
        __CFRunLoopSourceUnlock(rls);
        __CFRunLoopDebugInfoForRunLoopSource(rls);
        // 执行 source1 回调
        __CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE1_PERFORM_FUNCTION__(rls->_context.version1.perform,
#if DEPLOYMENT_TARGET_MACOSX || DEPLOYMENT_TARGET_EMBEDDED || DEPLOYMENT_TARGET_EMBEDDED_MINI
                                                                   msg, size, reply,
#endif
                                                                   rls->_context.version1.info);
        CHECK_FOR_FORK();
        sourceHandled = true;
    }
    else
    {
        // 写入到日志，source1 失效
        if (_LogCFRunLoop)
        {
            CFLog(kCFLogLevelDebug, CFSTR("%p (%s) __CFRunLoopDoSource1 rls %p is invalid"), CFRunLoopGetCurrent(), *_CFGetProgname(), rls);
        }
        // 对 source1 解锁
        __CFRunLoopSourceUnlock(rls);
    }
    // 释放 source1
    CFRelease(rls);
    // 对 runloop 加锁
    __CFRunLoopLock(rl);
    // 对 mode 加锁
    __CFRunLoopModeLock(rlm);
    return sourceHandled;
}
```

其中真正的回调函数为 `__CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE1_PERFORM_FUNCTION__` ，其内部实现为:

```c

static void __CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE1_PERFORM_FUNCTION__() __attribute__((noinline));
static void __CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE1_PERFORM_FUNCTION__(
#if DEPLOYMENT_TARGET_MACOSX || DEPLOYMENT_TARGET_EMBEDDED || DEPLOYMENT_TARGET_EMBEDDED_MINI
    void *(*perform)(void *msg, CFIndex size, CFAllocatorRef allocator, void *info),
    mach_msg_header_t *msg, CFIndex size, mach_msg_header_t **reply,
#else
    void (*perform)(void *),
#endif
    void *info)
{
    if (perform)
    {
#if DEPLOYMENT_TARGET_MACOSX || DEPLOYMENT_TARGET_EMBEDDED || DEPLOYMENT_TARGET_EMBEDDED_MINI
        *reply = perform(msg, size, kCFAllocatorSystemDefault, info);
#else
        perform(info);
#endif
    }
    asm __volatile__(""); // thwart tail-call optimization
}
```

<a name="CqLBJ"></a>
### __CFRunLoopDoTimers

```c
// rl and rlm are locked on entry and exit
static Boolean __CFRunLoopDoTimers(CFRunLoopRef rl, CFRunLoopModeRef rlm, uint64_t limitTSR)
{ /* DOES CALLOUT */
    // 初始化 timer 是否被处理成功 结果
    Boolean timerHandled = false;
    // 初始化一个空的 timers 集合
    CFMutableArrayRef timers = NULL;
    // 遍历 mode 的 _itemrs 定时器数组
    for (CFIndex idx = 0, cnt = rlm->_timers ? CFArrayGetCount(rlm->_timers) : 0; idx < cnt; idx++)
    {
        // 取出 timer
        CFRunLoopTimerRef rlt = (CFRunLoopTimerRef)CFArrayGetValueAtIndex(rlm->_timers, idx);
        
        // 如果 timer 有效且尚未被触发
        if (__CFIsValid(rlt) && !__CFRunLoopTimerIsFiring(rlt))
        {
            // 并且 timer 的触发时间小于等于 限定的时间
            if (rlt->_fireTSR <= limitTSR)
            {
                // 初始化 timers 集合
                if (!timers)
                    timers = CFArrayCreateMutable(kCFAllocatorSystemDefault, 0, &kCFTypeArrayCallBacks);
                // 把 timer 加入 timers 集合
                CFArrayAppendValue(timers, rlt);
            }
        }
    }

    // 遍历 timers 集合，里面装的都是要干活的 timer
    for (CFIndex idx = 0, cnt = timers ? CFArrayGetCount(timers) : 0; idx < cnt; idx++)
    {
        // 取出 timer
        CFRunLoopTimerRef rlt = (CFRunLoopTimerRef)CFArrayGetValueAtIndex(timers, idx);
        // 执行 timer 回调
        Boolean did = __CFRunLoopDoTimer(rl, rlm, rlt);
        timerHandled = timerHandled || did;
    }
    // 释放 timers 集合
    if (timers)
        CFRelease(timers);
    return timerHandled;
}

static Boolean __CFRunLoopDoTimer(CFRunLoopRef rl, CFRunLoopModeRef rlm, CFRunLoopTimerRef rlt)
{ /* DOES CALLOUT */
    // 初始化 timer 被处理成功结果
    Boolean timerHandled = false;
    uint64_t oldFireTSR = 0;

    /* Fire a timer */
    // 对 timer retain
    CFRetain(rlt);
    // 对 timer 加锁
    __CFRunLoopTimerLock(rlt);

    // 如果 timer 有效，并且 timer 触发时间点未到，以及 timer 没有被触发，以及 timer 的 runloop 就是传入的 runloop ，进入 if 逻辑
    if (__CFIsValid(rlt) && rlt->_fireTSR <= mach_absolute_time() && !__CFRunLoopTimerIsFiring(rlt) && rlt->_runLoop == rl)
    {
        // 初始化空的 context_info
        void *context_info = NULL;
        // 初始化空的 context_release 函数指针
        void (*context_release)(const void *) = NULL;
        // 如果 timer 的 context 属性的 retain 函数指针不为空，进入 if 逻辑
        if (rlt->_context.retain)
        {
            // 将 info retain 后赋值给 context_info
            context_info = (void *)rlt->_context.retain(rlt->_context.info);
            // 取出 timer 的 release 函数指针赋值给 context_release
            context_release = rlt->_context.release;
        }
        else
        {
            // 取出 timer 的 info
            context_info = rlt->_context.info;
        }
        // timer 的时间间隔是否为 0
        Boolean doInvalidate = (0.0 == rlt->_interval);
        // 设置 timer 的 是否触发的 bits 为1
        __CFRunLoopTimerSetFiring(rlt);
        // Just in case the next timer has exactly the same deadlines as this one, we reset these values so that the arm next timer code can correctly find the next timer in the list and arm the underlying timer.
        // 以防万一下一个计时器与该计时器的截止日期完全相同，我们重置这些值，以便启用下一个计时器代码可以正确地找到列表中的下一个计时器并启用基础计时器。
        rlm->_timerSoftDeadline = UINT64_MAX;
        rlm->_timerHardDeadline = UINT64_MAX;
        // 解锁 timer
        __CFRunLoopTimerUnlock(rlt);
        // 对 timer 触发加锁
        __CFRunLoopTimerFireTSRLock();
        // 取出 _fireTSR
        oldFireTSR = rlt->_fireTSR;
        // 对 timer 触发解锁
        __CFRunLoopTimerFireTSRUnlock();
        
        // 装配下一个 timer
        __CFArmNextTimerInMode(rlm, rl);

        // 对 mode 解锁
        __CFRunLoopModeUnlock(rlm);
        // 对 runloop 解锁
        __CFRunLoopUnlock(rl);
        // 执行 timer 真正的回调
        __CFRUNLOOP_IS_CALLING_OUT_TO_A_TIMER_CALLBACK_FUNCTION__(rlt->_callout, rlt, context_info);
        CHECK_FOR_FORK();
        // 如果 timer 只跑一次
        if (doInvalidate)
        {
            // 让 timer 失效
            CFRunLoopTimerInvalidate(rlt); /* DOES CALLOUT */
        }
        // 如果 context_release 函数指针不为空，则释放掉 context_info
        if (context_release)
        {
            context_release(context_info);
        }
        // 对 runloop 加锁
        __CFRunLoopLock(rl);
        // 对 mode 解锁
        __CFRunLoopModeLock(rlm);
        // 对 timer 加锁
        __CFRunLoopTimerLock(rlt);
        // 设置 处理 timer 结果为 true
        timerHandled = true;
        // 设置 timer 的 是否触发的 bits 为0
        __CFRunLoopTimerUnsetFiring(rlt);
    }
    // 如果 timer 有效，并且 timer 被处理成功
    if (__CFIsValid(rlt) && timerHandled)
    {
        /* This is just a little bit tricky: we want to support calling
         * CFRunLoopTimerSetNextFireDate() from within the callout and
         * honor that new time here if it is a later date, otherwise
         * it is completely ignored. */
        // 如果 旧的触发时间 小于 当前的触发时间
        if (oldFireTSR < rlt->_fireTSR)
        {
            /* Next fire TSR was set, and set to a date after the previous
            * fire date, so we honor it. */
            __CFRunLoopTimerUnlock(rlt);
            // The timer was adjusted and repositioned, during the
            // callout, but if it was still the min timer, it was
            // skipped because it was firing.  Need to redo the
            // min timer calculation in case rlt should now be that
            // timer instead of whatever was chosen.
            __CFArmNextTimerInMode(rlm, rl);
        }
        else
        {
            uint64_t nextFireTSR = 0LL;
            uint64_t intervalTSR = 0LL;
            if (rlt->_interval <= 0.0)
            {
            }
            else if (TIMER_INTERVAL_LIMIT < rlt->_interval)
            {
                intervalTSR = __CFTimeIntervalToTSR(TIMER_INTERVAL_LIMIT);
            }
            else
            {
                intervalTSR = __CFTimeIntervalToTSR(rlt->_interval);
            }
            if (LLONG_MAX - intervalTSR <= oldFireTSR)
            {
                nextFireTSR = LLONG_MAX;
            }
            else
            {
                if (intervalTSR == 0)
                {
                    // 15304159: Make sure we don't accidentally loop forever here
                    CRSetCrashLogMessage("A CFRunLoopTimer with an interval of 0 is set to repeat");
                    HALT;
                }
                uint64_t currentTSR = mach_absolute_time();
                nextFireTSR = oldFireTSR;
                while (nextFireTSR <= currentTSR)
                {
                    nextFireTSR += intervalTSR;
                }
            }
            CFRunLoopRef rlt_rl = rlt->_runLoop;
            if (rlt_rl)
            {
                CFRetain(rlt_rl);
                CFIndex cnt = CFSetGetCount(rlt->_rlModes);
                STACK_BUFFER_DECL(CFTypeRef, modes, cnt);
                CFSetGetValues(rlt->_rlModes, (const void **)modes);
                // To avoid A->B, B->A lock ordering issues when coming up
                // towards the run loop from a source, the timer has to be
                // unlocked, which means we have to protect from object
                // invalidation, although that's somewhat expensive.
                for (CFIndex idx = 0; idx < cnt; idx++)
                {
                    CFRetain(modes[idx]);
                }
                __CFRunLoopTimerUnlock(rlt);
                for (CFIndex idx = 0; idx < cnt; idx++)
                {
                    CFStringRef name = (CFStringRef)modes[idx];
                    modes[idx] = (CFTypeRef)__CFRunLoopFindMode(rlt_rl, name, false);
                    CFRelease(name);
                }
                __CFRunLoopTimerFireTSRLock();
                rlt->_fireTSR = nextFireTSR;
                rlt->_nextFireDate = CFAbsoluteTimeGetCurrent() + __CFTimeIntervalUntilTSR(nextFireTSR);
                for (CFIndex idx = 0; idx < cnt; idx++)
                {
                    CFRunLoopModeRef rlm = (CFRunLoopModeRef)modes[idx];
                    if (rlm)
                    {
                        __CFRepositionTimerInMode(rlm, rlt, true);
                    }
                }
                __CFRunLoopTimerFireTSRUnlock();
                for (CFIndex idx = 0; idx < cnt; idx++)
                {
                    __CFRunLoopModeUnlock((CFRunLoopModeRef)modes[idx]);
                }
                CFRelease(rlt_rl);
            }
            else
            {
                __CFRunLoopTimerUnlock(rlt);
                __CFRunLoopTimerFireTSRLock();
                rlt->_fireTSR = nextFireTSR;
                rlt->_nextFireDate = CFAbsoluteTimeGetCurrent() + __CFTimeIntervalUntilTSR(nextFireTSR);
                __CFRunLoopTimerFireTSRUnlock();
            }
        }
    }
    else
    {
        // 解锁 timer
        __CFRunLoopTimerUnlock(rlt);
    }
    // 释放掉 timer
    CFRelease(rlt);
    return timerHandled;
}
```

其中真正执行回调的是 __CFRUNLOOP_IS_CALLING_OUT_TO_A_TIMER_CALLBACK_FUNCTION__ 函数，其内部实现如下:

```c
static void __CFRUNLOOP_IS_CALLING_OUT_TO_A_TIMER_CALLBACK_FUNCTION__() __attribute__((noinline));
static void __CFRUNLOOP_IS_CALLING_OUT_TO_A_TIMER_CALLBACK_FUNCTION__(CFRunLoopTimerCallBack func, CFRunLoopTimerRef timer, void *info)
{
    if (func)
    {
        func(timer, info);
    }
    asm __volatile__(""); // thwart tail-call optimization
}
```

<a name="c2oZn"></a>
### __CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__

```c
static void __CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__() __attribute__((noinline));
static void __CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__(void *msg)
{
    _dispatch_main_queue_callback_4CF(msg);
    asm __volatile__(""); // thwart tail-call optimization
}
```

# 总结

![RunLoop 流程](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200316102221.jpg)


# 参考资料

[深入理解 RunLoop - ibireme](https://blog.ibireme.com/2015/05/18/runloop/)

[解密 RunLoop - MrPeak](http://mrpeak.cn/blog/ios-runloop/)

[Core Foundation 详解](https://www.jianshu.com/p/5c98ac2dab58)
