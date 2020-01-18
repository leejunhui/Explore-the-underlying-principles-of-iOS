![iOS 底层探索 - calloc 和 isa](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119022152.jpg)

上一篇文章主要我们探索了 `iOS`  对象的 `alloc` 和 `init` 以及对象是怎么开辟内存以及初始化的，如果在对象身上增加一些属性，是否会影响内存开辟呢？还有一个遗留问题就是通过 `calloc` ，我们的对象有了内存地址，但是对象结构里面的 `isa` 是怎么关联到我们的对象的内存地址的呢。

# `calloc` 底层探索

在探索 `calloc` 底层前，我们先补充一下内存对齐相关的知识点。

## 内存对齐三原则

在 `iOS` 中，对象的属性需要进行内存对齐，而对象本身也需要进行内存对齐。<br />内存对齐有三原则

- 数据成员对齐原则: 结构( `struct` )(或联合( `union` ))的数据成员，第<br />
一个数据成员放在 offset 为 0 的地方，以后每个数据成员存储的起始位置要<br />
从该成员大小或者成员的子成员大小
- 结构体作为成员: 如果一个结构里有某些结构体成员,则结构体成员要从<br />
其内部最大元素大小的整数倍地址开始存储
- 收尾工作: 结构体的总大小,也就是 `sizeof` 的结果,.必须是其内部最大<br />
成员的整数倍.不足的要补⻬。<br />
翻译一下就是：
- **前面的地址必须是后面的地址正数倍,不是就补齐**
- 结构体里面的嵌套结构体大小要以该**嵌套结构体最大元素大小的整数倍**
- **整个 **`**Struct**` **的地址必须是最大字节的整数倍**

## 对象申请内存和系统开辟内存

我们通过打印下面的代码：

```objectivec
NSLog(@"%lu - %lu",class_getInstanceSize([p class]),malloc_size((__bridge const void *)(p)));
```

可以发现对象自己申请的内存大小与系统实际给我们开辟的大小时不一样的，这里对象申请的内存大小是 **40** 个字节，而系统开辟的是 **48** 个字节。

40 个字节不难理解，是因为当前对象有 4 个属性，有三个属性为 8 个字节，有一个属性为 4个字节，再加上 isa 的 8 个字节，就是 32 + 4 = 36 个字节，然后根据内存对齐原则，36 不能被 8 整除，36 往后移动刚好到了 40 就是 8 的倍数，所以内存大小为 40。

48 个字节的话需要我们探索 `calloc` 的底层原理。

这里还有一个注意点，就是 `class_getInstanceSize` 和 `malloc_size` 对同一个对象返回的结果不一样的，原因是 `malloc_size` 是直接返回的 `calloc` 之后的指针的大小，回忆上一节课，这里有一步在调用 `calloc` 之前的操作如下:

```objective-c
size_t instanceSize(size_t extraBytes) {
    size_t size = alignedInstanceSize() + extraBytes;
    // CF requires all objects be at least 16 bytes.
    if (size < 16) size = 16;
    return size;
}
```

而 `class_getInstanceSize` 内部实现是:

```objective-c
size_t class_getInstanceSize(Class cls)
{
    if (!cls) return 0;
    return cls->alignedInstanceSize();
}
```

也就是说 `class_getInstanceSize` 会输出 8 个字节，`malloc_size` 会输出 16 个字节，当然前提是该对象没有任何属性。

## 探索 calloc 底层

我们从 `calloc` 函数出发，但是我们直接在 `libObjc` 的源码中是找不到其对应实现的，通过观察 Xcode 我们知道其实应该找 `libMalloc` 源码才对:

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119021547.jpg)

这里有个小技巧，其实我们研究的是 `calloc` 的底层原理，而 `libObjc` 和 `libMalloc` 是相互独立的，所以在 `libMalloc` 源码里面，我们没必要去走 `calloc` 前面的流程了。我们通过断点调试 `libObjc` 源码可以知道第二个参数是 40: (这是因为当前发送 `alloc` 消息的对象有 4 个属性，每个属性 8 个字节，再加上 isa 的 8 个字节，所以就是 40 个字节)

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119021606.jpg)


接下来我们打开 `libMalloc` 的源码，在新建的 target 中直接手动声明如下的代码:

```objective-c
void *p = calloc(1, 40);
NSLog(@"%lu",malloc_size(p));
```

但 `Command + Run` 之后我们会看到报错信息:

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119021741.jpg)


这个时候我们会使用搜索大法，直接 `Command + Shift + F` 进行全局搜索对应的符号，但是会发现找不到，我们再仔细观察，这些符号都是位于 `.o` 文件里面的，所以我们可以去掉符号前面的下划线再进行搜索，这个时候就可以把对应的代码注释然后重新运行了。

运行之后我们一直沿着源码断点下去，会来到这么一段代码

```objectivec
ptr = zone->calloc(zone, num_items, size);
```

我们如果直接去找 `calloc`，就会递归了，所以我们需要点进去，然后我们会发现一个很复杂的东西出现了:

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119021755.jpg)


这里我们可以直接在断点处使用 `LLDB` 命令打印这行代码来看具体实现是位于哪个文件中

```lldb
p zone->calloc
输出: (void *(*)(_malloc_zone_t *, size_t, size_t)) $1 = 0x00000001003839c7 (.dylib`default_zone_calloc at malloc.c:249)
```

也就是说 `zone->alloc` 的真正实现是在 `malloc.c` 源文件的249行处。

```c
static void *
default_zone_calloc(malloc_zone_t *zone, size_t num_items, size_t size)
{
	zone = runtime_default_zone();
	
	return zone->calloc(zone, num_items, size);
}
```

但是我们发现这里又是一次 `zone->calloc`，我们接着再次使用 `LLDB` 打印内存地址:

```lldb
p zone->calloc
输出: (void *(*)(_malloc_zone_t *, size_t, size_t)) $0 = 0x0000000100384faa (.dylib`nano_calloc at nano_malloc.c:884)
```

我们再次来到 `nano_calloc` 方法

```objectivec
static void *
nano_calloc(nanozone_t *nanozone, size_t num_items, size_t size)
{
	size_t total_bytes;

	if (calloc_get_size(num_items, size, 0, &total_bytes)) {
		return NULL;
	}

	if (total_bytes <= NANO_MAX_SIZE) {
		void *p = _nano_malloc_check_clear(nanozone, total_bytes, 1);
		if (p) {
			return p;
		} else {
			/* FALLTHROUGH to helper zone */
		}
	}
	malloc_zone_t *zone = (malloc_zone_t *)(nanozone->helper_zone);
	return zone->calloc(zone, 1, total_bytes);
}
```

我们简单分析一下，应该往 `_nano_malloc_check_clear` 里面继续走，然后我们发现 `_nano_malloc_check_clear` 里面内容非常多，这个时候我们要明确一点，我们的目的是找出 48 是怎么算出来的，经过分析之后，我们来到 `segregated_size_to_fit`

```c
static MALLOC_INLINE size_t
segregated_size_to_fit(nanozone_t *nanozone, size_t size, size_t *pKey)
{
	// size = 40
	size_t k, slot_bytes;

	if (0 == size) {
		size = NANO_REGIME_QUANTA_SIZE; // Historical behavior
	}
	// 40 + 16-1 >> 4 << 4
	// 40 - 16*3 = 48

	//
	// 16
	k = (size + NANO_REGIME_QUANTA_SIZE - 1) >> SHIFT_NANO_QUANTUM; // round up and shift for number of quanta
	slot_bytes = k << SHIFT_NANO_QUANTUM;							// multiply by power of two quanta size
	*pKey = k - 1;													// Zero-based!

	return slot_bytes;
}
```

这里可以看出进行的是 16 字节对齐，那么也就是说我们传入的 `size` 是 40，在经过 (40 + 16 - 1) >> 4 << 4 操作后，结果为48，也就是16的整数倍。

总结:

- 对象的属性是进行的 8 字节对齐
- 对象自己进行的是 16 字节对齐
  - 因为内存是连续的，通过 16 字节对齐规避风险和容错，防止访问溢出
  - 同时，也提高了寻址访问效率，也就是**空间换时间**

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119021811.jpg)


# `isa` 底层探索

## 联合体位域

```objectivec
union isa_t {
    isa_t() { }
    isa_t(uintptr_t value) : bits(value) { }

    Class cls;
    uintptr_t bits;
#if defined(ISA_BITFIELD)
    struct {
        ISA_BITFIELD;  // defined in isa.h
    };
#endif
};
```

我们探索 `isa` 的时候，会发现 `isa` 其实是一个联合体，而这其实是从内存管理层面来设计的，因为联合体是所有成员共享一个内存，联合体内存的大小取决于内部成员内存大小最大的那个元素，对于 `isa` 指针来说，就不用额外声明很多的属性，直接在内部的 `ISA_BITFIELD` 保存信息。同时由于联合体属性间是互斥的，所以 `cls` 和 `bits` 在 `isa` 初始化流程时是在两个分支中被赋值的。

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119021845.jpg)

## isa 结构

`isa` 作为一个联合体，有一个结构体属性为 `ISA_BITFIELD`，其大小为 8 个字节，也就是 64 位。<br />下面的代码是基于 `arm64` 架构的:

```c
#   define ISA_BITFIELD                                                      \
      uintptr_t nonpointer        : 1;                                       \
      uintptr_t has_assoc         : 1;                                       \
      uintptr_t has_cxx_dtor      : 1;                                       \
      uintptr_t shiftcls          : 33; /*MACH_VM_MAX_ADDRESS 0x1000000000*/ \
      uintptr_t magic             : 6;                                       \
      uintptr_t weakly_referenced : 1;                                       \
      uintptr_t deallocating      : 1;                                       \
      uintptr_t has_sidetable_rc  : 1;                                       \
      uintptr_t extra_rc          : 19
```

- `nonpointer`: 表示是否对 `isa` 指针开启指针优化
  - 0: 纯 `isa` 指针
  - 1: 不止是类对象地址, `isa` 中包含了类信息、对象的引用计数等
- has_assoc: 关联对象标志位，0 没有，1 存在
- has_cxx_dtor: 该对象是否有 C++ 或者 Objc 的析构器,如果有析构函数,则需要做析构逻辑, 如果没有,则可以更快的释放对象
- shiftcls: 存储类指针的值。开启指针优化的情况下，在 arm64 架构中有 33 位用来存储类指针。
- magic: 用于调试器判断当前对象是真的对象还是没有初始化的空间
- weakly_referenced: 标志对象是否被指向或者曾经指向一个 ARC 的弱变量，<br />
没有弱引用的对象可以更快释放。
- deallocating: 标志对象是否正在释放内存
- has_sidetable_rc: 当对象引用技术大于 10 时，则需要借用该变量存储进位
- extra_rc: 当表示该对象的引用计数值，实际上是引用计数值减 1， 例如，如果对象的引用计数为 10，那么 extra_rc 为 9。如果引用计数大于 10， 则需要使用到下面的 has_sidetable_rc。

## isa 关联对象和类

`isa` 是对象中的第一个属性，因为这一步是在继承的时候发生的，要早于对象的成员变量，属性列表，方法列表以及所遵循的协议列表。

我们在探索 `alloc` 底层原理的时候，有一个方法叫做 `initIsa`。

这个方法的作用就是初始化 `isa` 联合体位域。其中有这么一行代码：

```c
newisa.shiftcls = (uintptr_t)cls >> 3;
```

通过这行代码，我们知道 `shiftcls` 这个位域其实存储的是类的信息。这个类就是实例化对象所指向的那个类。

通过 `LLDB` 进行调试打印，我们可以知道一个对象的 `isa` 会关联到这个对象所属的类。

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119021910.jpg)


这里的左移右移操作其实很好理解，首先我们先观察 `isa` 的 `ISA_BITFIELD` 位域的结构:

```c
// 注：这里是x64架构
# elif __x86_64__
#   define ISA_MASK        0x00007ffffffffff8ULL
#   define ISA_MAGIC_MASK  0x001f800000000001ULL
#   define ISA_MAGIC_VALUE 0x001d800000000001ULL
#   define ISA_BITFIELD                                                        \
      uintptr_t nonpointer        : 1;                                         \
      uintptr_t has_assoc         : 1;                                         \
      uintptr_t has_cxx_dtor      : 1;                                         \
      uintptr_t shiftcls          : 44; /*MACH_VM_MAX_ADDRESS 0x7fffffe00000*/ \
      uintptr_t magic             : 6;                                         \
      uintptr_t weakly_referenced : 1;                                         \
      uintptr_t deallocating      : 1;                                         \
      uintptr_t has_sidetable_rc  : 1;                                         \
      uintptr_t extra_rc          : 8
```

我们可以看到，`ISA_BITFIELD` 的前 3 位是 `nonpointer`，`has_assoc`，`has_cxx_dtor`，中间 44 位是 `shiftcls` ，后面 17 位是剩余的内容，同时因为 iOS 是小端模式，那么我们就需要去掉右边的 3 位和左边的 17位，所以就会采用 >>3<<3 然后 <<17>>17 的操作了。

通过这个测试，我们就知道了 `isa` 实现了对象与类之间的关联。

我们还可以探索 `object_getClass` 底层，可以发现有这样一行代码：

```oc
return (Class)(isa.bits & ISA_MASK);
```

这行代码就是将 `isa` 中的联合体位域与上一个蒙版，这个蒙版定义是怎么样的呢?

```c
#   define ISA_MASK        0x00007ffffffffff8ULL
```

`0x00007ffffffffff8ULL` 这个值我们转成二进制表示:

```c
0000 0000 0000 0000 0111 1111 1111 1111 
1111 1111 1111 1111 1111 1111 1111 1000
```

结果一目了然，这个蒙版就是帮我们去过滤掉除 `shiftcls` 之外的内容。

我们直接将对象的 `isa` 地址与上这个mask之后，就会得到 `object.class` 一样的内存地址。

## isa 走位分析

### 类与元类

我们都知道对象可以创建多个，但是类是否可以创建多个呢?<br />答案很简单，一个。那么如果来验证呢?

```objective-c
//MARK: - 分析类对象内存存在个数
void lgTestClassNum(){
    Class class1 = [LGPerson class];
    Class class2 = [LGPerson alloc].class;
    Class class3 = object_getClass([LGPerson alloc]);
    Class class4 = [LGPerson alloc].class;
    NSLog(@"\n%p-\n%p-\n%p-\n%p",class1,class2,class3,class4);
}

// 打印输出如下:

0x100002108-
0x100002108-
0x100002108-
0x100002108
```

所以我们就知道了类在内存中只会存在一份。

```oc
(lldb) x/4gx LGTeacher.class
0x100001420: 0x001d8001000013f9 0x0000000100b38140
0x100001430: 0x00000001003db270 0x0000000000000000
(lldb) po 0x001d8001000013f9
17082823967917874

(lldb) p 0x001d8001000013f9
(long) $2 = 8303516107936761
(lldb) po 0x100001420
LGTeacher
```

我们通过上面的打印，就发现 类的内存结构里面的第一个结构打印出来还是 `LGTeacher`，那么是不是就意味着 对象->类->类 这样的死循环呢？这里的第二个类其实是 `元类`。是由系统帮我们创建的。这个元类也无法被我们实例化。

也就是下面的这种关系:

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119021925.jpg)


```oc
(lldb) p/x 0x001d8001000013f9 & 0x00007ffffffffff8
(long) $4 = 0x00000001000013f8
(lldb) po 0x00000001000013f8
LGTeacher

(lldb) x/4gx 0x00000001000013f8
0x1000013f8: 0x001d800100b380f1 0x0000000100b380f0
0x100001408: 0x0000000101c30230 0x0000000100000007
(lldb) p/x 0x001d800100b380f1 & 0x00007ffffffffff8
(long) $6 = 0x0000000100b380f0
(lldb) po 0x0000000100b380f0
NSObject
```

### isa 走位

我们在 Xcode 中测试有以下结果：

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119021938.jpg)


由此可以给出官方的经典 `isa` 走位图

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119022003.jpg)


## isa 初始化流程图

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119022016.jpg)


# 对象的本质

在我们认知里面，`OC` 对象的本质就是一个结构体，这个结论在 `libObjc` 源码的 `objc-private.h` 源文件中可以得到证实。

```c
struct objc_object {
private:
    isa_t isa;

public:

    // ISA() assumes this is NOT a tagged pointer object
    Class ISA();

    // getIsa() allows this to be a tagged pointer object
    Class getIsa();

    ...省略其他的内容...
}
```

而对于对象所属的类来说，我们也可以在 `objc-runtime-new.h` 源文件中找到

```c
struct objc_class : objc_object {
    // Class ISA;
    Class superclass;
    cache_t cache;             // formerly cache pointer and vtable
    class_data_bits_t bits;    // class_rw_t * plus custom rr/alloc flags
    
    ...省略其他的内容...
    
}
```

也就是说 `objc_class` 内存中第一个位置是 `isa`，第二个位置是 `superclass`。

不过我们本着求真的态度可以用 `clang` 来重写我们的 `OC` 源文件来查看是不是这么回事。

```bash
clang -rewrite-objc main.m -o main.cpp
```

这行命令会把我们的 `main.m` 文件编译成 `C++` 格式，输出为 `main.cpp`。

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119022040.jpg)


我们可以看到 `LGPerson` 对象在底层其实是一个结构体 `objc_object` 。

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119022051.jpg)

而我们的 `Class` 在底层也是一个结构体 `objc_class` 。

# 总结

至此， `iOS` 底层探索之对象篇更新完毕，现在来回顾一下我们所探索的内容。

- alloc & init 流程剖析
- 内存开辟
- 字节对齐算法
- isa 初始化和走位
- 对象的本质


