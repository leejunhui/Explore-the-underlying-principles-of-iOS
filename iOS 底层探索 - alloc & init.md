# iOS 底层探索对象篇之 alloc &amp; init

# alloc & init 探索

作为 `iOS` 开发者，我们每天打交道最多的应该就是对象了，从面向对象设计的角度来说，对象的创建以及初始化是最基础的内容。那么，今天我们就一起来探索一下 `iOS` 中最常用的 `alloc` 和 `init`  的底层是怎么实现的吧。

## 一、 如何进行底层探索

对于第三方开源框架来说，我们去剖析内部原理和细节是有一定的方法和套路可以掌握的。而对于 `iOS`  底层，特别是 `OC` 底层，我们可能就需要用到一些开发中不是很常用的方法。

我们这个系列主要的目的是为了进行底层探索，那么我们作为 `iOS` 开发者，需要关注应该就是从应用启动到应用被 `kill` 掉这一整个生命周期的内容。我们不妨从我们最熟悉的 `main` 函数开始，一般来说，我们在 `main.m` 文件中打一个断点，左侧的调用堆栈视图应该如下图所示:

![image.png](https://cdn.nlark.com/yuque/0/2019/png/225346/1576996413231-49712187-4d67-466a-8ef6-9d4560753061.png#align=left&display=inline&height=369&name=image.png&originHeight=738&originWidth=964&size=733652&status=done&style=none&width=482)

> 要得到这样的调用堆栈有两个注意点:
> - 需要关闭 `Xcode` 左侧 `Debug` 区域最下面的 `show only stack frames with debug symbols and between libraries`
> 
![image.png](https://cdn.nlark.com/yuque/0/2019/png/225346/1576995667098-971d192d-4e30-4114-ade5-7cfd7062160f.png#align=left&display=inline&height=37&name=image.png&originHeight=74&originWidth=902&size=31589&status=done&style=none&width=451)
> 
> - 需要增加一个 `_objc_init` 的符号端点
> 
![image.png](https://cdn.nlark.com/yuque/0/2019/png/225346/1576995762039-b844de75-3105-4957-a264-40e03721f0d5.png#align=left&display=inline&height=86&name=image.png&originHeight=172&originWidth=900&size=118926&status=done&style=none&width=450)


我们通过上面的调用堆栈信息不难得出一个简单粗略的加载流程结构

![iOS粗略流程](https://cdn.nlark.com/yuque/0/2019/png/225346/1576996997408-1ebf1e41-3aec-429b-9225-08ea1c51262d.png#align=left&display=inline&height=314&name=iOS%E7%B2%97%E7%95%A5%E6%B5%81%E7%A8%8B&originHeight=314&originWidth=783&size=0&status=done&style=none&width=783)


我们现在心中建立这么一个简单的流程结构，在后期分析底层的时候我们会回过头来梳理整个启动的流程。

接下来，让我们开始实际的探索过程。

我们直接打开 `Xcode` 新建一个 `Single View App` 工程，然后我们在 `ViewController.m` 文件中调用 `alloc` 方法。

```objectivec
NSObject *p = [NSObject alloc];
```

我们按照常规探索源码的方式，直接按住 `Command` + `Control` 来进入到 `alloc` 内部实现，但结果并非如我们所愿，我们来到的是一个头文件，只有 `alloc` 方法的声明，并没有对应的实现。这个时候，我们会陷入深深的怀疑中，其实这个时候我们只要记住下面三种常用探索方式就能迎刃而解：

### 1.1 直接下代码断点
具体操作方式为 `Control` + `in` 

![image.png](https://cdn.nlark.com/yuque/0/2019/png/225346/1576997689140-5faefb7e-3b06-419e-81af-eb8ce405d8f8.png#align=left&display=inline&height=33&name=image.png&originHeight=66&originWidth=372&size=4558&status=done&style=none&width=186) 这里的 `in` 指的是左侧图片中红色部分的按钮，其实这里的操作叫做 `Step into instruction` 。我们可以来到下图这里

![image.png](https://cdn.nlark.com/yuque/0/2019/png/225346/1576998100260-e7385d08-0887-44a1-976f-e4e92f28b52d.png#align=left&display=inline&height=200&name=image.png&originHeight=400&originWidth=1884&size=191365&status=done&style=none&width=942)

我们观察不难得出我们想要找的就是 `libobjc.A.dylib` 这个动态链接库了。

### 1.2 打开反汇编显示
具体操作方式为打开 `Debug` 菜单下的 `Debug Workflow` 下的 `Always Show Disassembly` 

![image.png](https://cdn.nlark.com/yuque/0/2019/png/225346/1577004035249-01e50500-1bec-4a7d-8b89-2d93bd88d23a.png#align=left&display=inline&height=112&name=image.png&originHeight=224&originWidth=1022&size=253989&status=done&style=none&width=511)

接着我们还是下代码断点，然后一步一步调试也会来到下图这里:

![image.png](https://cdn.nlark.com/yuque/0/2019/png/225346/1577004749590-e15ba858-2061-4f14-9f64-b8be96c3ec6a.png#align=left&display=inline&height=192&name=image.png&originHeight=384&originWidth=1940&size=188650&status=done&style=none&width=970)

### 1.3 下符号断点
我们先选择 `Symbolic Breakpoint`，然后输入 `objc_alloc` ，如下图所示：

![image.png](https://cdn.nlark.com/yuque/0/2019/png/225346/1577004913212-fc853c8f-b395-4c5f-adb0-c65c909970c9.png#align=left&display=inline&height=134&name=image.png&originHeight=268&originWidth=424&size=109931&status=done&style=none&width=212) ![image.png](https://cdn.nlark.com/yuque/0/2019/png/225346/1577004960976-e55fc68e-ec73-4c21-955f-44f9c12b450c.png#align=left&display=inline&height=191&name=image.png&originHeight=382&originWidth=952&size=565114&status=done&style=none&width=476)

![image.png](https://cdn.nlark.com/yuque/0/2019/png/225346/1577005028296-6459165b-b0e3-4326-8eb1-a73797e3e276.png#align=left&display=inline&height=189&name=image.png&originHeight=378&originWidth=1942&size=188236&status=done&style=none&width=971)

至此，我们得到了 `alloc` 实现位于 `libObjc` 这个动态库，而刚好苹果已经开源了这部分的代码，所以我们可以在 [苹果开源官网 最新版本 10.14.5](https://opensource.apple.com/release/macos-10145.html) 上下载即可。最新的 `libObc` 为 756。

![image.png](https://cdn.nlark.com/yuque/0/2019/png/225346/1577005271313-1e5eb2c9-3a5d-4c2f-8a24-869bbc66d528.png#align=left&display=inline&height=24&name=image.png&originHeight=48&originWidth=1312&size=6144&status=done&style=none&width=656)

## 二、 探索 `libObjc` 源码
我们下载了 `libObjc` 的源码到我们的电脑上后是不能直接运行的，我们需要进行一定的配置才能实现源码追踪流程。这一块内容不在本文范围内，读者可参考 [iOS_objc4-756.2 最新源码编译调试](https://juejin.im/post/5d9c829df265da5ba46f49c9)。

配置好 `libObjc` 之后，我们新建一个命令行的项目，然后运行如下代码:

```objectivec
NSObject *myObj = [NSObject alloc];
```

### 2.1 objc_alloc
然后我们直接下符号断点 `objc_alloc` ，然后一步步调试，先来到的是 `objc_alloc` 

```objectivec
// Calls [cls alloc].
id
objc_alloc(Class cls)
{
    return callAlloc(cls, true/*checkNil*/, false/*allocWithZone*/);
}
```

### 2.2 第一次 callAlloc
然后会来到 `callAlloc` 方法，注意这里第三个参数传的是 `false` 

```objectivec
static ALWAYS_INLINE id
callAlloc(Class cls, bool checkNil, bool allocWithZone=false)
{
    // 判断传入的 checkNil 是否进行判空操作
    if (slowpath(checkNil && !cls)) return nil;

    // 如果当前编译环境为 OC 2.0
#if __OBJC2__
    // 当前类没有自定义的 allocWithZone
    if (fastpath(!cls->ISA()->hasCustomAWZ())) {
        // No alloc/allocWithZone implementation. Go straight to the allocator.
        // 既没有实现 alloc，也没有实现 allocWithZone 就会来到这里，下面直接进行内存开辟操作。
        // fixme store hasCustomAWZ in the non-meta class and 
        // add it to canAllocFast's summary
        // 修复没有元类的类，用人话说就是没有继承于 NSObject
        // 判断当前类是否可以快速开辟内存，注意，这里永远不会被调用，因为 canAllocFast 内部
        // 返回的是false
        if (fastpath(cls->canAllocFast())) {
            // No ctors, raw isa, etc. Go straight to the metal.
            bool dtor = cls->hasCxxDtor();
            id obj = (id)calloc(1, cls->bits.fastInstanceSize());
            if (slowpath(!obj)) return callBadAllocHandler(cls);
            obj->initInstanceIsa(cls, dtor);
            return obj;
        }
        else {
            // Has ctor or raw isa or something. Use the slower path.
            id obj = class_createInstance(cls, 0);
            if (slowpath(!obj)) return callBadAllocHandler(cls);
            return obj;
        }
    }
#endif

    // No shortcuts available.
    if (allocWithZone) return [cls allocWithZone:nil];
    return [cls alloc];
}
```

### 2.3 _objc_rootAlloc
因为我们在 `objc_init`  中传入的第三个参数 `allocWithZone` 是 `true` ，并且我们的 `cls` 为 `NSObject` ，那么也就是说会这里直接来到 `return [cls alloc]` 。我们接着往下走会来到 `alloc` 方法：<br /> 
```objectivec
+ (id)alloc {
    return _objc_rootAlloc(self);
}
```

然后我们接着进入 `_objc_rootAlloc` 方法内部:

```objectivec
// Base class implementation of +alloc. cls is not nil.
// Calls [cls allocWithZone:nil].
id
_objc_rootAlloc(Class cls)
{
    return callAlloc(cls, false/*checkNil*/, true/*allocWithZone*/);
}
```

### 2.4 第二次 callAlloc

是不是有点似曾相似，没错，我们第一步进入的 `objc_init` 也是调用的 `callAlloc` 方法，但是这里有两个参数是不一样的，第二个参数 `checkNil` 是否需要判空直接传的是 `false` ，站在系统角度，前面已经在第一次调用 `callAlloc`  的时候进行了判空了，所以这里没必要再次进行判空的了。第三个参数 `allocWithZone` 传的是 `true` ，关于这个方法，我查阅了苹果开发者文档，文档解释如下:

> Do not override `allocWithZone:` to include any initialization code. Instead, class-specific versions of `init...` methods.
> This method exists for historical reasons; memory zones are no longer used by Objective-C.
> 译：不要去重载 `allocWithZone` 并在其内部填充任何初始化代码，相反的，应该在 `init...` 里面进行类的初始化操作。
> 这个方法的存在是有历史原因的，内存 `zone` 已经不再被 `Objective-C` 所使用的。


按照苹果开发者文档的说法，其实 `allocWithZone` 本质上和 `alloc` 是没有区别的，只是在 `Objective-C` 远古时代，程序员需要使用诸如 `allocWithZone` 来优化对象的内存结构，而在当下，其实你写 `alloc` 和 `allocWithZone` 在底层是一模模一样样的。

好的，话题扯远了，我们接着再次进入到 `callAlloc` 方法内部，第二次来到 `callAlloc` 的话，在 `!cls->ISA()->hasCustomAWZ()` 这里判断 `cls` 没有自定义的 `allocWithZone` 实现，这里的判断实质上是对 `cls` 也就是 `object_class` 这一结构体内部的 `class_rw_t` 的 `flags` 与上一个宏 `RW_HAS_DEFAULT_AWZ` 。经过笔者测试，在第一次进入 `callAlloc` 方法内部的时候， `flags` 值为 1 ，然后  `flags` 与上 `1<<16` 结果就是 0 ，返回过去也就是 `false` ，然后在 `hasCustomAWZ` 这里取反之后，返回的就是 `true` ，然后再一取反，自然就会跳过 `if` 里面的逻辑；而第二次进入 `callAlloc` 方法内部的时候， `flags` 值是一个很大的整数，与上 `1<<16` 后结果并不为0 ，所以 `hasDefaultAWZ` 会返回 `true` ，那么 `hasCustomAWZ` 这里就会返回 `false` ，那么返回到 `callAlloc` 的时候自然就会进入 `if` 里面的逻辑了。

> 这里插一句，在我们 OC 的类的结构中，有一个结构叫 `class_rw_t` ，有一个结构叫 `class_ro_t` 。其中 `class_rw_t` 是可以在运行时去拓展类的，包括属性，方法、协议等等，而 `class_ro_t` 则存储了成员变量，属性和方法等，不过这些是在编译时就确定了的，不能在运行时去修改。


```objectivec
 bool hasCustomAWZ() {
    return ! bits.hasDefaultAWZ();
 }   

 bool hasDefaultAWZ() {
 	return data()->flags & RW_HAS_DEFAULT_AWZ;
 }
```

然后我们会来到 `canAllocFast` 的判断，我们继续进入该方法内部

```objectivec
if (fastpath(cls->canAllocFast()))    
```

```objectivec
    bool canAllocFast() {
        assert(!isFuture());
        return bits.canAllocFast();
    }

    bool canAllocFast() {
        return false;
    }
```

结果很显然，这里 `canAllocFast` 是一直返回 `false` 的，也就是说会直接来到下面的逻辑

```objectivec
id obj = class_createInstance(cls, 0);
if (slowpath(!obj)) return callBadAllocHandler(cls);
return obj;            
```

我们再次进入 `class_createInstance` 方法内部

```objectivec
id 
class_createInstance(Class cls, size_t extraBytes)
{
    return _class_createInstanceFromZone(cls, extraBytes, nil);
}

static __attribute__((always_inline)) 
id
_class_createInstanceFromZone(Class cls, size_t extraBytes, void *zone, 
                              bool cxxConstruct = true, 
                              size_t *outAllocatedSize = nil)
{
    // 对 cls 进行判空操作
    if (!cls) return nil;
	// 断言 cls 是否实现了
    assert(cls->isRealized());

    // Read class's info bits all at once for performance
    // cls 是否有 C++ 的初始化构造器
    bool hasCxxCtor = cls->hasCxxCtor();
    // cls 是否有 C++ 的析构器
    bool hasCxxDtor = cls->hasCxxDtor();
    // cls 是否可以分配 Nonpointer，如果是，即代表开启了内存优化 
    bool fast = cls->canAllocNonpointer();
		
    // 这里传入的 extraBytes 为0，然后获取 cls 的实例内存大小
    size_t size = cls->instanceSize(extraBytes);
    // 这里 outAllocatedSize 是默认值 nil，跳过
    if (outAllocatedSize) *outAllocatedSize = size;

    id obj;
    // 这里 zone 传入的也是nil，而 fast 拿到的是 true，所以会进入这里的逻辑
    if (!zone  &&  fast) {
        // 根据 size 开辟内存
        obj = (id)calloc(1, size);
        // 如果开辟失败，返回 nil
        if (!obj) return nil;
        // 将 cls 和是否有 C++ 析构器传入给 initInstanceIsa，实例化 isa
        obj->initInstanceIsa(cls, hasCxxDtor);
    } 
    else {
        // 如果 zone 不为空，经过笔者测试，一般来说调用 alloc 不会来到这里，只有 allocWithZone
        // 或 copyWithZone 会来到下面的逻辑
        if (zone) {
            // 根据给定的 zone 和 size 开辟内存
            obj = (id)malloc_zone_calloc ((malloc_zone_t *)zone, 1, size);
        } else {
            // 根据 size 开辟内存
            obj = (id)calloc(1, size);
        }
        // 如果开辟失败，返回 nil
        if (!obj) return nil;

        // Use raw pointer isa on the assumption that they might be 
        // doing something weird with the zone or RR.
        // 初始化 isa
        obj->initIsa(cls);
    }

    // 如果有 C++ 初始化构造器和析构器，进行优化加速整个流程
    if (cxxConstruct && hasCxxCtor) {
        obj = _objc_constructOrFree(obj, cls);
    }

    // 返回最终的结果
    return obj;
}
```

至此，我们的 `alloc` 流程就探索完毕，但在这其中我们还是有一些疑问点，比如，对象的内存大小时怎么确定出来的， `isa` 是怎么初始化出来的呢，没关系，我们下一篇接着探索。这里，先给出笔者自己画的一个 `alloc` 流程图，限于笔者水平有限，有错误之处望读者指出:

![image.png](https://cdn.nlark.com/yuque/0/2019/png/225346/1577067508691-e125c2a2-0a62-4ff3-b0b6-920682cc1a48.png#align=left&display=inline&height=1430&name=image.png&originHeight=1430&originWidth=1769&size=162905&status=done&style=none&width=1769)
### 2.5 init 简略分析
分析完了 `alloc` 的流程，我们接着分析 `init` 的流程。相比于 `alloc` 来说， `init` 内部实现十分简单，先来到的是 `_objc_rootInit` ，然后就直接返回 `obj` 了。其实这里是一种抽象工厂设计模式的体现，对于 `NSObject` 自带的 `init` 方法来说，其实啥也没干，但是如果你继承于 `NSObject` 的话，然后就可以去重写 `initWithXXX` 之类的初始化方法来做一些初始化操作。

```objectivec
- (id)init {
    return _objc_rootInit(self);
}

id
_objc_rootInit(id obj)
{
    // In practice, it will be hard to rely on this function.
    // Many classes do not properly chain -init calls.
    return obj;
}
```

## 三、总结

先秦荀子的劝学中有言:

> 不积跬步，无以至千里；不积小流，无以成江海。


我们在探索 `iOS` 底层原理的时候，应该也是抱着这样的学习态度，注意点滴的积累，从小做起，积少成多。下一篇笔者将对本文留下的两个疑问进行解答:

- 对象初始化内存是如何分配的？
- isa 是如何初始化的?


