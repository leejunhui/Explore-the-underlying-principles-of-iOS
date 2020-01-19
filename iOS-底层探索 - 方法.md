![iOS 底层探索 - 方法](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119100021.jpg)


我们在前面探索了对象和类的底层原理，接下来我们要探索一下方法的本质，而在探索之前，我们先简单过一遍 `Runtime` 的知识点，如果读者对这块内容已经很熟悉了的话可以直接跳过第一章。
<!-- more -->
> PS: 由于笔者对汇编暂时还是摸索的阶段，关于汇编源码的部分如有错误，欢迎指正。

# `Runtime` 简介

众所周知，`Objective-C` 是一门动态语言，而承载整个 `OC` 动态特性的就是 `Runtime`。关于 `Runtime` 更多内容可以直接进入[官网文档](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Introduction/Introduction.html#//apple_ref/doc/uid/TP40008048)查看。

`Runtime` 是以 `C`/`C++`和汇编编写而成的，为什么不用 `OC` 呢，这是因为对我们编译器来说，`OC` 属于更高级的语言，相比于 `C` 和 `C++` 以及汇编，执行效率更慢，而在运行时系统需要尽可能快的执行效率。

## `Runtime` 的前世今生

`Runtime` 分为两个版本，`legacy` 和 `modern`，分别对标 `OC 1.0` 和 `OC 2.0`。我们通常只需要专注于 `modern` 版本即可，在 `libObjc` 源码中体现在 `new` 后缀的文件上。

## `Runtime` 三种交互方式

我们与 `Runtime` 打交道有三种方式:

- 直接在 `OC` 层进行交互：比如 `@selector`
- `NSObject` 的方法：`NSSelectorFromName`
- `Runtime` 的函数： `sel_registerName`

# 方法的本质探索

## 方法初探

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119095137.jpg)

我们可以看到，通过 `clang` 重写之后，`sayNB` 在底层其实是一个消息的发送。

我们把右侧的发送消息的代码简化一下:

```cpp
LGPerson *person = objc_msgSend((id)objc_getClass("LGPerson"), sel_registerName("alloc"));
objc_msgSend((id)person, sel_registerName("sayNB"));
```

由此可见，真正发送消息的地方是 `objc_msgSend`，这个方法有两个参数，一个是消息的接受者为 `id` 类型，第二个个是方法编号 `sel`。

作为对比，`run` 方法就直接执行了，并没有通过 `objc_msgSend` 进行消息发送:



## 方法发送的几种情况

```objectivec
LGStudent *s = [LGStudent new];
[s sayCode];        

objc_msgSend(s, sel_registerName("sayCode"));
```

上述代码表示的是向对象 `s` 发送 `sayCode` 消息。

---

```objectivec
id cls = [LGStudent class];
void *pointA = &cls;
[(__bridge id)pointA sayNB];

objc_msgSend(objc_getClass("LGStudent"), sel_registerName("sayNB"));
```

上述代码表示向 `LGStudent` 这个类发送 `sayNB` 消息。

---

```objectivec
// 向父类发消息(对象方法)
struct objc_super lgSuper;
lgSuper.receiver = s;
lgSuper.super_class = [LGPerson class];
objc_msgSendSuper(&lgSuper, @selector(sayHello));
```

上述代码表示向父类发送 `sayHello` 消息。

---

```objectivec
//向父类发消息(类方法)
struct objc_super myClassSuper;
myClassSuper.receiver = [s class];
myClassSuper.super_class = class_getSuperclass(object_getClass([s class]));// 元类
objc_msgSendSuper(&myClassSuper, sel_registerName("sayNB"));
```

上述代码表示向父类的类，也就是**元类**发送 `sayNB` 消息。

我们在 `OC` 中使用 `objc_msgSend` 的时候，要注意需要将 `Enbale Strict of Checking of objc_msgSend Calls` 设置为 `NO`。这样才不会报警告。

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119095253.jpg)

# 探索 `objc_msgSend`

`objc_msgSend` 之所以采用汇编来实现，是因为

- 汇编更容易能被机器识别
- 参数未知、类型未知对于 `C` 和 `C++` 来说不如汇编更得心应手

## 消息查找机制

- 快速流程
- 慢速流程

## 定位 `objc_msgSend` 汇编源码

```c
ENTRY _objc_msgSend
	UNWIND _objc_msgSend, NoFrame

	cmp	p0, #0			// nil check and tagged pointer check
```

判断 `p0` ，也就是我们 `objc_msgSend` 的第一个参数 `id` 消息的接收者是否为空。

```
ldr	p13, [x0]		// p13 = isa
	GetClassFromIsa_p16 p13		// p16 = class
```

读取 `x0` 然后赋值到 `p13` ，这里 `p13` 拿到的是 `isa`。为什么要拿 `isa` 呢，因为不论是对象方法还是类方法，我们都需要在类或者元类的缓存或方法列表中去查找，所以 `isa` 是必需的。

## GetClassFromIsa_p16

通过 `GetClassFromIsa_p16`，将获取到的 `class` 存在 `p16` 上面。

`GetClassFromIsa_p16` 源码如下:

```c
.macro GetClassFromIsa_p16 /* src */

#if SUPPORT_INDEXED_ISA
	// Indexed isa
	mov	p16, $0			// optimistically set dst = src
	tbz	p16, #ISA_INDEX_IS_NPI_BIT, 1f	// done if not non-pointer isa
	// isa in p16 is indexed
	adrp	x10, _objc_indexed_classes@PAGE
	add	x10, x10, _objc_indexed_classes@PAGEOFF
	ubfx	p16, p16, #ISA_INDEX_SHIFT, #ISA_INDEX_BITS  // extract index
	ldr	p16, [x10, p16, UXTP #PTRSHIFT]	// load class from array
1:

#elif __LP64__
	// 64-bit packed isa
	and	p16, $0, #ISA_MASK

#else
	// 32-bit raw isa
	mov	p16, $0

#endif

.endmacro
```

这个方法的目的就是通过位移操作获取 `isa` 的 `shiftcls` 然后进行位运算与操作得到真正的类信息。

```c
LGetIsaDone:
	CacheLookup NORMAL		// calls imp or objc_msgSend_uncached
```

## CacheLookup

获取完 `isa` 之后，接下来就要进行 `CacheLookup` ，查找方法缓存，我们再来到 `CacheLookup` 的源码处:

```c
/********************************************************************
 *
 * CacheLookup NORMAL|GETIMP|LOOKUP
 * 
 * Locate the implementation for a selector in a class method cache.
 *
 * Takes:
 *	 x1 = selector
 *	 x16 = class to be searched
 *
 * Kills:
 * 	 x9,x10,x11,x12, x17
 *
 * On exit: (found) calls or returns IMP
 *                  with x16 = class, x17 = IMP
 *          (not found) jumps to LCacheMiss
 *
 ********************************************************************/

.macro CacheLookup
	// p1 = SEL, p16 = isa
	ldp	p10, p11, [x16, #CACHE]	// p10 = buckets, p11 = occupied|mask
#if !__LP64__
	and	w11, w11, 0xffff	// p11 = mask
#endif
	and	w12, w1, w11		// x12 = _cmd & mask
	add	p12, p10, p12, LSL #(1+PTRSHIFT)
		             // p12 = buckets + ((_cmd & mask) << (1+PTRSHIFT))

	ldp	p17, p9, [x12]		// {imp, sel} = *bucket
1:	cmp	p9, p1			// if (bucket->sel != _cmd)
	b.ne	2f			//     scan more
	CacheHit $0			// call or return imp
	
2:	// not hit: p12 = not-hit bucket
	CheckMiss $0			// miss if bucket->sel == 0
	cmp	p12, p10		// wrap if bucket == buckets
	b.eq	3f
	ldp	p17, p9, [x12, #-BUCKET_SIZE]!	// {imp, sel} = *--bucket
	b	1b			// loop

3:	// wrap: p12 = first bucket, w11 = mask
	add	p12, p12, w11, UXTW #(1+PTRSHIFT)
		                        // p12 = buckets + (mask << 1+PTRSHIFT)

	// Clone scanning loop to miss instead of hang when cache is corrupt.
	// The slow path may detect any corruption and halt later.

	ldp	p17, p9, [x12]		// {imp, sel} = *bucket
1:	cmp	p9, p1			// if (bucket->sel != _cmd)
	b.ne	2f			//     scan more
	CacheHit $0			// call or return imp
	
2:	// not hit: p12 = not-hit bucket
	CheckMiss $0			// miss if bucket->sel == 0
	cmp	p12, p10		// wrap if bucket == buckets
	b.eq	3f
	ldp	p17, p9, [x12, #-BUCKET_SIZE]!	// {imp, sel} = *--bucket
	b	1b			// loop

3:	// double wrap
	JumpMiss $0
	
.endmacro
```

通过上述代码可知 `CacheLookup` 有三种模式：`NORMAL`，`GETIMP`， `LOOKUP`

```c
ldp	p10, p11, [x16, #CACHE]
```

- `CacheLookup` 需要读取上一步拿到的类的 `cache` 缓存，而根据我们前面对类结构的学习，这里显然进行 16 字节地址平移操作，然后把拿到的 `cache_t` 中的 `buckets` 和 `occupied`、`mask` 赋值给 `p10`, `p11`。

```c
and	w12, w1, w11		// x12 = _cmd & mask
	add	p12, p10, p12, LSL #(1+PTRSHIFT)
		             // p12 = buckets + ((_cmd & mask) << (1+PTRSHIFT))

	ldp	p17, p9, [x12]		// {imp, sel} = *bucket
```

- 这里是将 `w1` 和 `w11` 进行与操作，其实本质就是 `_cmd` & `mask`。这一步和我们探索 `cache_t` 时遇到的
![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119095327.jpg)
是一模模一样样的道理。目的就是拿到下标。然后经过哈希运算之后，得到了 `bucket` 结构体指针，然后将这个结构体指针中的 `imp`，`sel` 分别存在 `p17`，`p9` 中。

```c
1:	cmp	p9, p1			// if (bucket->sel != _cmd)
	b.ne	2f			//     scan more
	CacheHit $0			// call or return imp
```

- 接着我们将上一步获取到的 `sel` 和我们要查找的 `sel`（在这里也就是所谓的 `_cmd`）进行比较，如果匹配了，就通过 `CacheHit` 将 `imp` 返回；如果没有匹配，就走下一步流程。

```c
2:	// not hit: p12 = not-hit bucket
	CheckMiss $0			// miss if bucket->sel == 0
	cmp	p12, p10		// wrap if bucket == buckets
	b.eq	3f
	ldp	p17, p9, [x12, #-BUCKET_SIZE]!	// {imp, sel} = *--bucket
	b	1b			// loop
```

- 由于上一步的 `sel` 没有匹配上，我们需要接着进行搜索。

## CheckMiss

我们来到 `CheckMiss` 的源码:

```c
.macro CheckMiss
	// miss if bucket->sel == 0
.if $0 == GETIMP
	cbz	p9, LGetImpMiss
.elseif $0 == NORMAL
	cbz	p9, __objc_msgSend_uncached
.elseif $0 == LOOKUP
	cbz	p9, __objc_msgLookup_uncached
.else
.abort oops
.endif
.endmacro
```

这里由于我们是 `NORMAL` 模式，所以会来到 `__objc_msgSend_uncached`:

```c
STATIC_ENTRY __objc_msgSend_uncached
	UNWIND __objc_msgSend_uncached, FrameWithNoSaves

	// THIS IS NOT A CALLABLE C FUNCTION
	// Out-of-band p16 is the class to search
	
	MethodTableLookup
	TailCallFunctionPointer x17

	END_ENTRY __objc_msgSend_uncached
```

`__objc_msgSend_uncached` 中最核心的逻辑就是 `MethodTableLookup`，意为查找方法列表。

<a name="ab4985f9"></a>
## 3.6 MethodTableLookup

我们再来到 `MethodTableLookup` 的定义:

```c
.macro MethodTableLookup
	
	// push frame
	SignLR
	stp	fp, lr, [sp, #-16]!
	mov	fp, sp

	// save parameter registers: x0..x8, q0..q7
	sub	sp, sp, #(10*8 + 8*16)
	stp	q0, q1, [sp, #(0*16)]
	stp	q2, q3, [sp, #(2*16)]
	stp	q4, q5, [sp, #(4*16)]
	stp	q6, q7, [sp, #(6*16)]
	stp	x0, x1, [sp, #(8*16+0*8)]
	stp	x2, x3, [sp, #(8*16+2*8)]
	stp	x4, x5, [sp, #(8*16+4*8)]
	stp	x6, x7, [sp, #(8*16+6*8)]
	str	x8,     [sp, #(8*16+8*8)]

	// receiver and selector already in x0 and x1
	mov	x2, x16
	bl	__class_lookupMethodAndLoadCache3

	// IMP in x0
	mov	x17, x0
	
	// restore registers and return
	ldp	q0, q1, [sp, #(0*16)]
	ldp	q2, q3, [sp, #(2*16)]
	ldp	q4, q5, [sp, #(4*16)]
	ldp	q6, q7, [sp, #(6*16)]
	ldp	x0, x1, [sp, #(8*16+0*8)]
	ldp	x2, x3, [sp, #(8*16+2*8)]
	ldp	x4, x5, [sp, #(8*16+4*8)]
	ldp	x6, x7, [sp, #(8*16+6*8)]
	ldr	x8,     [sp, #(8*16+8*8)]

	mov	sp, fp
	ldp	fp, lr, [sp], #16
	AuthenticateLR

.endmacro
```

我们观察 `MethodTableLookup` 内容之后会定位到 `__class_lookupMethodAndLoadCache3`。在 `__class_lookupMethodAndLoadCache3` 之前会做一些准备工作，真正的方法查找流程核心逻辑是位于 `__class_lookupMethodAndLoadCache3` 里面的。 但是我们全局搜索 `__class_lookupMethodAndLoadCache3` 会发现找不到，这是因为此时我们会从汇编跳入到 `C/C++`。所以去掉一个下划线就能找到:

```c
IMP _class_lookupMethodAndLoadCache3(id obj, SEL sel, Class cls)
{
    return lookUpImpOrForward(cls, sel, obj, 
                              YES/*initialize*/, NO/*cache*/, YES/*resolver*/);
}
```

<a name="c57341de"></a>
# 总结

- 方法的本质就是消息发送，消息发送是通过 `objc_msgSend` 以及其派生函数来实现的。
- `objc_msgSend` 为了执行效率以及 C/C++ 不能支持参数未知，类型未知的代码，所以采用汇编来实现 `objc_msgSend`。
- 消息查找或者说方法查找，会优先去从类中查找缓存，找到了就返回，找不到就需要去类的方法列表中查找。
- 由汇编过渡到 C/C++，在类的方法列表中查找失败之后，会进行转发。核心逻辑位于 `lookUpImpOrForward`。

我们下一章将会从 `lookUpImpOrForward` 开始探索，探索底层的方法查找的具体流程到底是怎么样的，敬请期待~
