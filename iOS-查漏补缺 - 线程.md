多线程是我们开发和面试中都会遇到的一个重要概念，相比于其他编程语言和平台，iOS 的多线程使用起来要比较友好和易用一些。但是对于多线程的基本概念，我们还是需要重视起来，这对于我们探索 `pthread`、`NSThread`、`GCD` 以及 `RunLoop` 都大有裨益。

> 本节的大部分内容基于苹果官方文档。文档地址: [About Threaded Programming](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Multithreading/AboutThreads/AboutThreads.html#//apple_ref/doc/uid/10000057i-CH6-SW2)
<!-- more -->
# 前导知识

* POSIX

> POSIX Threads, usually referred to as pthreads, is an execution model that exists independently from a language, as well as a parallel execution model. It allows a program to control multiple different flows of work that overlap in time. Each flow of work is referred to as a thread, and creation and control over these flows is achieved by making calls to the POSIX Threads API.

> – POSIX Threads, Wikipedia

> 译：

> POSIX（Portable Operating System Interface of UNIX，可移植操作系统接口）线程，即 pthreads，是一种**不依赖于语言的执行模型**，也称作并行（Parallel）执行模型。其允许一个程序控制多个时间重叠的不同工作流。每个工作流即为一个线程，通过调用 POSIX 线程 API 创建并控制这些流。

> – POSIX 线程，维基百科

* detached 和 joinable

> 无论在windows中还是Posix中，主线程和子线程的默认关系是：无论子线程执行完毕与否，一旦主线程执行完毕退出，所有子线程执行都会终止。这时整个进程结束或僵死，部分线程保持一种终止执行但还未销毁的状态，而进程必须在其所有线程销毁后销毁，这时进程处于僵死状态。线程函数执行完毕退出，或以其他非常方式终止，线程进入终止态，但是为线程分配的系统资源不一定释放，可能在系统重启之前，一直都不能释放，终止态的线程，仍旧作为一个线程实体存在于操作系统中，什么时候销毁，取决于线程属性。在这种情况下，主线程和子线程通常定义以下两种关系：

> 1、**可会合（joinable**）：这种关系下，主线程需要明确执行等待操作，在子线程结束后，主线程的等待操作执行完毕，子线程和主线程会合，这时主线程继续执行等待操作之后的下一步操作。主线程必须会合可会合的子线程。在主线程的线程函数内部调用子线程对象的wait函数实现，即使子线程能够在主线程之前执行完毕，进入终止态，也必须执行会合操作，否则，系统永远不会主动销毁线程，分配给该线程的系统资源也永远不会释放。

> 2、**相分离（detached）**：表示子线程无需和主线程会合，也就是相分离的，这种情况下，子线程一旦进入终止状态，这种方式常用在线程数较多的情况下，有时让主线程逐个等待子线程结束，或者让主线程安排每个子线程结束的等待顺序，是很困难或不可能的，所以在并发子线程较多的情况下，这种方式也会经常使用。

> 在任何一个时间点上，线程是可结合（joinable）或者是可分离的（detached），一个可结合的线程能够被其他线程回收资源和杀死，在被其他线程回收之前，它的存储器资源如栈，是不释放的，相反，一个分离的线程是不能被其他线程回收或杀死的，它的存储器资源在它终止时由系统自动释放。

> 线程的分离状态决定一个线程以什么样的方式来终止自己，在默认的情况下，线程是非分离状态的，这种情况下，原有的线程等待创建的线程结束，只有当pthread_join函数返回时，创建的线程才算终止，释放自己占用的系统资源，而分离线程没有被其他的线程所等待，自己运行结束了，线程也就终止了，马上释放系统资源。

joinable 和 detaced 其实是主线程与子线程之间的一种关系。app 退出后，detached 线程直接进入终止态，其栈空间资源被系统回收，但是 joinable 线程资源不会被回收，所以可以在 app 退出的时候使用 joinable 线程也就是 pthread 来做一些保存资源到磁盘的操作。也就是说在主线程执行完毕要退出之前，会去处理joinable的线程。

# 线程初探

## 线程定义

要先了解什么是线程，我们需要先了解什么是进程。对于 iOS 来说，每一个 `app` 其实就是一个进程，这一点和 `Android` 有很大的区别。除了单进程之外，在未越狱之前，每个 `app` 只能访问每个 `app` 自身的沙盒环境，不能访问之外的内容。得益于这样的设计，`iOS` 成为了世界上最安全的操作系统（苹果是这么的说的🐶）。

下面给出苹果官方对于线程的定义

> Threads are a relatively lightweight way to implement multiple paths of execution inside of an application.
> 
> 【译】线程是在应用程序内部实现多个执行路径的相对轻量的方法。  

> From a technical standpoint, a thread is a combination of the kernel-level and application-level data structures needed to manage the execution of code. The kernel-level structures coordinate the dispatching of events to the thread and the preemptive scheduling of the thread on one of the available cores. The application-level structures include the call stack for storing function calls and the structures the application needs to manage and manipulate the thread’s attributes and state.
> 
> 【译】从技术角度来看，线程是管理代码执行所需的**内核级和应用程序级**数据结构的组合。内核负责线程事件的分发和线程优先级的调度，应用层则负责存储线程函数中断时的状态和属性的存储方便下次内核切换时再从存储的地方运行。

线程的定义可以总结为以下三点:

* 线程是进程的基本执行单元，一个进程的所有任务都在线程中执行
* 进程想要执行任务，必须得有线程，进程至少要有一条线程
* 程序启动会默认开启一条线程，这条线程被称为主线程或 UI 线程

下面我们看一下进程的定义：

* 进程是指在系统中正在运行的一个应用程序
* 每个进程之间是独立的，每个进程均运行在其专用的且受保护的内存空间内

这两句话不难理解，只针对于 `iOS` 来说，一个 `app` 就是一个进程，而由于有沙盒机制，每个 `app` 所对应的进程只能访问当前 `app` 沙盒所对应的内存空间，是相互独立的。

我们的 `app` 只有一条进程，而线程的话，默认只有一条主线程。我们可以通过多线程技术来开线程然后启动线程来执行任务。而进程与线程之间的关系可以总结为下列几点：

* 地址空间：同一进程的**线程共享本进程**的地址空间，而**进程之间则是独立的**地址空间。
* 资源拥有：同一进程内的线程共享本进程的资源如内存、I/O、CPU 等，但是进程之间资源是独立的。
* 一个线程崩溃后，在保护模式下不会对其它进程产生影响，但是一个线程崩溃后会导致整个进程的崩溃，所以多进程相比多线程更加健壮。
* 进程切换时，消耗的资源大，效率低。所以涉及到频繁的上下文切换的时候，使用线程要好于进程。同样的，如果要求同时进行且要共享某些变量的并发操作，只能用线程，不能用进程。
* 执行过程：每个独立的进程有一个程序运行的入口、顺序执行序列和程序入口。但是线程不能独立执行，必须依存于应用程序中，由应用程序提供多个线程执行控制。
* 线程是 CPU 调度的基本单位，进程不是。

## 线程相关的术语

在深入讨论线程及其支持技术之前，有必要弄清楚一些基本术语。

如果你熟悉 `UNIX` 系统，术语 `任务` 于表示正在运行的进程，但在本文中并不是这样定义的。

本文采用的术语如下:

* 术语 `线程` 用于指代代码的独立执行路径。
* 术语 `进程` 用于指代一个正在运行的可执行文件，它可以包含多个线程。
* 术语 `任务` 用于指代需要执行的工作的抽象概念。

## 线程的替代方案

手动创建线程会给我们的代码带来一定的**不确定性**。相对来说线程属于**抽象层次比较低**且使用起来比较麻烦的一种让应用程序支持并发的方案。如果你对于直接使用线程不熟悉的话，那么很容易遇到线程同步和时序问题，其严重性可能从细微的问题到应用程序崩溃和用户数据损坏。

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/15840724400661.jpg)


所以如上图所示，苹果官方给出了以下几种线程的替代方案:

* `Operation Objects`: 任务对象(`NSOpeartion`)是 `OS X 10.5` 推出的一个特性。通过封装在子线程执行的任务，隐藏底层的线程管理的具体细节，让开发者聚焦于任务执行的本身。通常来说任务对象会与任务对象队列(`NSOperationQueue`)结合使用，来实现在一个或多个线程上任务的执行。
* `GCD`: `OS 10.6` 正式推出，`GCD` 是另外一种可以让开发者无需知道线程细节而专注于任务本身的一项技术。通过使用 `GCD`，你可以定义你需要执行的任务，然后把这个任务加入到一个队列中来让 `GCD` 选择在一个合适的线程上执行这个任务。队列考虑了可用核心的数量和当前负载，这样与直接使用线程相比可以更有效地执行任务。
* `Idle-time notifications`: 空闲时通知，对于优先级和复杂度相对来说不高的任务，空闲时通知技术可以在应用程序空闲时执行任务。`Cocoa` 使用 `NSNotificationQueue` 来实现这一技术。为了获得空闲时通知，你需要使用 `NSPostWhenIdle` 选项来向 `NSNotificationQueue` 队列发送通知，然后直到 `Runloop` 空闲时，队列才会执行的通知里面的具体任务。

> `NSNotificationQueue` 官方定义
> 
> Whereas a notification center distributes notifications when posted, notifications placed into the queue can be delayed until the end of the current pass through the run loop or until the run loop is idle. Duplicate notifications can be coalesced so that only one notification is sent although multiple notifications are posted.
> 
> 【译】通知中心收到发出的通知后会向在通知中心注册的观察者分发这些通知，而添加到 `NSNotificationQueue` 队列中的通知只会在两种情况下分发出去，分别是当前 `Runloop` 即将退出或者 `Runloop` 处于空闲状态时。重复的向 `NSNotificationQueue` 队列中加入通知会导致相同的通知被合并，这样到了发送时机，对于重复的通知只会发送一条出去。
>
> A notification queue maintains notifications in first in, first out (FIFO) order. When a notification moves to the front of the queue, the queue posts it to the notification center, which in turn dispatches the notification to all objects registered as observers.
>
> 【译】一个通知队列以先进先出的方式维护着通知。当一个通知移动到了队列的头部，队列会将这个通知发往通知中心，通知中心将通知分派给所有注册为观察者的对象。
>
> Every thread has a default notification queue, which is associated with the default notification center for the process. You can create your own notification queues and have multiple queues per center and thread.
> 
> 【译】每一个线程都有一个默认的通知队列，并且这个通知队列会与当前进程的默认的通知中心相关联。但是你可以创建自己的通知队列，使得每个通知中心和线程有多条通知队列

> 关于 `NSNotificationQueue` 的更多底层细节，可以参考这篇文章 [一文全解iOS通知机制](https://www.jianshu.com/p/ce8ea320ec69)

* `Asynchronous functions`: 异步的函数。系统自带的 `api` 包含许多可以为你提供自动并发功能的函数。这些函数的实现你无需关心，它们依托于系统的守护进程和进程或者会创建子线程来执行你所需要的任务。
* `Timers`: 定时器。你可以在应用程序的主线程上使用定时器来执行周期性的任务，这些任务虽然琐碎而无法使用线程，但仍需要定期进行维护。
* `Separate processes`: 尽管比线程更重，但是在任务仅与应用程序有切线关系的情况下，创建单独的进程可能会很有用。 如果任务需要大量内存或必须使用 root 特权执行，则可以使用进程。 例如，当 32 位应用程序将结果显示给用户时，你可以使用 64 位服务器进程来计算大型数据集。

## 线程支持

### `Cocoa` 中线程相关的技术

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/15839064594884.jpg)

如上图所示，这是苹果官网对于在 `Cocoa` 框架下能使用的线程技术。

简单翻译过来就是:

* `Cocoa Threads`: `Cocoa` 使用 `NSThread` 来实现线程。除此之外，`NSObject` 还提供了一揽子方法来派生线程和在已存在线程上执行任务。
* `POSIX threads`: `POSIX` 提供了基于 `C` 语言的一套创建线程的 `API`。如果不是创建 `Cocoa` 程序，那么这将是最好的创建线程的方案。`POSIX` 接口使用起来非常简单，并且为配置线程提供了足够的灵活性
* `Multiprocessing Services`: 多处理服务是从旧版 `Mac OS` 过渡到的应用程序使用的基于 `C` 的旧接口。此技术仅在 `OS X` 中可用，任何新开发都应避免使用。

线程启动之后，主要以三种状态运行，分别是:

* 运行态
* 就绪态
* 阻塞态

如果线程当前未在运行态，则它要么被阻塞并等待输入，要么准备运行，但尚未计划这样做。线程继续在这些状态之间来回移动，直到最终退出并进入终止状态。

### Runloop

> A run loop is a piece of infrastructure used to manage events arriving asynchronously on a thread. A run loop works by monitoring one or more event sources for the thread. As events arrive, the system wakes up the thread and dispatches the events to the run loop, which then dispatches them to the handlers you specify. If no events are present and ready to be handled, the run loop puts the thread to sleep.
> 
> 【译】一个运行循环是一个处理线程上所接收到的异步的事件的结构。运行循环管理线程上的一个或多个事件源。当事件到达时，系统将唤醒线程并将事件分配给运行循环，然后运行循环将其分配给你指定的处理程序。如果不存在任何事件或有待处理的事件，则运行循环会将线程置于睡眠状态。

> You are not required to use a run loop with any threads you create but doing so can provide a better experience for the user. Run loops make it possible to create long-lived threads that use a minimal amount of resources. Because a run loop puts its thread to sleep when there is nothing to do, it eliminates the need for polling, which wastes CPU cycles and prevents the processor itself from sleeping and saving power.
> 
> 【译】你不需要对你所创建的线程使用运行循环，但是使用运行循环可以提高用户体验。运行循环可以创建使用最少资源的常驻线程。因为当没事做的时候，运行循环会让线程休眠，这样就不许需要通过轮询这种需要消耗 CPU 的低效操作从而节能。

> To configure a run loop, all you have to do is launch your thread, get a reference to the run loop object, install your event handlers, and tell the run loop to run. The infrastructure provided by OS X handles the configuration of the main thread’s run loop for you automatically. If you plan to create long-lived secondary threads, however, you must configure the run loop for those threads yourself.
> 
> 【译】要配置运行循环，你要做的就是启动线程，获取运行循环对象的引用，安装事件处理程序，并告诉运行循环运行。 OS X 提供的基础结构会自动为你处理主线程运行循环的。但是，如果计划创建寿命长的辅助线程，则必须自己为这些线程配置运行循环。

通过上面官方文档的描述，runloop 其实和线程是紧密关联的，通过 runloop 可以让子线程一直存活而不被系统回收。同时，runloop 还能提升用户体验，可以重复的在子线程工作而无需为了执行任务多次开同样工作内容的线程。

### 线程同步

使用多线程技术会遇到多个线程同时访问同一份资源的情况，而如果这些线程同时尝试使用或修改资源，则会出现严重的问题。解决此问题的办法通常来说有两种，一种是消除共享资源，让每个线程独享其特有的资源进行操作。第二种就是通过 locks(锁), conditions(条件), atomic operations(原子操作)等其它技术。显然第二种方案使用频率更高。

### 线程间通信

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/15839085538498.jpg)

如上图所示，苹果官方给出了几种线程间通信的方式，简单总结一下如下：

* **直接消息传递**: 通过 `performSelector` 的一系列方法，可以实现由某一线程指定在另外的线程上执行任务。因为任务的执行上下文是目标线程，这种方式发送的消息将会自动的被序列化。
* **全局变量、共享内存块和对象**: 在两个线程之间传递信息的另一种简单方法是使用全局变量，共享对象或共享内存块。尽管共享变量既快速又简单，但是它们比直接消息传递更脆弱。必须使用锁或其他同步机制仔细保护共享变量，以确保代码的正确性。 否则可能会导致竞争状况，数据损坏或崩溃。
* **条件执行**: 条件是一种**同步工具**，可用于控制线程何时执行代码的特定部分。您可以将条件视为关守，让线程仅在满足指定条件时运行。
* **`Runloop sources`**: 一个自定义的 `Runloop source` 配置可以让一个线程上收到特定的应用程序消息。由于 `Runloop source` 是事件驱动的，因此在无事可做时，线程会自动进入睡眠状态，从而提高了线程的效率。
* `Ports and sockets`: 基于端口的通信是在两个线程之间进行通信的一种更为复杂的方法，但它也是一种非常可靠的技术。更重要的是，端口和套接字可用于与外部实体（例如其他进程和服务）进行通信。为了提高效率，使用 `Runloop source` 来实现端口，因此当端口上没有数据等待时，线程将进入睡眠状态。
* **消息队列**: 传统的多处理服务定义了先进先出（`FIFO`）队列抽象，用于管理传入和传出数据。尽管消息队列既简单又方便，但是它们不如其他一些通信技术高效。
* **`Cocoa` 分布式对象**: 分布式对象是一种 `Cocoa` 技术，可提供基于端口的通信的高级实现。尽管可以将这种技术用于线程间通信，但是强烈建议不要这样做，因为它会产生大量开销。分布式对象更适合与其他进程进行通信，尽管在这些进程之间进行事务的开销也很高

## 使用线程的注意点

* 避免显式的创建线程

> 手动编写线程创建代码很繁琐，并且可能容易出错，因此应尽可能避免这样做。
> 
> `OS X` 和 `iOS` 通过其他 `API` 为并发提供隐式支持。与其自己创建一个线程，不如考虑使用异步 `API`，`GCD` 或操作对象来完成工作。这些技术可以在底层为您完成与线程相关的工作，并且可以保证正确进行。此外，`GCD` 和操作对象等技术旨在根据当前系统负载调整活动线程的数量，从而比您自己的代码更有效地管理线程。

* 让线程合理的执行任务

> 如果决定手动创建和管理线程，请记住线程会消耗宝贵的系统资源。您应该尽力确保分配给线程的所有任务都可以长期有效地工作。同时，您不必担心终止花费大部分空闲时间的线程。线程占用的内存非常少，其中一些是 `wired memory`，因此释放空闲线程不仅有助于减少应用程序的内存占用，还可以释放更多的物理内存供其他系统进程使用。

> PS: Mac 中的内存使用可以分为四大类
> 
> * **Wired**(联动): 系统核心占用的，永远不会从系统物理内存中驱除。
> * **Active**(活跃): 表示这些内存数据正在使用中，或者刚被使用过。
> * **Inactive**(非活跃): 表示这些内存中的数据是有效的，但是最近没有被使用。
> * **Free**(可用空间): 表示这些内存中的数据是无效的，即内存剩余量。

> 开始终止空闲线程之前，应始终记录一组应用程序当前性能的基准测量值。 尝试更改后，请进行其他测量以确认更改实际上在提高性能，而不是损害性能。

* 避免共享数据结构

> 避免与线程相关的资源冲突的最简单和容易的方法是为程序中的每个线程提供所需数据的自己的副本。当您最小化线程之间的通信和资源争用时，并行代码最有效。

* 线程和用户界面

> 如果你的应用有图形化的用户界面，强烈建议在应用程序的主线程上接收用户相关的事件和启动界面更新的操作。这种方法有助于避免与处理用户事件和绘制窗口内容相关的同步问题。某些框架（例如 `Cocoa`）通常需要此行为，但是即使对于那些不需要的框架，在主线程上保留此行为也具有简化管理用户界面的逻辑的优势。

> 当然，在某些情况下，在非主线程上执行图形操作是有助于提高性能的。例如，您可以使用子线程来创建和处理图像以及执行其他与图像有关的计算操作。但是遇到不确定的图形操作的时候，最好在主线程上执行。

* 注意线程退出时的问题

> 进程一直运行到所有**非分离线程**都退出为止。默认情况下，仅将应用程序的主线程创建为**非分离式**。当然，你也可以创建**非分离**的子线程。 当用户退出应用程序时，通常认为立即终止所有分离的线程是适当的行为，因为分离的线程完成的工作被认为是可选的。但是，如果你的应用程序正在使用后台线程将数据保存到磁盘或执行其他关键工作，则可能需要将这些线程创建为非分离线程，以防止在应用程序退出时丢失数据。

> 将线程创建为**非分离线程**（也称为**可连接线程**）需要你进行额外的工作。因为大多数高级线程技术默认情况下都不创建**可连接线程**，所以你可能必须使用 `POSIX API` 创建线程。此外，你必须在应用程序的主线程中添加代码，以便在非分离线程最终退出时加入它们。

* 处理异常

> 抛出异常时，异常处理机制依赖于当前的调用堆栈来执行任何必要的清除工作。因为每个线程都有自己的调用堆栈，所以每个线程负责捕获自己的异常。在辅助线程中未能捕获异常与在主线程中未能捕获异常会有相同的后果：进程终止。你不能将未捕获的异常抛出到另一个线程进行处理。

> 如果你需要在当前线程中将异常情况通知另一个线程（例如主线程），则应捕获该异常，并简单地向该另一个线程发送一条消息，指出发生了什么。 根据您的模型和您要执行的操作，捕获到异常的线程可以继续处理（如果可能的话），等待指令或直接退出。

> 在某些情况下，可能会自动为你创建一个异常处理程序。 例如，`Objective-C` 中的`@synchronized` 指令包含一个隐式异常处理程序。

* 彻底的退出线程

> 退出线程的最佳方法自然是让线程到达其主入口点例程的末尾。尽管有立即终止线程的功能，但这些功能仅应作为最后的手段使用。在线程到达其自然终点之前终止该线程会阻止该线程自身的清理。如果线程已分配内存，打开文件或获取其他类型的资源，则您的代码可能无法回收这些资源，从而导致内存泄漏或其他潜在问题。

* 库中的线程安全

> 尽管应用程序开发人员可以控制应用程序是否使用多线程执行，但库开发人员不能。在开发库时，必须假设调用应用程序是多线程的，或者可以随时切换到多线程的。因此，你应该始终对代码的关键部分使用锁。
> 
对于库开发人员来说，仅当应用程序变成多线程时才创建锁是不明智的。如果你需要在某个时候锁定代码，请在使用库的早期创建 lock 对象，最好是通过某种显式调用来初始化库。尽管你也可以使用静态库初始化函数来创建此类锁，但只有在没有其他方法时才尝试这样做。执行初始化函数会增加加载库所需的时间，并可能对性能产生不利影响。

> 始终记住在库中平衡互斥锁的加锁和解锁。你还应该记住对库中的数据结构加锁，而不是依赖调用代码来提供线程安全的环境。
> 
> 如果你正在开发 `Cocoa` 库，那么如果你希望在应用程序变为多线程时得到通知，则可以注册为 `NSWillBecomeMultiThreadedNotification` 的观察者。但是，你不应该依赖于接收此通知，因为它可能在调用库代码之前被发送。

# 线程管理

## 线程开销

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/15840203905223.jpg)

如上图所示:

* 内核层的数据结构：开销为 1KB。这部分用于存储线程数据结构和属性，其中大部分是作为有线内存分配的，因此无法分页到磁盘。
* 栈空间：iOS 程序主线程开销为 1MB，macOS 程序主线程开销为 8MB，子线程开销为 512KB。
* 创建时间：大约 90ms，此值反映创建线程的初始调用与线程的入口点例程开始执行之间的时间。这些数字是通过分析在基于英特尔的 iMac 上创建线程时生成的平均值和中值来确定的，iMac 具有一个 2GHz 的双核处理器和一个运行 OSX V10.5 的 1GB RAM。

> 由于其底层内核支持，`Operation` 对象通常可以更快地创建线程。它们不是每次都从头开始创建线程，而是使用已经驻留在内核中的线程池来节省分配时间。

## 创建线程

线程创建出来后必须要执行任务才有意义，下面介绍几种创建线程的方式。

### 使用 NSThread 创建线程

通过 `NSThread` 创建线程有两种方式：

* 使用 `detachNewThreadSelector:toTarget:withObject:` 类方法来派生新的线程。
* 创建一个 `NSThread` 实例对象，然后调用 `start` 方法。

> 这两种技术都会在应用程序中创建**分离式线程**。分离的线程意味着当线程退出时，系统会自动回收该线程的资源。这也意味着您的代码以后不必显式地与线程联接。

下面给出两种方式的实际用法:

```objectivec
[NSThread detachNewThreadSelector:@selector(myThreadMainMethod:) toTarget:self withObject:nil];

NSThread* myThread = [[NSThread alloc] initWithTarget:self
                                        selector:@selector(myThreadMainMethod:)
                                        object:nil];
[myThread start];  // Actually create the thread
```

> 这里有一个注意点，采用实例化 `NSThread` 对象的方式，只有调用 `start` 方法后，在底层线程才会被创建出来。
> 
> 使用 `initWithTarget:selector:object:method` 的另一种方法是将 `NSThread` 子类化并重写其 `main` 方法。你可以使用此方法的重写版本来作为线程的主入口点。

如果你有一个已经创建好并且在运行中的 `NSThread` 线程对象，你可以通过 `performSelector:onThread:withObject:waitUntilDone:` 方法来向这个线程发送消息。这个方法是 `NSObject` 的分类中的方法，也就意味着几乎任何对象都能适用。使用这个方法发送的消息将由另一个线程直接执行，作为其正常运行循环处理的一部分。当然，这意味着目标线程必须在其运行循环中运行）。在这过程中你可能还需要适用锁来进行线程同步，当然，这比适用基于 port 的线程间同信要简单一些。

虽然 `performSelector:onThread:withObject:waitUntilDone:` 方法用起来很简单，但是对于频繁间的线程通信或时间敏感类的任务执行，请不要使用这个方法。

### 使用 POSIX 创建线程

> OS X 和 iOS 为使用 POSIX 线程 API 创建线程提供了基于 C 的支持。这种技术实际上可以用于任何类型的应用程序（包括 Cocoa 和 Cocoa Touch 应用程序），如果您为多个平台编写软件，则可能更方便。用于创建线程的 `POSIX` 例程被适当地调用为 `pthread_create`。

下面是对于 `pthread_create` 的简单使用示例:

```objectivec
#include <assert.h>
#include <pthread.h>
 
// 线程要执行的任务 
void* PosixThreadMainRoutine(void* data)
{
    // Do some work here.
 
    return NULL;
}
 
// 启动线程 
void LaunchThread()
{
    // 线程属性
    pthread_attr_t  attr;
    // 线程对象
    pthread_t       posixThreadID;
    // 返回值
    int             returnVal;
 
    // 初始化线程属性
    returnVal = pthread_attr_init(&attr);
    assert(!returnVal);
    // 设置线程为 detach 状态
    returnVal = pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_DETACHED);
    assert(!returnVal);
 
    // 创建线程，传入属性和要执行的任务
    int     threadError = pthread_create(&posixThreadID, &attr, &PosixThreadMainRoutine, NULL);
    
    // 销毁属性
    returnVal = pthread_attr_destroy(&attr);
    assert(!returnVal);
    if (threadError != 0)
    {
         // Report an error.
    }
}
```

关于 `pthread` 详细的使用请参考这篇文章 [iOS 多线程技术实践之 pthreads（一)](https://kingcos.me/posts/2019/multithreading_techs_in_ios-1/)

### 使用 NSObject 派生线程

所有继承于 `NSObject` 的对象都可以通过 `performSelectorInBackground:withObject:` 来在子线程上执行任务。

这个方法底层实际上是调用的 `NSThread` 类方法 `detachNewThreadSelector:toTarget:withObject:` 来创建线程并执行任务。

### 在 `Cocoa` 应用程序中使用 `POSIX` 线程

虽然在 `Cocoa` 中使用 `NSThread` 是创建线程的主要方式，但是当需要的时候，还是可以使用 `POSIX` 线程的。但是需要遵守以下几点:

* `Protecting the Cocoa Frameworks`: 保护 `Cocoa` 中的框架
    * 对于多线程应用程序，`Cocoa` 框架使用锁和其他形式的内部同步来确保它们的行为正确。但是，为了防止这些锁在单线程情况下降低性能，在应用程序使用 `NSThread` 类生成其第一个新线程之前，`Cocoa` 不会创建这些同步的元素。如果只使用 `POSIX` 线程例程生成线程，`Cocoa` 将不会收到需要知道应用程序现在是多线程的通知。当这种情况发生时，涉及 `Cocoa` 框架的操作可能会破坏应用程序的稳定性或崩溃。
    * 要让 `Cocoa` 知道你打算使用个线程，你只需使用 `NSThread` 类生成一个线程，并让该线程立即退出。你的线程入口点不需要做任何事情。使用 `NSThread` 生成一个线程的行为就足以确保 `Cocoa` 框架所需的锁被放置到位。
    * 如果不确定 `Cocoa` 是否认为你的应用程序是多线程的，则可以使用 `NSThread` 的 `isMultiThreaded` 方法进行检查。

* 混合使用 `Cocoa` 中的锁与 `POSIX` 锁
    * 在同一个应用程序中混合使用 `POSIX` 和 `Cocoa` 锁是安全的。`Cocoa` 锁和条件对象本质上只是 `POSIX` 互斥锁和条件对象的包装。但是，对于给定的锁，必须始终使用相同的接口来创建和操作该锁。换句话说，不能使用 `Cocoa` 的 `NSLock` 对象来操作使用 `pthread_mutex_init` 函数创建的互斥锁对象，反之亦然。

## 配置线程属性

在线程创建之前或者之后，你可能想要配置一些线程相关的属性，具体内容如下:

### 配置线程所占栈空间大小

> 对于你创建的每个新线程，系统都会在进程空间中分配特定数量的内存，以充当该线程的堆栈。
> 
> 堆栈管理堆栈帧，线程内部声明的局部变量就会存于此处。

要设置线程所占用的栈空间大小，只能在线程创建之前指定。

* Cocoa

通过实例化 `NSThread` 对象，然后在调用 `start` 方法之前调用 `setStackSize:` 来设置栈空间大小。

* POSIX

创建 `pthread_attr_t` 结构体对象，然后将其传入 `pthread_attr_setstacksize` 方法来设置栈大小，然后将 `pthread_attr_t` 传入 `pthread_create` 函数来创建 POSIX 线程。

### 配置 TLS

每个线程会维护一个键值对的字典，用来在线程执行过程中存储一些内容，这个字典

`Cocoa` 和 `POSIX` 以不同的方式存储线程字典，因此你不能混合和匹配对这两种技术的调用。 但是，只要您在线程代码中坚持使用一种技术，最终结果应该是相似的。在 `Cocoa` 中，你可以使用 `NSThread` 对象的 `threadDictionary` 方法来获取 `TLS`，你可以在该对象中添加线程所需的任何键。在 `POSIX` 中，使用 `pthread_setspecific` 和`pthread_getspecific` 函数来设置和获取 `TLS`。

### 设置线程的分离状态

通过 `NSThread` 创建的线程默认是分离式的，当应用程序退出时，这种类型的线程也将退出，所以为了执行诸如在应用程序退出时保存数据的任务，需要创建 `joinable` 类型的线程。而目前的方案只有通过 `POSIX` 来实现，通过 `pthread_attr_setdetachstate` 方法来设置 `pthread_t` 的属性来达到非分离式的效果。当然，如果想改变线程的状态，可以通过 `pthread_attr_setdetachstate` 来将线程设置为分离式的。

### 设置线程优先级

你所创建的线程都有与之关联默认的优先级，内核在调度线程的时候，优先级高的线程相比于优先级低的线程更有可能性执行。但是优先级高的线程并不能保证有固定的运行时间，只是更可能被调度而已。

> It is generally a good idea to leave the priorities of your threads at their default values. Increasing the priorities of some threads also increases the likelihood of starvation among lower-priority threads. If your application contains high-priority and low-priority threads that must interact with each other, the starvation of lower-priority threads may block other threads and create performance bottlenecks
> 
> 【译】 通常最好将线程的优先级保留为默认值。增加某些线程的优先级也增加了低优先级线程之间出现饥饿的可能性。如果你的应用程序包含必须相互交互的高优先级和低优先级线程，则低优先级线程的饥饿可能会阻塞其他线程并造成性能瓶颈。
> 
> 这里可以联想到已经不再安全的自旋锁 OSSpinLock，具体内容参见 YYKit 作者的博文 [不再安全的 OSSpinLock](https://blog.ibireme.com/2016/01/16/spinlock_is_unsafe_in_ios/)

如果你确实想修改线程优先级，对于 `Cocoa` 线程，可以使用 `NSThread` 的`setThreadPriority` 类方法设置当前正在运行的线程的优先级。对于 `POSIX` 线程，使用 `pthread_setschedparam` 函数。

## 编写线程入口方法

> 在大多数情况下，OS X 中线程的入口点例程的结构与其他平台上的相同。你可以初始化数据结构，进行一些工作或有选择地设置运行循环，并在线程代码完成后进行清理。根据你的设计，编写输入例程时可能需要采取一些其他步骤。

* 创建一个自动释放池

在 `Objective-C` 框架中链接的应用程序通常必须在其**每个线程中至少创建一个自动释放池**。如果应用程序使用 `ARC`，则自动释放池将捕获从该线程自动释放的所有对象。

下面的代码演示了在 `MRC` 下需要在线程入口方法里面创建一个自动释放池
 
```objectivec
- (void)myThreadMainRoutine
{
    NSAutoreleasePool *pool = [[NSAutoreleasePool alloc] init]; // Top-level pool
 
    // Do thread work here.
 
    [pool release];  // Release the objects in the pool.
}
```

因为顶层的自动释放池直到线程退出之后才会释放对象，所以常驻线程需要创建一些额外的自动释放池来达到经常性的清理效果。例如，使用运行循环的线程可能每次通过该运行循环都会创建并释放一个自动释放池。频繁释放对象可以防止应用程序的内存占用过大，从而导致性能问题。 但是，与任何与性能相关的行为一样，你应该衡量代码的实际性能，并适当调整自动释放池的使用。

* 设置异常处理

如果在你的应用程序里面捕获并抛出了异常，那么在线程的入口方法中也需要做相应的处理，如果有抛出的异常在线程内部没有被捕获到将会导致程序的退出。你可以使用 `C++` 和 `OC` 风格的 `final-try&catch` 代码块来处理。

* 设置 RunLoop

在子线程上执行任务的时候，有两种选择，一种是执行完之后线程会自动退出，一种是希望线程可以一直存活来处理任务。第一种方式不需要额外的操作，而第二种方式则需要 runloop 的配合。而 `iOS` 和 `macOS` 中的每个线程都有对应的 runloop 对象，`app` 的主线程启动之后，其对应的主运行循环也会自动开启，但是对于子线程来说，则需要手动开启。

## 终止线程

退出线程的建议方法是让其正常退出其入口方法。尽管 `Cocoa`，`POSIX` 和`Multiprocessing Services` 提供了直接杀死线程的例程，但是强烈建议不要使用此类例程。直接杀死线程会防止它自己清理掉从而导致线程分配的内存可能会泄漏，线程当前使用的任何其他资源可能无法正确清理，从而在以后产生潜在问题。

如果你预计需要在操作过程中终止线程，则应从一开始就设计线程以响应取消或退出消息。对于长时间运行的操作，这可能意味着要定期停止工作并检查是否收到此消息。如果确实有消息要求线程退出，则该线程将有机会执行所需的清理并正常退出；否则，它可以简单地返回工作并处理下一个数据块。

下面是通过 runloop 以及 threadDictionary 来实现定时检查是否要退出线程的代码：

```objectivec
- (void)threadMainRoutine
{
    // 是否还有更多工作要做
    BOOL moreWorkToDo = YES;
    // 是否要退出线程
    BOOL exitNow = NO;
    // 获取当前线程对应的 runloop 对象
    NSRunLoop* runLoop = [NSRunLoop currentRunLoop];
 
    // 将 exitNow 存入 threadDictionary 中
    NSMutableDictionary* threadDict = [[NSThread currentThread] threadDictionary];
    [threadDict setValue:[NSNumber numberWithBool:exitNow] forKey:@"ThreadShouldExitNow"];
 
    // 设置 runloop 的输入源
    [self myInstallCustomInputSource];
 
    while (moreWorkToDo && !exitNow)
    {
        // 具体要做的工作
        // 完成时改变 moreWorkToDo 的值
 
        // 让 runloop 跑起来，但如果没有要触发的输入源，则立即超时。
        [runLoop runUntilDate:[NSDate date]];
 
        // 检查输入源处理程序是否更改了 exitNow 值
        exitNow = [[threadDict valueForKey:@"ThreadShouldExitNow"] boolValue];
    }
}

```

# 总结

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/iOS%20Thread.png)


# 参考资料

[Threading Programming Guide - Apple](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Multithreading/Introduction/Introduction.html#//apple_ref/doc/uid/10000057i-CH1-SW1)

[Detached vs. Joinable POSIX threads - StackOverflow](https://stackoverflow.com/questions/3756882/detached-vs-joinable-posix-threads)

[c++11中thread join和detach的区别](https://blog.csdn.net/c_base_jin/article/details/79233705)



