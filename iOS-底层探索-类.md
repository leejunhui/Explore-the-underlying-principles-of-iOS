![iOS 底层探索 - 类](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119023124.jpg)


我们在前面探索了 `iOS` 中的对象原理，面向对象编程中有一句名言:

> 万物皆对象

那么对象又是从哪来的呢？有过面向对象编程基础的同学肯定都知道是类派生出对象的，那么今天我们就一起来探索一下类的底层原理吧。

# `iOS` 中的类到底是什么？

我们在日常开发中大多数情况都是从 `NSObject` 这个基类来派生出我们需要的类。那么在 `OC` 底层，我们的类 `Class` 到底被编译成什么样子了呢？

我们新建一个 `macOS` 控制台项目，然后新建一个 `Animal` 类出来。


```objectivec
// Animal.h
#import <Foundation/Foundation.h>

NS_ASSUME_NONNULL_BEGIN

@interface Animal : NSObject

@end

NS_ASSUME_NONNULL_END

// Animal.m
@implementation Animal

@end

// main.m
#import <Foundation/Foundation.h>
#import "Animal.h"
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        Animal *animal = [[Animal alloc] init];
        NSLog(@"%p", animal);
    }
    return 0;
}
```

我们在终端执行 `clang` 命令:

```bash
clang -rewrite-objc main.m -o main.cpp
```

这个命令是将我们的 `main.m` 重写成 `main.cpp`，我们打开这个文件搜索 `Animal`:

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119022446.jpg)


我们发现有多个地方都出现了 `Animal`:

```cpp
// 1
typedef struct objc_object Animal;

// 2
struct Animal_IMPL {
	struct NSObject_IMPL NSObject_IVARS;
};

// 3
objc_getClass("Animal")
```

我们先全局搜索第一个 `typedef struct objc_object`，发现有 843 个结果

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119022507.jpg)

我们通过 `Command + G` 快捷键快速翻阅一下，最终在 7626 行找到了 `Class` 的定义:

```cpp
typedef struct objc_class *Class;
```

由这行代码我们可以得出一个结论，`Class` 类型在底层是一个结构体类型的指针，这个结构体类型为 `objc_class`。
再搜索 `typedef struct objc_class` 发现搜不出来了，这个时候我们需要在 `objc4-756` 源码中进行探索了。

我们在 `objc4-756` 源码中直接搜索 `struct objc_class` ，然后定位到 `objc-runtime-new.h` 文件

```cpp
struct objc_class : objc_object {
    // Class ISA;
    Class superclass;
    cache_t cache;             // formerly cache pointer and vtable
    class_data_bits_t bits;    // class_rw_t * plus custom rr/alloc flags

    class_rw_t *data() { 
        return bits.data();
    }
}
```

看到这里，细心的读者可能会发现，我们在前面探索对象原理中遇到的 `objc_object` 再次出现了，并且这次是作为 `objc_class` 的父类。这里再次引用那句经典名言 **万物皆对象**，也就是说类其实也是一种**对象**。

由此，我们可以简单总结一下类和对象在 `C` 和 `OC` 中分别的定义

| C | OC |  
| --- | --- | 
| objc_object | NSObject |  
| objc_class | NSObject(Class) |  

# 类的结构是什么样的呢？

通过上面的探索，我们已经知道了类本质上也是对象，而日常开发中常见的成员变量、属性、方法、协议等都是在类里面存在的，那么我们是不是可以猜想在 `iOS` 底层，类其实就存储了这些内容呢？

我们可以通过分析源码来验证我们的猜想。

从上一节中 `objc_class` 的定义处，我们可以梳理出 `Class` 中的 4 个属性

* `isa` 指针
* `superclass` 指针
* `cache`
* `bits`

> 需要值得注意的是，这里的 `isa` 指针在这里是隐藏属性.

## `isa` 指针

首先是 `isa` 指针，我们之前已经探索过了，在对象初始化的时候，通过 `isa` 可以让对象和类关联，这一点很好理解，可是为什么在类结构里面还会有 `isa` 呢？看过上一篇文章的同学肯定知道这个问题的答案了。没错，就是**元类**。我们的对象和类关联起来需要 `isa`，同样的，类和元类之间关联也需要 `isa`。

## `superclass` 指针

顾名思义，`superclass` 指针表明当前类指向的是哪个父类。一般来说，类的根父类基本上都是 `NSObject` 类。根元类的父类也是 `NSObject` 类。

## `cache` 缓存

`cache` 的数据结构为 `cache_t`，其定义如下:

```cpp
struct cache_t {
    struct bucket_t *_buckets;
    mask_t _mask;
    mask_t _occupied;
    
    ...省略代码...
}
```

类的缓存里面存放的是什么呢？是属性？是实例变量？还是方法？我们可以通过阅读 `objc-cache.mm` 源文件来解答这个问题。

> * objc-cache.m
> * Method cache management
> * Cache flushing
> * Cache garbage collection
> * Cache instrumentation
> * Dedicated allocator for large caches

上面是 `objc-cache.mm` 源文件的注释信息，我们可以看到 `Method cache management` 的出现，翻译过来就是方法缓存管理。那么是不是就是说 `cache` 属性就是缓存的方法呢？而 `OC` 中的方法我们现在还没有进行探索，先假设我们已经掌握了相关的底层原理，这里先简单提一下。

> 我们在类里面编写的方法，在底层其实是以 `SEL` + `IMP` 的形式存在。`SEL` 就是方法的选择器，而 `IMP` 则是具体的方法实现。这里可以以书籍的目录以及内容来类比，我们查找一篇文章的时候，需要先知道其标题(`SEL`)，然后在目录中看有没有对应的标题，如果有那么就翻到对应的页，最后我们就找到了我们想要的内容。当然，`iOS` 中方法要比书籍的例子复杂一些，不过暂时可以这么简单的理解，后面我们会深入方法的底层进行探索。

## `bits` 属性

`bits` 的数据结构类型是 `class_data_bits_t`，同时也是一个结构体类型。而我们阅读 `objc_class` 源码的时候，会发现很多地方都有 `bits` 的身影，比如:

```cpp
class_rw_t *data() { 
    return bits.data();
}

bool hasCustomRR() {
    return ! bits.hasDefaultRR();
}    

bool canAllocFast() {
    assert(!isFuture());
    return bits.canAllocFast();
}
```

这里值得我们注意的是，`objc_class` 的 `data()` 方法其实是返回的 `bits` 的 `data()` 方法，而通过这个 `data()` 方法，我们发现诸如类的字节对齐、`ARC`、元类等特性都有 `data()` 的出现，这间接说明 `bits` 属性其实是个大容器，有关于内存管理、C++ 析构等内容在其中有定义。

这里我们会遇到一个十分重要的知识点: `class_rw_t`，`data()` 方法的返回值就是 `class_rw_t` 类型的指针对象。我们在本文后面会重点介绍。

# 类的属性存在哪？

上一节我们对 `OC` 中类结构有了基本的了解，但是我们平时最常打交道的内容-**属性**，我们还不知道它究竟是存在哪个地方。接下来我们要做一件事情，就是在 `objc4-756` 的源码中新建一个 `Target`，为什么不直接用上面的 `macOS` 命令行项目呢？因为我们要开始结合 `LLDB` 打印一些类的内部信息，所以只能是新建一个依靠于 `objc4-756` 源码 `project` 的 `target` 出来。同样的，我们还是选择 `macOS` 的命令行作为我们的 `target`。

接着我们新建一个类 `Person`，然后添加一些实例变量和属性出来。

```objectivec
// Person.h
#import <Foundation/Foundation.h>

NS_ASSUME_NONNULL_BEGIN

@interface Person : NSObject
{
    NSString *hobby;
}
@property (nonatomic, copy) NSString *nickName;
@end

NS_ASSUME_NONNULL_END

// main.m
#import <Foundation/Foundation.h>
#import <objc/runtime.h>
#import "Person.h"
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        Person *p = [[Person alloc] init];
        Class pClass = object_getClass(p);
        NSLog(@"%s", p);
    }
    return 0;
}
```

我们打一个断点到 `main.m` 文件中的 `NSLog` 语句处，然后运行刚才新建的 `target`。

`target` 跑起来之后，我们在控制台先打印输出一下 `pClass` 的内容:

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119022526.jpg)


## 类的内存结构

我们这个时候需要借助指针平移来探索，而对于类的内存结构我们先看下面这张表格:

| 类的内存结构 | 大小(字节) | 
| --- | --- | 
| isa | 8 |  |
| superclass | 8 |  
| cache | 16 |  

前两个大小很好理解，因为 `isa` 和 `superclass` 都是结构体指针，而在 `arm64` 环境下，一个结构体指针的内存占用大小为 8 字节。而第三个属性 `cache` 则需要我们进行抽丝剥茧了。

```Cpp
cache_t cache;

struct cache_t {
    struct bucket_t *_buckets; // 8
    mask_t _mask;  // 4
    mask_t _occupied; // 4
}

typedef uint32_t mask_t; 
```

从上面的代码我们可以看出，`cache` 属性其实是 `cache_t` 类型的结构体，其内部有一个 8 字节的结构体指针，有 2 个各为 4 字节的 `mask_t`。所以加起来就是 16 个字节。也就是说前三个属性总共的内存偏移量为 8 + 8 + 16 = 32 个字节，32 是 10 进制的表示，在 16 进制下就是 20。

## 探索 `bits` 属性

我们刚才在控制台打印输出了 `pClass` 类对象的内容，我们简单画个图如下所示:

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119022603.jpg)


那么，类的 `bits` 属性的内存地址顺理成章的就是在 `isa` 的初始偏移量地址处进行 16 进制下的 20 递增。也就是

```
0x1000021c8 + 0x20 = 0x1000021e8
```

我们尝试打印这个地址，注意这里需要强转一下:

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119022621.jpg)


这里报错了，问题其实是出在我们的 `target` 没有关联上 `libobjc.A.dylib` 这个动态库，我们关联上重新运行项目

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119022633.jpg)


我们重复一遍上面的流程:

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119022639.jpg)


这一次成功了。在 `objc_class` 源码中有:

```cpp
class_rw_t *data() { 
    return bits.data();
}
```

我们不妨打印一下里面的内容：

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119022649.jpg)


返回了一个 `class_rw_t` 指针对象。我们在 `objc4-756` 源码中搜索 `class_rw_t`:

```cpp
struct class_rw_t {
    // Be warned that Symbolication knows the layout of this structure.
    uint32_t flags;
    uint32_t version;

    const class_ro_t *ro;

    method_array_t methods;
    property_array_t properties;
    protocol_array_t protocols;

    Class firstSubclass;
    Class nextSiblingClass;
    
    ...省略代码...    
}
```

显然的，`class_rw_t` 也是一个结构体类型，其内部有 `methods`、`properties`、`protocols` 等我们十分熟悉的内容。我们先猜想一下，我们的属性应该存放在 `class_rw_t` 的 `properties` 里面。为了验证我们的猜想，我们接着进行 `LLDB` 打印:

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119022702.jpg)


我们再接着打印 `properties`:

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119022708.jpg)

`properties` 居然是空的，难道是 bug?其实不然，这里我们还漏掉了一个非常重要的属性 `ro`。我们来到它的定义:

```cpp
struct class_ro_t {
    uint32_t flags;
    uint32_t instanceStart;
    uint32_t instanceSize;
#ifdef __LP64__
    uint32_t reserved;
#endif

    const uint8_t * ivarLayout;
    
    const char * name;
    method_list_t * baseMethodList;
    protocol_list_t * baseProtocols;
    const ivar_list_t * ivars;

    const uint8_t * weakIvarLayout;
    property_list_t *baseProperties;

    ...隐藏代码...    
}
```

`ro` 的类型是 `class_ro_t` 结构体，它包含了 `baseMethodList`、`baseProtocols`、`ivars`、`baseProperties` 等属性。我们刚才在 `class_rw_t` 中没有找到我们声明在 `Person` 类中的实例变量 `hobby` 和属性 `nickName`，那么希望就在 `class_ro_t` 身上了，我们打印看看它的内容:

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119022719.jpg)


根据名称我们猜测属性应该在 `baseProperties` 里面，我们打印看看:

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119022728.jpg)


Bingo! 我们的属性 `nickName` 被找到了，那么我们的实例变量 `hobby` 呢？我们从 $8 的 count 为 1 可以得知肯定不在 `baseProperites` 里面。根据名称我们猜测应该是在 `ivars` 里面。

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119022739.jpg)


哈哈，`hobby` 实例变量也被我们找到了，不过这里的 `count` 为什么是 2 呢？我们打印第二个元素看看:

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119022750.jpg)


结果为 `_nickName`。这一结果证实了编译器会帮助我们给属性 `nickName` 生成一个带下划线前缀的实例变量 `_nickName`。

至此，我们可以得出以下结论:

> `class_ro_t` 是在编译时就已经确定了的，存储的是类的成员变量、属性、方法和协议等内容。
> `class_rw_t` 是可以在运行时来拓展类的一些属性、方法和协议等内容。

# 类的方法存在哪？

研究完了类的属性是怎么存储的，我们再来看看类的方法。

我们先给我们的 `Person` 类增加一个 `sayHello` 的实例方法和一个 `sayHappy` 的类方法。

```objectivec
// Person.h
- (void)sayHello;
+ (void)sayHappy;

// Person.m
- (void)sayHello
{
    NSLog(@"%s", __func__);
}

+ (void)sayHappy
{
    NSLog(@"%s", __func__);
}
```

按照上面的思路，我们直接读取 `class_ro_t` 中的 `baseMethodList` 的内容:

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119022801.jpg)


`sayHello` 被打印出来了，说明 `baseMethodList` 就是存储实例方法的地方。我们接着打印剩下的内容:

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119022806.jpg)

可以看到 `baseMethodList` 中除了我们的实例方法 `sayHello` 外，还有属性 `nickName` 的 `getter` 和 `setter` 方法以及一个 `C++` 析构方法。但是我们的类方法 `sayHappy` 并没有被打印出来。

# 类的类方法存在哪？

我们上面已经得到了属性，实例方法的是怎么样存储，还留下了一个疑问点，就是类方法是怎么存储的，接下来我们用 `Runtime` 的 API 来实际测试一下。

```objectivec
// main.m
void testInstanceMethod_classToMetaclass(Class pClass){
    
    const char *className = class_getName(pClass);
    Class metaClass = objc_getMetaClass(className);
    
    Method method1 = class_getInstanceMethod(pClass, @selector(sayHello));
    Method method2 = class_getInstanceMethod(metaClass, @selector(sayHello));

    Method method3 = class_getInstanceMethod(pClass, @selector(sayHappy));
    Method method4 = class_getInstanceMethod(metaClass, @selector(sayHappy));
    
    NSLog(@"%p-%p-%p-%p",method1,method2,method3,method4);
    NSLog(@"%s",__func__);
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        Person *p = [[Person alloc] init];
        Class pClass = object_getClass(p);
        
        testInstanceMethod_classToMetaclass(pClass);
        NSLog(@"%p", p);
    }
    return 0;
}
```

运行后打印结果如下:

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119022824.jpg)


首先 `testInstanceMethod_classToMetaclass` 方法测试的是分别从类和元类去获取实例方法、类方法的结果。由打印结果我们可以知道：

* 对于类对象来说，`sayHello` 是实例方法，存储于类对象的内存中，不存在于元类对象中。而 `sayHappy` 是类方法，存储于元类对象的内存中，不存在于类对象中。
* 对于元类对象来说，`sayHello` 是类对象的实例方法，跟元类没关系；`sayHappy` 是元类对象的实例方法，所以存在元类中。

我们再接着测试:

```objectivec
// main.m
void testClassMethod_classToMetaclass(Class pClass){
    
    const char *className = class_getName(pClass);
    Class metaClass = objc_getMetaClass(className);
    
    Method method1 = class_getClassMethod(pClass, @selector(sayHello));
    Method method2 = class_getClassMethod(metaClass, @selector(sayHello));

    Method method3 = class_getClassMethod(pClass, @selector(sayHappy));
    Method method4 = class_getClassMethod(metaClass, @selector(sayHappy));
    
    NSLog(@"%p-%p-%p-%p",method1,method2,method3,method4);
    NSLog(@"%s",__func__);
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        Person *p = [[Person alloc] init];
        Class pClass = object_getClass(p);
        
        testClassMethod_classToMetaclass(pClass);
        NSLog(@"%p", p);
    }
    return 0;
}
```

运行后打印结果如下:

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119022834.jpg)


从结果我们可以看出，对于类对象来说，通过 `class_getClassMethod` 获取 `sayHappy` 是有值的，而获取 `sayHello` 是没有值的；对于元类对象来说，通过 `class_getClassMethod` 获取 `sayHappy` 也是有值的，而获取 `sayHello` 是没有值的。这里第一点很好理解，但是第二点会有点让人糊涂，不是说类方法在元类中是体现为对象方法的吗？怎么通过 `class_getClassMethod` 从元类中也能拿到 `sayHappy`，我们进入到 `class_getClassMethod` 方法内部可以解开这个疑惑:

```cpp
Method class_getClassMethod(Class cls, SEL sel)
{
    if (!cls  ||  !sel) return nil;

    return class_getInstanceMethod(cls->getMeta(), sel);
}

Class getMeta() {
    if (isMetaClass()) return (Class)this;
    else return this->ISA();
}    
```

可以很清楚的看到，`class_getClassMethod` 方法底层其实调用的是 `class_getInstanceMethod`，而 `cls->getMeta()` 方法底层的判断逻辑是如果已经是元类就返回，如果不是就返回类的 `isa`。这也就解释了上面的 `sayHappy` 为什么会出现在最后的打印中了。

除了上面的 `LLDB` 打印，我们还可以通过 `isa` 的方式来验证类方法存放在元类中。

* 通过 isa 在类对象中找到元类
* 打印元类的 baseMethodsList

具体的过程笔者不再赘述。

# 类和元类的创建时机


我们在探索类和元类的时候，对于其创建时机还不是很清楚，这里我们先抛出结论：

* 类和元类是在编译期创建的，即在进行 alloc 操作之前，类和元类就已经被编译器创建出来了。

那么如何来证明呢，我们有两种方式可以来证明:

* `LLDB` 打印类和元类的指针

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119022845.jpg)


* 编译项目后，使用 `MachoView` 打开程序二进制可执行文件查看:

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119022852.jpg)


# 总结

* 类和元类创建于编译时，可以通过 `LLDB` 来打印类和元类的指针，或者 `MachOView` 查看二进制可执行文件
* 万物皆对象：类的本质就是对象
* 类在 `class_ro_t` 结构中存储了编译时确定的属性、成员变量、方法和协议等内容。
* 实例方法存放在类中
* 类方法存放在元类中

我们完成了对 `iOS` 中类的底层探索，下一章我们将对类的缓存进行深一步探索，敬请期待~

