![iOS 底层探索 - 类的加载](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119122552.jpg)


# 一、应用加载回顾

上一章我们对应用的加载有了初步的认识，我们知道了

- 系统调用 `exec()` 会我们的应用**映射**到新的地址空间
- 然后通过 `dyld` 进行加载、链接、初始化主程序和主程序所依赖的各种动态库
- 最后在 `initializeMainExecutable` 方法中经过一系列初始化调用 `notifySingle` 函数，该函数会执行一个 `load_images` 的回调
- 然后在 `doModinitFuntions` 函数内部会调用 `__attribute__((constructor))` 的 `c` 函数
- 然后 `dyld` 返回主程序的入口函数，开始进入主程序的 `main` 函数
<!-- more -->
在 `main` 函数执行执行，其实 `dyld` 还会在流程中初始化 `libSystem`，而 `libSystem` 又会去初始化 `libDispatch`，在 `libDispatch` 初始化方法里面又会有一步 `_os_object_init`，在 `_os_object_init` 内部就会调起 `_objc_init`。而对于 `_objc_init` 我们还需要继续探索，因为这里面会进行类的加载等一系列重要的工作。

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119113501.jpg)


# 二、探索 `_objc_init`

首先来到 `libObjc` 源码的 `_objc_init` 方法处，你可以直接添加一个符号断点 `_objc_init` 或者全局搜索关键字来到这里：

```cpp
void _objc_init(void)
{
    static bool initialized = false;
    if (initialized) return;
    initialized = true;
    
    // fixme defer initialization until an objc-using image is found?
    environ_init();
    tls_init();
    static_init();
    lock_init();
    exception_init();

    _dyld_objc_notify_register(&map_images, load_images, unmap_image);
}
```

我们接着进行分析:

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119113515.jpg)


- 判断是否已经初始化了，如果初始化过了，直接返回。

## 2.1 environ_init

接着来到 `environ_init` 方法内部:

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119113523.jpg)


我们可以看到，这里主要是读取影响 `Runtime` 的一些环境变量，如果需要，还可以打印环境变量帮助提示。

我们可以在终端上测试一下，直接输入 `export OBJC_HELP=1`:

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119113531.jpg)


可以看到不同的环境变量对应的内容都被打印出来了。

## 2.2 tls_init

接着来到 `tls_init` 方法内部:

```cpp
void tls_init(void)
{
#if SUPPORT_DIRECT_THREAD_KEYS
    _objc_pthread_key = TLS_DIRECT_KEY;
    pthread_key_init_np(TLS_DIRECT_KEY, &_objc_pthread_destroyspecific);
#else
    _objc_pthread_key = tls_create(&_objc_pthread_destroyspecific);
#endif
}
```

这里执行的是关于线程 `key` 的绑定，比如每线程数据的析构函数。

## 2.3 static_init

接着来到 `static_init` 方法内部:

```cpp
/***********************************************************************
* static_init
* Run C++ static constructor functions.
* libc calls _objc_init() before dyld would call our static constructors, 
* so we have to do it ourselves.
**********************************************************************/
static void static_init()
{
    size_t count;
    auto inits = getLibobjcInitializers(&_mh_dylib_header, &count);
    for (size_t i = 0; i < count; i++) {
        inits[i]();
    }
}
```

这里会运行 `C++` 的静态构造函数，在 `dyld` 调用我们的静态构造函数之前，`libc` 会调用 `_objc_init`，所以这里我们必须自己来做，并且这里只会初始化系统内置的 `C++` 静态构造函数，我们自己代码里面写的并不会在这里初始化。

## 2.4 lock_init

接着来到 `lock_init` 方法内部:

```cpp
void lock_init(void)
{
}
```

我们可以看到，这是一个空的实现。也就是说 `objc` 的锁是完全采用的 `C++` 那一套的锁逻辑。

## 2.5 exception_init

接着来到 `exception_init` 方法内部:

```cpp
/***********************************************************************
* exception_init
* Initialize libobjc's exception handling system.
* Called by map_images().
**********************************************************************/
void exception_init(void)
{
    old_terminate = std::set_terminate(&_objc_terminate);
}
```

这里是初始化 `libobjc` 的异常处理系统，我们程序触发的异常都会来到：

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119113552.jpg)


我们可以看到 `_objc_terminate` 是未处理异常的回调函数，其内部逻辑如下:

- 检查是否是一个活跃的异常
- 如果是活跃的异常，检查是否是 `OC` 抛出的异常
- 如果是 `OC` 抛出的异常，调用 `uncaught_handeler` 回调函数指针
- 如果不是 `OC` 抛出的异常，则继续 `C++` 终止操作

## 2.6 _dyld_objc_notify_register

接下来使我们今天探索的重点了： `_dyld_objc_notify_register` ，我们先看下它的定义:

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119113601.jpg)


> 注意：仅供 `objc` 运行时使用
> 当 `objc` 镜像被**映射（mapped）**、**卸载（unmapped）**和**初始化（initialized）**的时候，注册的回调函数就会被调用。
> 这个方法是 `dlyd` 中声明的，一旦调用该方法，调用结果会作为该函数的参数回传回来。比如，当所有的 `images` 以及 ` section` 为 `objc-image-info` 被加载之后会回调 `mapped` 方法。
> `load` 方法也将在这个方法中被调用。


`_dyld_objc_notify_register`  方法的三个参数 `map_images` 、 `load_images` 、 `unmap_image`  其实都是函数指针：

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119113615.jpg)


这三个函数指针是在 `dyld` 中回调的，我们打开 `dyld` 的源码即可一探究竟，我们直接搜索 `_dyld_objc_notify_register` :

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119113626.jpg)


接着来到 `dyld` 的 `registerObjCNotifiers` 方法内部：

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119113634.jpg)


![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119113641.jpg)


通过上面两张截图的内容说明在 `registerObjCNotifiers` 内部， `libObjc` 传过来的这三个函数指针被 `dyld` 保存在了本地静态变量中。换句话来说，最终函数指针是否能被调用，取决于这三个静态变量：

- `sNotifyObjCMapped` 
- `sNotifyObjCInit` 
- `sNotifyObjCUnmapped` 

我们注意到 `registerObjCNotifiers` 的 `try-catch` 语句中的 `try` 分支注释如下：

> call 'mapped' function with all images mapped so far
> 调用 `mapped` 函数来映射所有的镜像


那么也就是说 `notifyBatchPartial` 里面会进行真正的函数指针的调用，我们进入这个方法内部：

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119113650.jpg)


我们可以看到，在 `notifyBatchPartial` 方法内部，这里的注释:

> tell objc about new images 告诉 `objc` 镜像已经映射完成了


而图中箭头所指的地方正是 `sNotifyObjCMapped` 函数指针真正调用的地方。

弄清楚了三个函数指针是怎么调用的还不够，接下来我们要深入各个函数的内部看里面究竟做了什么样的事情。

# 三、探索 map_images

首先是 `map_images` ，我们来到它的实现:

```cpp
/***********************************************************************
* map_images
* Process the given images which are being mapped in by dyld.
* Calls ABI-agnostic code after taking ABI-specific locks.
*
* Locking: write-locks runtimeLock
**********************************************************************/
void
map_images(unsigned count, const char * const paths[],
           const struct mach_header * const mhdrs[])
{
    mutex_locker_t lock(runtimeLock);
    return map_images_nolock(count, paths, mhdrs);
}C
```

> Process the given images which are being mapped in by dyld.
> Calls ABI-agnostic code after taking ABI-specific locks.


> 处理由 `dyld` 映射的给定镜像
> 取得特定于 `ABI` 的锁后，调用与 `ABI` 无关的代码。


这里会继续往下走到 `map_images_nolock` 

`map_images_nolock` 内部代码十分冗长，我们经过分析之后，前面的工作基本上都是进行镜像文件信息的提取与统计，所以可以定位到最后的 `_read_images` ：

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119113711.jpg)


> 这里进入 `_read_images` 的条件是 `hCount` 大于 0， `hCount` 表示的是 `Mach-O` 中 `header` 的数量


OK，我们的主角登场了， `_read_images` 和 `lookupImpOrForward` 可以说是我们学习 `Runtime` 和 `iOS` 底层里面非常重要的两个概念了， `lookUpImpOrForward` 已经探索过了，剩下的 `_read_images` 我们也不能落下。

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119113721.jpg)


## 3.1 _read_images 定义

> Perform initial processing of the headers in the linked list beginning with headerList. 
> 从 `headerList` 开始，对已经链接了的 `Mach-O` 镜像表中的头部进行初始化处理


我们可以看到，整个 `_read_images` 有接近 400 行代码。我们不妨折叠一下里面的分支代码，然后总览一下：

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119113734.jpg)

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119113742.jpg)

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119113749.jpg)


通过折叠代码，以及日志打印提示信息，我们大致可以将 `_read_images` 分为下面几个流程:

## 3.2 _read_images 具体流程

---


**doneOnce 流程**

我们从第一个分支 `doneOnce` 开始，这个名词顾名思义，只会执行一次：

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119113811.jpg)


- 通过宏 `SUPPORT_NONPOINTER_ISA` 判断当前是否支持开启内存优化的 `isa` 
  - 如果支持，则在某些条件下需要禁用这个优化
- 通过宏 `SUPPORT_INDEXED_ISA` 判断当前是否是将类存储在 `isa` 作为类表索引
  - 如果是的话，再递归遍历所有的 `Mach-O` 的头部，并且判断如果是 `Swift 3.0` 之前的代码，就需要禁用对 `isa` 的内存优化

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119113824.jpg)

- 通过宏 `TARGET_OS_OSX` 判断是否是 `macOS` 执行环境
- 判断 `macOS` 的系统版本，如果小于 `10.11` 则说明 `app` 太陈旧了，需要禁用掉 `non-pointer isa` 
- 然后再遍历所有的 `Mach-O` 的头部，判断如果有 `__DATA__,__objc_rawisa` 段的存在，则禁用掉 `non-pointer isa` ，因为很多新的 `app` 加载老的扩展的时候会需要这样的判断操作。

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119113854.jpg)


> 预先优化过的类不会加入到 `gdb_objc_realized_classes` 这个哈希表中来， `gdb_objc_realized_classes` 哈希表的装载因子为 0.75，这是一个经过验证的效率很高的扩容临界值。


- 加载所有类到类的 `gdb_objc_realized_classes` 表中来<br />

我们查看这个表的定义：

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119113952.jpg)


> // This is a misnomer: gdb_objc_realized_classes is actually a list of 
> // named classes not in the dyld shared cache, whether realized or not.


> 这是一个误称：gdb_objc_realized_classes 表实际上存储的是不在 `dyld` 共享缓存里面的命名类，无论这些类是否实现


除了 `gdb_objc_realized_classes` 表之外，还有一张表 `allocatedClasses` :

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119114000.jpg)


- 通过 `objc_allocateClassPair` 开辟之后的类和元类存储的表（也就是说需要 `alloc` ）

其实 `gdb_objc_realized_classes` 对 `allocatedClasses` 是一种包含的关系，一张是类的总表，一张是已经开辟了内存的类表，

---


**Discover classes 流程**

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119114029.jpg)


> Discover classes. Fix up unresolved future classes. Mark bundle classes.
> 发现类。修正未解析的 `future` 类，标记 `bundle` 类。


- 先通过 `_getObjc2ClassList` 来获取到所有的类，我们可以通过 `MachOView` 来验证：

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119114038.jpg)


- 接着还是遍历所有的 `Mach-O` 的 `header` 部分，然后通过 `mustReadClasses` 来判断哪些条件可以跳过读取类这一步骤
- 读取 `header` 是否是 `Bundle` 
- 读取 `header` 是否开启了 **预优化**
- 遍历 `_getObjc2ClassList` 取出的所有的类
  - 通过 `readClass` 来读取类信息
  - 判断如果不相等并且 `readClass` 结果不为空，则需要重新为类开辟内存

---


**Fix up remapped classes 流程**

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119114058.jpg)


> 修复 重映射类
> 类表和非懒加载类表没有被重映射 (也就是 **_objc_classlist**)
> 由于消息转发，类引用和父类引用会被重映射 (也就是 **_objc_classrefs**)


- 通过 `noClassesRemapped` 方法判断是否有类引用(**_objc_classrefs**)需要进行重映射
  - 如果需要，则遍历 `EACH_HEADER` 
  - 通过 `_getObjc2ClassRefs` 和 `_getObjc2SuperRefs` 取出当前遍历到的 `Mach-O` 的类引用和父类引用，然后调用 `remapClassRef` 进行重映射  

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119114114.jpg)



---


**Fix up @selector references 流程**

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119114341.jpg)


> 修正 `SEL` 引用


- 操作前先加一个 `selLock` 锁
- 然后遍历 `EACH_HEADER` 
  - 如果开启了**预优化**，contiue 到下一个 `Mach-O` 
  - 通过 `_getObjc2SelectorRefs` 拿到所有的 `SEL` 引用
  - 然后对所有的 `SEL` 引用调用 `sel_registerNameNoLock` 进行注册

也就是说这一流程最主要的目的就是注册 `SEL` ，我们注册真正发生的地方: `__sel_registerName` ，这个函数如果大家经常玩 `Runtime` 肯定不会陌生：

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119114358.jpg)


我们简单分析一下 `__sel_registerName` 方法的流程：

- 判断是否要加锁
- 如果 `sel` 为空，则返回一个空的 `SEL` 
- 从 `builtins` 中搜索，看是否已经注册过，如果找到，直接返回结果
- 从 `namedSelectors` 哈希表中查询，找到了就返回结果
- 如果 `namedSelectors` 未初始化，则创建一下这个哈希表
- 如果上面的流程都没有找到，则需要调用 `sel_alloc` 来创建一下 `SEL` ，然后把新创建的 `SEL` 插入哈希表中进行缓存的填充

---


**Fix up old objc_msgSend_fixup call sites 流程**

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119114428.jpg)


> 修正旧的 `objc_msgSend_fixup` 调用

这个流程的执行前提是 `FIXUP` 被开启。

- 还是老套路，遍历 `EACH_HEADER` 
  - 通过 `_getObjc2MessageRefs` 方法来获取当前遍历到的 `Mach-O` 镜像的所有消息引用
  - 然后遍历这些消息引用，然后调用 `fixupMessageRef` 进行修正

---


**Discover protocols 流程**

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119114506.jpg)


> 发现协议，并修正协议引用

---


**Fix up @protocol references 流程**

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119114534.jpg)



> 对所有的协议做重映射

---

**Realize non-lazy classes 流程**

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119114757.jpg)


> 初始化**非懒加载类( **`**+load**` 方法和静态实例**)**


---

**Realize newly-resolved future classes 流程**

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119114821.jpg)


> 初始化新解析出来的 `future` 类


---


**Discover categories 流程**

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119114844.jpg)


> **处理所有的分类，包括类和元类**


---

到这里， `_read_images` 的流程就分析完毕，我们可以新建一个文件来去掉一些干扰的信息，只保留核心的逻辑，这样从宏观的角度来分析更直观:

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119114851.jpg)


> Q & A 环节
> Q： `dyld` 主要逻辑是加载库，也就是镜像文件，但是加载完是怎么读取的呢？
> A： `_read_images` 是真正读取的地方
> 
> Q: `SEL` 方法编号何时加载？
> A: `_read_images`


## 3.3 read_class 分析

我们探索了 `_read_images` 方法的流程，接下来让我们把目光放到本文的主题 - **类的加载**<br />既然是类的加载，那么我们在前面所探索的类的结构中出现的内容都会一一重现。<br />所以我们不妨直接进行断点调试，让我们略过其它干扰信息，聚焦于类的加载。

- 根据上一小节我们探索的结果， `doneOnce` 流程中会创建两个哈希表，并没有涉及到类的加载，所以我们跳过
- 我们来到第二个流程 - **类处理**

**我们在下图所示的位置处打上断点：**

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119114933.jpg)

如上图所示，从 `classList` 中取出的 `cls` 只是一个内存地址，我们尝试通过 `LLDB` 打印 `cls` 的 `class_rw_t` :

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119114946.jpg)


可以看到 `cls` 的属性、方法、协议以及类名都为空，说明这里类并没有被真正加载完成，我们接着聚焦到 `read_class` 函数上面，我们进入其内部实现，我们大致浏览之后会定位到如下图所示的代码：

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119115011.jpg)


看起来类的信息在这里完成了加载，那么为了验证我们的猜想，直接断点调试一下但发现断点根本走不进来，原因在于这里的判断语句 

```cpp
if (Class newCls = popFutureNamedClass(mangledName))
```

判断当前传入的类的类名是否有 `future` 类的实现，但是我们刚才已经打印了，类名是空的，所以肯定不会执行这里。我们接着往下走：

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119115022.jpg)


- addNamedClass 内部其实是将 `cls`  插入到 `gdb_objc_realized_classes` 表 

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119115035.jpg)


- addclassTableEntry 内部是将 `cls` 插入到 `allocatedClasses` 表

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119115044.jpg)


分析完 `read_class` ，我们回到 `_read_images` 方法

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119115054.jpg)


我们可以看到 `read_class` 返回的 `newCls` 会进行一个判断，判断与传入 `read_class` 之前的 `cls` 是否相等，而在 `read_class` 内部只有一个地方对类的内容进行了改动，但是我们刚才测试了是进不去的，所以这个 `if` 里面的内容我们可以略过，也就是说 `resolvedFutureClasses` 的内容我们都可以暂时略过。

总结一下 `readClass` ：

- 判断是不是要后期处理的类
  - 如果是的话，就取出后期处理的类，读取这个类的 `data()` 类设置 `ro/rw` 
- addNamedClass 插入总表
- addClassTableEntry 插入已开辟内存的类的表  

## 3.4 realizeClassWithoutSwift 分析

通过分析 `read_class` ，我们可以得知，类已经被注册到两个哈希表中去了，那么现在一切时机都已经成熟了。但是我们还是要略过像 `Fix up remapped classes` 、 `Fix up @selector references` 、 `fix up old objc_msgSend_fixup call sites` 、 `Discover protocols. Fix up protocol refs` 、 `Fix up @protocol references` ，因为我们的重点是类的加载，我们最终来到了 `Realize non-lazy classes (for +load methods and static instances)` ，略去无关信息之后，我们可以看到我们的<br />主角 `realizeClassWithoutSwift` 闪亮登场了：

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119115106.jpg)


从方法的名称以及方法注释我们可以知道， `realizeClassWithoutSwift` 是进行类的第一次初始化操作，包括分配读写数据也就是我们常说的 `rw` ，但是并不会进行任何的 `Swift` 端初始化。我们直接聚焦下面的代码：

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119115150.jpg)


- 通过 `calloc` 开辟内存空间，返回一个新的 `rw` 
- 把 `cls` 取出来的 `ro` 赋值给这个 `rw` 
- 将 `rw` 设置到 `cls` 身上

那么是不是说在这里 `rw` 就有值了呢，我们 `LLDB` 打印大法走起:

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119115158.jpg)


可以清楚地看到，此时 `rw` 还是为空，说明这里只是对 `rw` 进行了初始化，但是方法、属性、协议这些都没有被添加上。

我们接着往下走:

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119115206.jpg)


这里可以看到父类和元类都会递归调用 `realizeClassWithoutSwift` 来初始化各自的 `rw` 。为什么在类的加载操作里面要去加载类和元类呢？回忆一下类的结构，答案很简单，要保证 `superclass` 和 `isa` 的完整性，也就是保证类的完整性，

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119115218.jpg)


上面的截图就是最好的证明，初始化完毕的父类和元类被赋值到了类的 `superclass` 和 `isa` 上面。

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119115226.jpg)


接着往下走可以看到，不光要把父类关联到类上面，还要让父类知道子类的存在。

最后一行代码是 `methodizeClass(cls)` ，注释显示的是 `attach categories` ，附加分类到类？我们进入其内部实现一探究竟。

在探索 `methodizeClass` 前，我们先总结一下 `realizeClassWithoutSwift` :

- 读取 `class`  的 `data()` 
- `ro/rw`  赋值
- 父类和元类实现
  - supercls = realizeClassWithoutSwift(remapClass(cls->superclass))
  - metacls = realizeClassWithoutSwift(remapClass(cls->ISA()))
- 父类和元类归属关系
  - cls->superclass = supercls
  - cls->initClassIsa(metacls)
- 将当前类链接到其父类的子类列表 addSubclass(supercls, cls)

## 3.5 methodizeClass 分析

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119115239.jpg)


> 对类的方法列表、协议列表和属性列表进行修正
> 附加 `category`  到类上面来


我们直接往下面走：

```cpp
    // Install methods and properties that the class implements itself.
    method_list_t *list = ro->baseMethods();
    if (list) {
        prepareMethodLists(cls, &list, 1, YES, isBundleClass(cls));
        rw->methods.attachLists(&list, 1);
    }
```

- 从 `ro` 中取出**方法列表**附加到 `rw` 上

```cpp
    property_list_t *proplist = ro->baseProperties;
    if (proplist) {
        rw->properties.attachLists(&proplist, 1);
    }
```

- 从 `ro` 中取出**属性列表**附加到 `rw` 上

```cpp
    protocol_list_t *protolist = ro->baseProtocols;
    if (protolist) {
        rw->protocols.attachLists(&protolist, 1);
    }
```

- 从 `ro` 中取出**协议列表**附加到 `rw` 上

```cpp
    category_list *cats = unattachedCategoriesForClass(cls, true /*realizing*/);
    attachCategories(cls, cats, false /*don't flush caches*/);
```

- 从 `cls` 中取出未附加的分类进行附加操作

我们可以看到，这里有一个操作叫 `attachLists` ，为什么方法、属性、协议都能调用这个方法呢？

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119115313.jpg)


![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119115322.jpg)


![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119115327.jpg)


我们可以看到，方法、属性、协议的数据结构都是一个二维数组，我们深入 `attachLists` 方法内部实现:

```cpp
    void attachLists(List* const * addedLists, uint32_t addedCount) {
        if (addedCount == 0) return;

        if (hasArray()) {
            // many lists -> many lists
            uint32_t oldCount = array()->count;//10
            uint32_t newCount = oldCount + addedCount;//4
            setArray((array_t *)realloc(array(), array_t::byteSize(newCount)));
            array()->count = newCount;// 10+4
   
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

- 判断要添加的数量是否为 0，如果为 0，直接返回
- 判断当前调用 `attachLists` 的 `list_array_tt` 二维数组有多个一维数组
  - 如果是，说明是**多对多**的关系
  - 这里会通过 `realloc` 对容器进行重新分配，大小为原来的大小加上新增的大小
  - 然后通过 `memmove` 把原来的数据移动到容器的末尾
  - 最后把新的数据拷贝到容器的起始位置
- 如果调用 `attachLists` 的 `list_array_tt` 二维数组为空且新增大小数目为 1，则直接取 `addedList` 的第一个 `list` 返回
- 如果当前调用 `attachLists` 的 `list_array_tt` 二维数组只有一个一维数组
  - 如果是，说明是**一对多**的关系
  - 这里会通过 `realloc` 对容器进行重新分配，大小为原来的大小加上新增的大小
  - 因为原来只有一个一维数组，所以直接赋值到新 `Array` 的最后一个位置
  - 然后把新数据拷贝到容器的起始位置

# 四、探索 load_images

我们接着探索 `_dyld_objc_notify_register` 的第二个参数 `load_images` ，这个函数指针是在什么时候调用的呢，同样的，我们接着在 `dyld` 源码中搜索对应的函数指针 `sNotifyObjCInit` :

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119115339.jpg)


可以看到，在 `notifySingle` 方法内部， `sNotifyObjCInit` 函数指针被调用了。根据我们上一篇文章探索 `dyld` 底层可以知道， `_load_images` 应该是对于每一个加载进来的 `Mach-O` 镜像都会递归调用一次。

我们来到 `libObjc` 源码中 `load_images` 的定义处:

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119115441.jpg)


> 处理由 `dyld` 映射的给定镜像中的 `+load`  方法


- 判断是否有 `load` 方法，如果没有，直接返回
- 搜索 `load` 方法，具体实现通过 `prepare_load_methods` 
- 调用 `load` 方法，具体实现通过 `call_load_methods` 

## 4.1 prepare_load_methods 分析

从这个方法名称，我们猜测这里应该做的是 `load` 方法的一些预处理工作，让我们来到源码进行分析：

```cpp
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
        if (cls->isSwiftStable()) {
            _objc_fatal("Swift class extensions and categories on Swift "
                        "classes are not allowed to have +load methods");
        }
        realizeClassWithoutSwift(cls);
        assert(cls->ISA()->isRealized());
        add_category_to_loadable_list(cat);
    }
}

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

/***********************************************************************
* add_class_to_loadable_list
* Class cls has just become connected. Schedule it for +load if
* it implements a +load method.
**********************************************************************/
void add_class_to_loadable_list(Class cls)
{
    IMP method;

    loadMethodLock.assertLocked();

    method = cls->getLoadMethod();
    if (!method) return;  // Don't bother if cls has no +load method
    
    if (PrintLoading) {
        _objc_inform("LOAD: class '%s' scheduled for +load", 
                     cls->nameForLogging());
    }
    
    if (loadable_classes_used == loadable_classes_allocated) {
        loadable_classes_allocated = loadable_classes_allocated*2 + 16;
        loadable_classes = (struct loadable_class *)
            realloc(loadable_classes,
                              loadable_classes_allocated *
                              sizeof(struct loadable_class));
    }
    
    loadable_classes[loadable_classes_used].cls = cls;
    loadable_classes[loadable_classes_used].method = method;
    loadable_classes_used++;
}
```

- 首先通过 `_getObjc2NonlazyClassList` 获取所有已经加载进去的类列表
- 然后通过 `schedule_class_load` 遍历这些类
  - 递归调用遍历父类的 `load` 方法，确保父类的 `load` 方法顺序排在子类的前面
  - 通过 `add_class_to_loadable_list` , 把类的 `load` 方法存在 `loadable_classes` 里面
  - ![image.png](https://upload-images.jianshu.io/upload_images/95471-11cae8927cfdf166?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- 完成 `schedule_class_load` 之后，通过 `_getObjc2NonlazyCategoryList` 取出所有分类数据
- 然后遍历这些分类
  - 通过 `realizeClassWithoutSwift` 来防止类没有初始化，如果已经初始化了则不影响
  - 通过 `add_category_to_loadable_list` ，加载分类中的 `load` 方法到 `loadable_categories` 里面
  - ![image.png](https://upload-images.jianshu.io/upload_images/95471-74f44fde4b425ea7?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 4.2 call_load_methods 分析

<br />通过名称我们可以知道 `call_load_methods` 应该就是 `load` 方法被调用的地方了。我们直接看源码：<br />

```cpp
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


> call_load_methods
> 调用类和类别中所有未决的 `+load` 方法
> 类里面 `+load` 方法是父类优先调用的
> 而在父类的 `+load` 之后才会调用分类的 `+load` 方法


- 通过 `objc_autoreleasePoolPush` 压栈一个自动释放池
- `do-while` 循环开始
  - 循环调用类的 `+load` 方法直到找不到为止
  - 调用一次分类中的 `+load` 方法
- 通过 `objc_autoreleasePoolPop` 出栈一个自动释放池

# 五、总结

至此， `_objc_init` 和 `_dyld_objc_notify_register` 我们就分析完了，我们对类的加载有了更细致的认知。 `iOS` 底层有时候探索起来确实很枯燥，但是如果能找到高效的方法以及明确自己的所探索的方向，会让自己从宏观上重新审视这门技术。是的，技术只是工具，我们不能被技术所绑架，我们要做到有的放矢的去探索，这样才能事半功倍。


