![iOS 底层探索 - 应用加载](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119113149.jpg)


`App` 从被用户在主屏幕上点击之后就开启了它的生命周期，那么在这之中，究竟发生了什么呢?让我们从 `App` 启动开始探索。在探索之前，我们需要熟悉一些前导知识点。

# 一、前导知识

以下参考自 `WWDC 2016 Optimizing App Startup Time` ：

## 1.1 Mach-O

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119104621.jpg)


> Mach-O is a bunch of file types for different run time executables.
> `Mach-O` 是 `iOS` 系统不同运行时期**可执行的文件**的文件类型统称。


维基百科上关于 `Mach-O` 的描述：

> Mach-O 是 Mach object 文件格式的缩写，它是一种用于记录可执行文件、对象代码、共享库、动态加载代码和内存转储的文件格式。作为 a.out 格式的替代品，Mach-O 提供了更好的扩展性，并提升了符号表中信息的访问速度。
> 大多数基于 Mach 内核的操作系统都使用 Mach-O。NeXTSTEP、OS X 和 iOS 是使用这种格式作为本地可执行文件、库和对象代码的例子。

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119104635.jpg)


`Mach-O` 有三种文件类型: `Executable`、`Dylib`、`Bundle`

- `Executable` 类型

> So the first executable, that's the main binary in an app, it's also the main binary in an app extension.
> `executable` 是 `app` 的二进制主文件，同时也是 `app extension` 的二进制主文件


我们一般可以在 `Xcode` 项目中的 `Products` 文件夹中找到它：

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119104647.jpg)

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119104653.jpg)

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119104659.jpg)


如上图箭头所示，`App加载流程` 就是我们 `App` 的二进制主文件。

- `Dylib` 类型

> A dylib is a dynamic library, on other platforms meet, you may know those as DSOs or DLLs.
> `dylib` 是动态库，在其他平台也叫 `DSO` 或者 `DLL`。


对于接触 `iOS` 开发比较早的同学，可能知道我们在 `Xcode 7` 之前添加一些比如 `sqlite` 的库的时候，其后缀名为 `dylib`，而 `Xcode 7` 之后后缀名都改成了 `tbd`。

这里引用 [StackoverFlow](https://stackoverflow.com/questions/31450690/why-xcode-7-shows-tbd-instead-of-dylib) 上的一篇回答。

> So it appears that the .dylib file is the actual library of binary code that your project is using and is located in the /usr/lib/ directory on the user's device. The .tbd file, on the other hand, is just a text file that is included in your project and serves as a link to the required .dylib binary. Since this text file is much smaller than the binary library, it makes the SDK's download size smaller.
> 看起来 `.dylib` 文件是项目中真正使用到的二进制库文件，它位于用户设备上的 `/usr/lib` 目录下。而 `.tbd` 文件，只是位于你项目中的一个文本文件，它扮演的是链接到真正的 `.dylib` 二进制文件的角色。因为文本文件的大小远远小于二进制文件的大小，所以让 `Xcode 的`SDK` 的下载大小更小。


这里再插一句，那么有动态库，肯定就有静态库，它们的区别是什么呢？

我们先梳理一下整个的编译过程。

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119104715.jpg)


当然，这个过程中间其实还设计到编译器前端的 `词法分析`、`语法分析`、`语义分析`、`优化` 等流程，我们在后面探索 `LLVM` 和 `Clang` 的时候会详细介绍。

回到刚才的话题，静态库和动态库的区别：

> Static frameworks are linked at **compile time**. Dynamic frameworks are linked **at runtime**.


静态库和动态库都是编译好的二进制文件，只是用法不同。那为什么要分动态和静态库呢？

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119104729.jpg)

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119104736.jpg)


通过上面两幅图我们可以知道：

- 静态库表现为：在链接阶段会将汇编生成的目标文件与引用的库一起链接打包进可执行文件中。
- 动态库表现为：程序编译并不会链接到目标代码中，在程序可执行文件里面会保留对动态库的引用。其中，动态库分为动态链接库和动态加载库。
  - **动态链接库**：在没有被加载到内存的前提下，当可执行文件被加载，动态库也随着被加载到内存中。在 `Linked Framework and Libraries` 设置的一些 `share libraries`。【随着程序启动而启动】
  - **动态加载库**：当需要的时候再使用 `dlopen` 等通过代码或者命令的方式来加载。【在程序启动之后】
- `Bundle` 类型

> Now a bundle's a special kind of dylib that you cannot link against, all you can do is load it at run time by an dlopen and that's used on a Mac OS for plug-ins.
> 现阶段 `Bundle` 是一种特殊类型的 `dylib`，你是无法对其进行链接的。你所能做的是在 `Runtime` 运行时去通过 `dlopen` 来加载它，它可以在 `macOS` 上用于插件。


- `Image` 和 `Framework`

> Image refers to any of these three types.
> 镜像文件包含了上述的三种文件类型


> a framework is a dylib with a special directory structure around it to holds files needed by that dylib.
> 有很多东西都叫做 `Framework`，但在本文中，`Framework` 指的是一个 `dylib`，它周围有一个特殊的目录结构来保存该 `dylib` 所需的文件。


### 1.1.1 Mach-O 结构分析

#### 1.1.1.1 segment 段

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119104753.jpg)


`Mach-O` 镜像文件是由 `segments` 段组成的。

- 段的名称为大写格式<br />
所有的段都是 `page size` 的倍数。
- arm64 上段大小为 `16` 字节
- 其它架构为 `4` 字节

这里再普及一下**虚拟内存**和**内存页**的知识：

> 具有 `VM` 机制的操作系统，会对每个运行的进程创建一个逻辑地址空间 `logical address space` 或者叫虚拟地址空间 `virtual address space`；该空间的大小由操作系统位数决定：`32` 位的操作系统，其逻辑地址空间的大小为 `4GB`，64位的操作系统为 `18 exabyes`（其计算方式是 `2^32` || `2^64`）。

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119104805.jpg)


> 虚拟地址空间(或者逻辑地址空间)会被分为相同大小的块，这些块被称为内存页（page）。计算机处理器和它的内存管理单元（MMU - memory management uinit）维护着一张将程序的逻辑地址空间映射到物理地址上的分页表 `page table`。


> 在 `masOS` 和早版本的 `iOS` 中，分页的大小为 `4kB`。在之后的基于 `A7` 和 `A8` 的系统中，虚拟内存（`64` 位的地址空间）地址空间的分页大小变为了 `16KB`，而物理RAM上的内存分页大小仍然维持在 `4KB`；基于A9及之后的系统，虚拟内存和物理内存的分页都是`16KB`。


#### 1.1.1.2 section

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119104814.jpg)


在 `segment` 段内部还有许多的 `section` 区。`section` 名称为小写格式。

> But sections are really just a subrange of a segment, they don't have any of the constraints of being page size, but they are non-overlapping.
> 但是 `sections` 节实际上只是一个 `segment` 段的子范围，它们没有页面大小的任何限制，但是它们是不重叠的。


通过 `MachOView` 工具查看 `app` 的二进制可执行文件可以查看到:

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119104825.jpg)

#### 1.1.1.3 常见的 `segments`

- `__TEXT`：代码段，包括头文件、代码和常量。只读不可修改

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119104838.jpg)


- `__DATA`：数据段，包括全局变量, 静态变量等。可读可写。

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119104849.jpg)

- `__LINKEDIT`：如何加载程序, 包含了方法和变量的元数据（位置，偏移量），以及代码签名等信息。只读不可修改。

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119104902.jpg)


### 1.1.2 Mach-O Universal Files

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119104912.jpg)


`Mach-O` 通用文件，将多种架构的 `Mach-O` 文件合并而成。它通过 `header` 来记录不同架构在文件中的偏移量，`segement` 占多个分页，`header`占一页的空间。可能有人会觉得 `header` 单独占一页会浪费空间，但这有利于虚拟内存的实现。

## 1.2 虚拟内存

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119105004.jpg)

虚拟内存是一层**间接寻址**。

虚拟内存解决的是管理所有进程使用**物理 RAM** 的问题。通过添加间接层来让每个进程使用**逻辑地址空间**，它可以映射到 RAM 上的某个物理页上。这种映射不是一对一的，逻辑地址可能映射不到 RAM 上，也可能有多个逻辑地址映射到同一个物理 RAM 上。

- 针对第一种情况，当进程要存储逻辑地址内容时会触发 `page fault`。
- 而第二种情况就是多进程共享内存。
- 对于文件可以不用一次性读入整个文件，可以使用分页映射 `mmap()` 的方式读取。也就是把文件**某个片段**映射到进程逻辑内存的**某个页**上。当某个想要读取的页没有在内存中，就会触发 `page fault`，内核只会读入那一页，实现文件的**懒加载**。也就是说 `Mach-O` 文件中的 `__TEXT` 段可以映射到多个进程，并可以懒加载，且进程之间**共享内存**。
- `__DATA` 段是可读写的。这里使用到了 `Copy-On-Write` 技术，简称 `COW`。也就是多个进程共享一页内存空间时，一旦有进程要做写操作，它会先将这页内存内容**复制**一份出来，然后重新映射逻辑地址到新的 `RAM` 页上。也就是这个进程自己拥有了那页内存的拷贝。这就涉及到了 `clean/dirty page` 的概念。`dirty page` 含有进程自己的信息，而 `clean page` 可以被内核重新生成（重新读磁盘）。所以 `dirty page` 的代价大于 `clean page`。

## 1.3 多进程加载 Mach-O 镜像

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119105025.jpg)


- 所以在多个进程加载 `Mach-O` 镜像时 `__TEXT` 和 `__LINKEDIT` 因为只读，都是可以共享内存的，读取速度就会很快。
- 而 `__DATA` 因为可读写，就有可能会产生 `dirty page`，如果检测到有 `clean page` 就可以直接使用，反之就需要重新读取 `DATA page`。一旦产生了 `dirty page`，当 `dyld` 执行结束后，`__LINKEDIT` 需要通知内核当前页面不再需要了，当别人需要的使用时候就可以重新 `clean` 这些页面。

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119105037.jpg)


## 1.4 ASLR

`ASLR` (Address Space Layout Randomization) 地址空间布局随机化，镜像会在随机的地址上加载。

## 1.5 Code Signing

可能我们认为 `Xcode` 会把整个文件都做加密 `hash` 并用做数字签名。其实为了在运行时验证 `Mach-O` 文件的签名，并不是每次重复读入整个文件，而是把每页内容都生成一个单独的加密散列值，并存储在 `__LINKEDIT` 中。这使得文件每页的内容都能及时被校验确并保不被篡改。

## 1.6 `exec()`

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119105102.jpg)

> Exec is a system call. When you trap into the kernel, you basically say I want to replace this process with this new program.


> `exec()` 是一个系统调用。系统内核把应用映射到新的地址空间，且每次起始位置都是随机的（因为使用 `ASLR`）。并将起始位置到 `0x000000` 这段范围的进程权限都标记为不可读写不可执行。如果是 `32` 位进程，这个范围至少是 `4KB`；对于 `64` 位进程则至少是 `4GB` 。`NULL` 指针引用和指针截断误差都是会被它捕获。这个范围也叫做 `PAGEZERO`。

## 1.7 dyld

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119111212.jpg)


> Unix 的前二十年很安逸，因为那时还没有发明动态链接库。有了动态链接库后，一个用于加载链接库的帮助程序被创建。在苹果的平台里是 `dyld`，其他 `Unix` 系统也有 `ld.so`。 当内核完成映射进程的工作后会将名字为 `dyld` 的 `Mach-O` 文件映射到进程中的随机地址，它将 `PC` 寄存器设为 `dyld` 的地址并运行。`dyld` 在应用进程中运行的工作是加载应用依赖的所有动态链接库，准备好运行所需的一切，它拥有的权限跟应用一样。

## 1.8 dyld 流程

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119111227.jpg)


- Load dylibs

> 从主执行文件的 `header` 获取到需要加载的所依赖动态库列表，而 `header` 早就被内核映射过。然后它需要找到每个 `dylib`，然后打开文件读取文件起始位置，确保它是 `Mach-O` 文件。接着会找到代码签名并将其注册到内核。然后在 `dylib` 文件的每个 `segment` 上调用 `mmap()`。应用所依赖的 `dylib` 文件可能会再依赖其他 `dylib`，所以 `dyld` 所需要加载的是动态库列表一个递归依赖的集合。一般应用会加载 `100` 到 `400` 个 `dylib` 文件，但大部分都是系统 `dylib`，它们会被预先计算和缓存起来，加载速度很快。


- Fix-ups

> 在加载所有的动态链接库之后，它们只是处在相互独立的状态，需要将它们**绑定**起来，这就是 `Fix-ups`。代码签名使得我们不能修改指令，那样就不能让一个 `dylib` 的调用另一个 `dylib`。这时需要加很多间接层。
> 现代 `code-gen` 被叫做动态 **PIC（Position Independent Code）**，意味着代码可以被加载到间接的地址上。当调用发生时，`code-gen` 实际上会在 `__DATA` 段中创建一个指向被调用者的指针，然后加载指针并跳转过去。所以 `dyld` 做的事情就是修正（`fix-up`）指针和数据。`Fix-up` 有两种类型，`rebasing` 和 `binding`。


- Rebasing 和 Binding

> Rebasing：在镜像内部**调整**指针的指向
> Binding：将指针**指向镜像外部**的内容


`dyld` 的时间线由上图可知为：

Load dylibs -> Rebase -> Bind -> ObjC -> Initializers

## 1.9 dyld2 && dyld3

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119111245.jpg)


在 `iOS 13` 之前，所有的第三方 `App` 都是通过 `dyld 2` 来启动 `App` 的，主要过程如下：

- 解析 `Mach-O` 的 `Header` 和 `Load Commands`，找到其依赖的库，并递归找到所有依赖的库
- 加载 `Mach-O` 文件
- 进行符号查找
- 绑定和变基
- 运行初始化程序

`dyld3` 被分为了**三个组件**：

- 一个进程外的 `MachO` 解析器
  - 预先处理了所有可能影响启动速度的 `search path`、`@rpaths` 和环境变量
  - 然后分析 `Mach-O` 的 `Header` 和依赖，并完成了所有符号查找的工作
  - 最后将这些结果创建成了一个启动闭包
  - 这是一个普通的 `daemon` 进程，可以使用通常的测试架构
- 一个进程内的引擎，用来运行启动闭包
  - 这部分在进程中处理
  - 验证启动闭包的安全性，然后映射到 `dylib` 之中，再跳转到 `main` 函数
  - 不需要解析 `Mach-O` 的 `Header` 和依赖，也不需要符号查找。
- 一个启动闭包缓存服务
  - 系统 `App` 的启动闭包被构建在一个 `Shared Cache` 中， 我们甚至不需要打开一个单独的文件
  - 对于第三方的 `App`，我们会在 `App` 安装或者升级的时候构建这个启动闭包。
  - 在 `iOS`、`tvOS`、`watchOS`中，这这一切都是 `App` 启动之前完成的。在 `macOS` 上，由于有 `Side Load App`，进程内引擎会在首次启动的时候启动一个 `daemon` 进程，之后就可以使用启动闭包启动了。

dyld 3 把很多耗时的查找、计算和 `I/O` 的事前都预先处理好了，这使得启动速度有了很大的提升。

好了，先导知识就总结到这里，接下来让我们调整呼吸进入下一章~

# 二、App 加载分析

我们在探索 `iOS` 底层的时候，对于对象、类、方法有了一定的认知哦，接下来我们就一起来探索一下应用是怎么加载的。

我们直接新建一个 `Single View App` 的项目，然后在 `main.m` 中打一个断点:

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119111300.jpg)


然后我们可以看到在 `main` 方法执行前有一步 `start`，而这一流程是由 `libdyld.dylib` 这个动态库来执行的。

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119111305.jpg)


这个现象说明了什么呢？说明我们的 `app` 在 `main` 函数执行之前其实还通过 `dyld` 做了很多事情。那为了搞清楚具体的流程，我们不妨从 [Apple OpenSource](https://opensource.apple.com/tarballs/) 上下载 `dyld` 的源码来进行探索。

我们选择最新的 `655.1.1` 版本：

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119111329.jpg)

# 三、`dyld` 源码分析

面对 `dyld` 的源码，我们不可能一行一行的去分析。我们不妨在刚才创建的项目中断点一下 `load` 方法，看下调用堆栈:

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119111343.jpg)


这一次我们发现，`load` 方法的调用要早于 `main` 函数的调用，其次，我们得到了一个非常有价值的线索: `_dyld_start`。

## 3.1 _dyld_start

我们直接在 `dyld 655.1.1` 中全局搜索这个 `_dyld_start`，我们可以来到 `dyldStartup.s` 这个汇编文件，然后我们聚焦于 `arm64` 架构下的汇编代码:

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119111418.jpg)


对于这里的汇编代码，我们肯定也没必要逐行分析，我们直接定位到 `bl` 语句后面(`bl` 在汇编层面是跳转的意思)：

```assembly
bl	__ZN13dyldbootstrap5startEPK12macho_headeriPPKclS2_Pm
```

我们可以看到这里有一行注释:

```
// call dyldbootstrap::start(app_mh, argc, argv, slide, dyld_mh, &startGlue)
```

这行注释的意思是调用位于 `dyldbootstrap` 命名空间下的 `start` 方法，我们继续搜索一下这个 `start` 方法，结果位于 `dyldInitialization.cpp` 文件(从文件名我们可以看出该文件主要是用来初始化 `dyld`)，这里查找 `start` 的时候可能会有很多结果，我们其实可以先搜索命名空间，再搜索 `start` 方法。

## 3.2 dyldbootstrap::start

`start` 方法源码如下：

```cpp
//
//  This is code to bootstrap dyld.  This work in normally done for a program by dyld and crt.
//  In dyld we have to do this manually.
//
uintptr_t start(const struct macho_header* appsMachHeader, int argc, const char* argv[], 
				intptr_t slide, const struct macho_header* dyldsMachHeader,
				uintptr_t* startGlue)
{
	// if kernel had to slide dyld, we need to fix up load sensitive locations
	// we have to do this before using any global variables
    slide = slideOfMainExecutable(dyldsMachHeader);
    bool shouldRebase = slide != 0;
#if __has_feature(ptrauth_calls)
    shouldRebase = true;
#endif
    if ( shouldRebase ) {
        rebaseDyld(dyldsMachHeader, slide);
    }

	// allow dyld to use mach messaging
	mach_init();

	// kernel sets up env pointer to be just past end of agv array
	const char** envp = &argv[argc+1];
	
	// kernel sets up apple pointer to be just past end of envp array
	const char** apple = envp;
	while(*apple != NULL) { ++apple; }
	++apple;

	// set up random value for stack canary
	__guard_setup(apple);

#if DYLD_INITIALIZER_SUPPORT
	// run all C++ initializers inside dyld
	runDyldInitializers(dyldsMachHeader, slide, argc, argv, envp, apple);
#endif

	// now that we are done bootstrapping dyld, call dyld's main
	uintptr_t appsSlide = slideOfMainExecutable(appsMachHeader);
	return dyld::_main(appsMachHeader, appsSlide, argc, argv, envp, apple, startGlue);
}
```

我们刚才探索到了 `start` 方法，具体流程如下:

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119111440.jpg)


- 根据 `dyld` 的 `Mach-O` 文件的 `header` 判断是否需要对 `dyld` 这个 `Mach-O` 进行 `rebase` 操作

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119111448.jpg)


- 初始化 `mach`，使得 `dyld` 可以进行 `mach` 通讯。

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119111504.jpg)


- 内核将 `env` 指针设置为刚好超出 `agv` 数组的末尾；内核将 `apple` 指针设置为刚好超出 `envp` 数组的末尾

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119111511.jpg)

- 栈溢出保护

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119111519.jpg)


- 读取 `app` 主二进制文件 `Mach-O` 的 `header` 来得到偏移量 `appSlide`，然后调用 `dyld` 命名空间下的 `_main` 方法。

## 3.3 dyldbootstrap::_main

我们通过搜索来到 `dyld.cpp` 文件下的 `_main` 方法：

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119111542.jpg)


`_main方法` 官方的注释如下：

> `dyld` 的入口。内核加载了 `dyld` 然后跳转到 `__dyld_start` 来设置一些寄存器的值然后调用到了这个方法。
> 返回 `__dyld_start` 所跳转到的目标程序的 `main` 函数地址。


我们乍一看，这个方法有四五百行，所以我们不能老老实实的一行一行来看，这样太累了。我们应该着重于有注释的地方。

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119111551.jpg)


- 我们首先可以看到这里是从环境变量中获取主要可执行文件的 `cdHash` 值。这个哈希值 `mainExecutableCDHash` 在后面用来校验 `dyld3` 的启动闭包。

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119111601.jpg)


- 上图代码作用是追踪 `dyld` 的加载。然后判断当前是否为模拟器环境，如果不是模拟器，则追踪主二进制可执行文件的加载。

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119111608.jpg)


- 显示宏定义判断是否为 `macOS` 执行环境，如果是则判断 `DYLD_ROOT_PATH` 环境变量是否存在，如果存在，然后判断模拟器是否有自己的 `dyld`，如果有就使用，如果没有，则返回错误信息。

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119111647.jpg)


- 打印日志：`dyld 启动开始`
- 根据传入 `dyldbootstrap::_main` 方法的参数来设置上下文
- 拾取指向 `exec` 路径的指针
- 从 `dyl` d移除临时 `apple [0]` 过渡代码
- 判断 `exec` 路径是否为绝对路径，如果为相对路径，使用 `cwd` 转化为绝对路径
- 为了后续的日志打印从 `exec` 路径中取出进程的名称 (`strrchr` 函数是获取第二个参数出现的最后的一个位置，然后返回从这个位置开始到结束的内容)
- 根据 `App` 主二进制可执行文件 `Mach-O` 的 `Header` 的内容配置进程的一些限制条件

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119111938.jpg)


- 判断是否为 `macOS` 执行环境，如果是的话，再判断上下文的一些配置属性是否被设置了，如果没有被设置，则再次进行一次 `setContext` 上下文配置操作。
- 根据传入的参数 `envp` 检查环境变量
- 默认未初始化的后备路径
- 判断是否为 `macOS` 执行环境，如果是的话，再判断当前 `app` 的 `Mach-O` 可执行文件是否为 `iOSMac` 类型且不为 `macOS` 类型的话，则重置上下文的根路径，然后再判断 `DYLD_FALLBACK_LIBRARY_PATH` 和 `DYLD_FALLBACK_FRAMEWORK_PATH` 这两个环境变量是否都是默认后备路径，如果是的话赋值为受限的后备路径。<br />

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119111950.jpg)


- 根据环境变量 `DYLD_PRINT_OPTS` 和 `DYLD_PRINT_ENV` 来判断是否需要打印
- 通过当前 `app` 的 `Mach-O` 可执行文件的 `header` 和 `ASLR` 之后的偏移量来获取架构信息。在这里会判断如果是 `GC` 的程序则会禁用掉共享缓存。<br />

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119112001.jpg)

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119112010.jpg)


- 判断共享缓存是否开启，如果开启了就将共享缓存映射到当前进程的逻辑内存空间内

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119112047.jpg)


- 检查共享缓存这里会先判断 `app` 的 `Mach-O` 二进制可执行文件是否有段覆盖了共享缓存区域，如果覆盖了则禁用共享缓存。但是这里的前提是 `macOS`，在 `iOS` 中，共享缓存是必需的。

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119112057.jpg)


> 这里为了方便查看，我们可以折叠一些分支条件。


- 通过共享缓存中的头的版本信息来判断是走 `dyld 2` 还是 `dyld 3` 的流程

## 3.4 dyld3 的处理

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119112122.jpg)

- 由于 `dyld3` 会创建一个启动闭包![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119112105.jpg)
，我们需要来读取它，这里会现在缓存中查找是否有启动闭包的存在，前面我们已经说过了，系统级的 `app` 的启动闭包是存在于共享缓存中，而我们自己开发的 `app` 的启动闭包是在 `app` 安装或者升级的时候构建的，所以这里检查 `dyld` 中的缓存是有意义的。

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119112229.jpg)


- 宏定义判断代码执行条件为真机。
- 如果 `dyld` 缓存中没有找到启动闭包或者找到了启动闭包但是验证失败（我们最开始提到的 `cdHash` 在这里出现了）
  - 从启动闭包缓存中查找
    - 如果还是没有找到，那就创建一个新的启动闭包

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119112240.jpg)


- 打印日志信息：`dyld3 启动开始`
- 尝试通过启动闭包进行启动
  - 如果启动失败，则创建一个新的启动闭包尝试再次启动
  - 如果启动成功，由于 `start()` 是以函数指针的方式调用 `_main` 方法的返回的指针，需要进行签名。

至此，`dyld3` 的流程就处理完毕，我们再接着往下分析 `dyld2` 的流程。

## 3.5 dyld2 的处理

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119112254.jpg)


- 这里会添加 `dyld` 的镜像文件到 `UUID` 列表中，主要的目的是启用堆栈的符号化。<br />

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119112300.jpg)


---


**reloadAllImages**

> `ImageLoader` 是一个用于加载可执行文件的基类，它负责链接镜像，但不关心具体文件格式，因为这些都交给子类去实现。每个可执行文件都会对应一个 `ImageLoader`实例。`ImageLoaderMachO` 是用于加载 `Mach-O` 格式文件的 `ImageLoader` 子类，而 `ImageLoaderMachOClassic` 和 `ImageLoaderMachOCompressed` 都继承于 `ImageLoaderMachO`，分别用于加载那些 `__LINKEDIT` 段为传统格式和压缩格式的 `Mach-O` 文件。


接下来就来到重头戏了 `reloadAllImages` 了：

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119112310.jpg)


---

**实例化主程序**

这里我们看到有一行代码:

```cpp
// instantiate ImageLoader for main executable
		sMainExecutable = instantiateFromLoadedImage(mainExecutableMH, mainExecutableSlide, sExecPath);
```

显然，在这里我们的主程序被实例化了，我们进入这个方法内部：

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119112334.jpg)


这里相当于要为已经映射到主可执行文件中的文件创建一个 `ImageLoader*`。

从上面代码我们不难看出这里真正执行的逻辑是 `ImageLoaderMachO::instantiateMainExecutable` 方法：

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119112344.jpg)


我们再进入 `sniiffLoadCommands` 方法内部：

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119112354.jpg)


通过注释不难看出：`sniiffLoadCommands` 会确定此 `mach-o` 文件是否具有原始的或压缩的 `LINKEDIT` 以及 `mach-o` 文件的 `segement` 的个数。

`sniiffLoadCommands` 完成后，判断 `LINKEDIT` 是压缩的格式还是传统格式，然后分别调用对应的 `instantiateMainExecutable` 方法来实例化主程序。

---

**加载任何插入的动态库**<br />

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119112524.jpg)


---

**链接库**

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119112533.jpg)


先是链接主二进制可执行文件，然后链接任何插入的动态库。这里都用到了 `link` 方法，在这个方法内部会执行递归的 `rebase` 操作来修正 `ASLR` 偏移量问题。同时还会有一个 `recursiveApplyInterposing` 方法来递归的将动态加载的镜像文件插入。

---

**运行所有初始化程序**

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119112542.jpg)


完成链接之后需要进行初始化了，这里会来到 `initializeMainExecutable`:

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119112548.jpg)


这里注意执行顺序：

- 先为所有插入并链接完成的动态库执行初始化操作
- 然后再为主程序可执行文件执行初始化操作

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119112555.jpg)


在 `runInitializers` 内部我们继续探索到 `processInitializers`:

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119112602.jpg)


然后我们来到 `recursiveInitialization`:

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119112609.jpg)


然后我们来到 `notifySingle`:

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119112617.jpg)


箭头所示的地方是获取镜像文件的真实地址。

我们全局搜索一下 `sNotifyObjcInit` 可以来到 `registerObjCNotifiers`：

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119112623.jpg)


接着搜索 `registerObjCNotifiers`：

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119112630.jpg)


此时，我们打开 `libObjc` 的源码可以看到:

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119112638.jpg)

上面这一连串的跳转，结果很显然：`dyld` 注册了回调才使得 `libobjc` 能知道镜像何时加载完毕。

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119112702.jpg)


在 `ImageLoader::recursiveInitialization` 方法中还有一个 `doInitialization` 值得注意，这里是真正做初始化操作的地方。

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119112714.jpg)


`doInitialization` 主要有两个操作，一个是 `doImageInit`，一个是 `doModInitFunctions`:

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119112723.jpg)


`doImageInit` 内部会通过初始地址 + 偏移量拿到初始化器 `func`，然后进行签名的验证。验证通过后还要判断初始化器是否在镜像文件中以及 `libSystem` 库是否已经初始化，最后才执行初始化器。

---

**通知监听 dyld 的 main**

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119112731.jpg)


一切工作做完后通知监听 `dyld` 的 `main`，然后为主二进制可执行文件找到入口，最后对结果进行签名。


# 四、探索 _objc_init

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119112747.jpg)


我们直接通过 `LLDB` 大法来断点调试 `libObjc` 中的 `_objc_init`，然后通过 `bt` 命令打印出当前的调用堆栈，根据上一节我们探索 `dyld` 的源码，此刻一切的一切都是那么的清晰明了：

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119112754.jpg)


我们可以看到 `dyld` 的最后一个流程是 `doModInitFunctions` 方法的执行。

我们打开 `libSystem` 的源码，全局搜索 `libSystem_initializer` 可以看到：

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119112803.jpg)


然后我们打开 `libDispatch` 的源码，全局搜索 `libdispatch_init` 可以看到：

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119113010.jpg)


我们再搜索 `_os_object_init`:

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119113018.jpg)


完美~，`_objc_init` 在这里就被调用了。所以 `_objc_init` 的流程是

dyld -> libSystem -> libDispatch -> libObc -> `_objc_init`

# 五、总结

本文主要探索了 `app` 启动之后 `dyld` 的流程，整个分析过程确实比较复杂，但在探索的过程中，我们不仅对底层源码有了新的认知，同时对于优化我们 `app` 启动也是有很多好处的。下一章，我们会对 `objc_init` 内部的 `map_images` 和 `load_images` 进行更深入的分析，敬请期待~

# 六、参考资料

[Optimizing App Startup Time](https://developer.apple.com/videos/play/wwdc2016/406/)

[优化 App 启动](https://xiaozhuanlan.com/topic/4690823715)

[iOS 开发中的『库』(一)](https://github.com/Damonvvong/DevNotes/blob/master/Notes/framework.md)

[iOS应用的内存管理（二）](https://www.awsomejiang.com/2018/01/15/Memory-Management-For-iOS-Apps-No-2/)

[优化 App 的启动时间](http://yulingtianxia.com/blog/2016/10/30/Optimizing-App-Startup-Time/#%E5%8A%A0%E8%BD%BD-Dylib)
