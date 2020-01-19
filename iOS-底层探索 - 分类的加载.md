![iOS 底层探索 - 分类的加载](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119122254.jpg)


# 一、初探懒加载类

上一章我们探索了 `iOS` 中类的加载，让我们简单回顾一下大概的流程。

## 1.1 类的加载回顾

- `libObjc` 向 `dyld` 注册了回调 `_dyld_objc_notify_register`，当 `dyld` 把 `App` 以及 `App` 所依赖的一系列 `Mach-O` 镜像加载到当前 `App` 被分配的内存空间之后，`dyld` 会通过 `_dyld_objc_notify_mapped` 也就是 `map_images` 来通知 `libObjc` 来完成具体的加载工作，`map_images` 被调用之后会来到 `_read_images`
- `_read_images`
  - 主要会进行类的加载工作，会插入 **所有的类** 到 `gdb_objc_realized_classes` 哈希表中（插入方式为 类名为 `key`，类对象为`value`, 不包括通过 _共享缓存_ 里面的类），同时还会把类插入到 `allocatedClasses` 这个集合里面，注意，`allocatedClasses` 的类型为 `NXHashTable`，可以类比为 `NSSet`，而 `gdb_objc_realized_classes` 的类型为 `NXMapTable`，可以类比为 `NSDictionary`
  - 对所有的类进行重映射
  - 将所有的 `SEL` 插入到 `namedSelectors` 哈希表中(插入方式为：`SEL` 名称为 `key`，`SEL` 为`value`)
  - 修复函数指针遗留
  - 将所有的 `Protocol` 插入到 `readProtocol` 哈希表中(插入方式为：`Protocol` 名称为 `key`，`Protocol` 为 `value`)
  - 对所有的 `Protocol` 做重映射
  - 初始化所有的**非懒加载类**，包括 `rw` 和 `ro` 的初始化操作
  - 处理所有的分类(包括类的分类和元类的分类)

## 1.2 验证类的加载流程

我们大致明白了类的加载流程，接下来，让我们在 `_read_images` 源码中打印一下类加载之后的结果验证一下是否加载了我们自己创建的类。

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119120206.jpg)


如上图所示，我们增加一行代码:

```cpp
printf("_getObjc2NonlazyClassList Class:%s\n",cls->mangledName());
```

接着我们观察打印结果:

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119120352.jpg)


忘了提一句，我们这一个有三个类： `LGPerson` 、 `LGStudent` 、 `LGTeacher` 

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119121650.jpg)


但是打印出来的结果没有 `LGPerson` ，这是为什么呢？答案看这里，我们其实是在 `LGStudent` 和 `LGTeacher` 内部实现了 `+load` 方法。而 `LGPerson` 则是没有实现 `+load` 方法。

## 1.3 懒加载类的发现

我们这个时候观察 `_read_images` 源码这部分的注释:

> Realize non-lazy classes (for +load methods and static instances)
> 
> 实现**非懒加载**类(实现了 `+load` 方法和静态实例)


什么意思呢，我们这里其实打印的都是所谓的**非懒加载类**，这里除了我们自己实现了 `+load` 方法的两个类之外，其他的内容都是系统内置的类，包括我们十分熟悉的 `NSObject` 类。通过这里其实反过来推论，我们没有实现 `+load` 方法就是所谓的**懒加载类，这种类并不会在 **`**_read_images**` 环节被加载，那么应该是在哪里加载呢？我们稍微思考一下，我们一般第一次操作一个类是不是在初始化这个类的时候，而初始化类不就是发送 `alloc` 消息吗，而根据我们前面探索消息查找的知识，在第一次发送某个消息的时候，是没有缓存的，所以会来到一个非常重要的方法叫 `lookUpImpOrForward` ，我们在 `main.m` 中 `LGPerson` 类初始化的地方和 `lookUpImpOrForward` 入口处打上断点:

> Tips: 这里有个小技巧，我们先打开 `main.m` 文件中的断点，等断点来到了我们想要探索的 `LGPerson` 初始化的位置的时候，我们再打开 `lookUpImpOrForward` 处的断点，这样才能确保当前执行 `lookUpImpOrForward` 的是我们的研究对象 `LGPerson` 


因为我们断点的位置是 `LGPerson` 类发送 `alloc` 消息，而显然 `alloc` 作为类方法是存储在元类上的，也就是说 `lookUpImpOrForward` 的 `cls` 其实是 `LGPerson` 元类。那么 `inst` 就应该是真正的对象，可实际如下图所示：

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119121717.jpg)


此时的 `inst` 只是一个地址，说明还没有初始化。我们让程序接着下面走，会来到这样一行代码:

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119121724.jpg)


这里的 `if` 判断通过方法名我们不难看出是只有当 `cls` 未实现的时候才会走里面的 `realizeClassMaybeSwiftAndLeaveLocked` 方法，那也就是说 `LGPerson` 元类没有被实现，也就是 `LGPerson` 类没有实现或者说没有被加载。

我们就顺着 `realizeClassMaybeSwiftAndLeaveLocked` 方法往下面走走看，看到底是在哪把我们这个懒加载类给加载出来的:

```cpp
static Class
realizeClassMaybeSwiftMaybeRelock(Class cls, mutex_t& lock, bool leaveLocked)
{
    lock.assertLocked();

    if (!cls->isSwiftStable_ButAllowLegacyForNow()) {
        // Non-Swift class. Realize it now with the lock still held.
        // fixme wrong in the future for objc subclasses of swift classes
        realizeClassWithoutSwift(cls);
        if (!leaveLocked) lock.unlock();
    } else {
        // Swift class. We need to drop locks and call the Swift
        // runtime to initialize it.
        lock.unlock();
        cls = realizeSwiftClass(cls);
        assert(cls->isRealized());    // callback must have provoked realization
        if (leaveLocked) lock.lock();
    }

    return cls;
}
```

我们一路跟随断点来到了 `realizeClassMaybeSwiftMaybeRelock` 方法，然后我们看到了我们熟悉的一个方法 `realizeClassWithoutSwift` ，这个方法内部会进行 `ro/rw` 的赋值操作以及 `category` 的 `attatch` ，关于这个方法更多内容可以查看上一篇文章。

接着我们返回到 `lookUpImpOrForward` 方法中来，然后进行一下 `LLDB` 打印，看一下当前这个 `inst` 也就是 `LGPerson` 对象是否已经被加载了。

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119121740.jpg)


通过上面的打印，我们可以看到 `rw` 已经有值了，也就是说 `LGPerson` 类被加载了。

我们总结一下，如果类没有实现 `load` 方法，那么这个类就是**懒加载类**，其调用堆栈如下图所示：

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119121758.jpg)


反之、这个类如果实现了 `load` 方法，那么这个类就是**非懒加载类**，其调用堆栈如下图所示：

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119121804.jpg)


## 1.4 懒加载类的流程

关于**非懒加载类**的加载流程我们已经很熟悉了，我们总结下**懒加载类**的流程：

- 类第一次发送消息的时候是没有缓存的，所以会来到 `_class_lookupMethodAndLoadCache3` ，关于这个方法我们在前面的消息查找章节已经介绍过了，不熟悉的同学可以去查阅一下。
- `_class_lookupMethodAndLoadCache3` 会调用 `lookUpImpOrForward` ，这个方法的重要性在我们学习 `Runtime` 的过程中不言而喻
- `lookUpImpOrForward` 内部会进行一下判断，如果 `cls` 没有被实现，会调用 `realizeClassMaybeSwiftAndLeaveLocked` 方法
- `realizeClassMaybeSwiftAndLeaveLocked` 方法又会调用 `realizeClassMaybeSwiftMaybeRelock` 方法
- `realizeClassMaybeSwiftMaybeRelock` 方法内部会进行一下是否是 `Swift` 的判断，如果不是 `Swift` 环境的话，就会来到 `realizeClassWithoutSwift` ，也就是最终的类的加载的地方

# 二、分类的底层实现

分类作为 `Objective-C` 中常见的特性，相信大家都不会陌生，不过在底层它是怎么实现的呢？

## 2.1 重写分类源文件

为了探究分类的底层实现，我们只需要用 `clang` 的重写命令

```bash
clang -rewrite-objc LGTeacher+test.m -o category.cpp
```

我们查看 `category.cpp` 这个文件，来到文件尾部可以看到:

```cpp
static struct _category_t _OBJC_$_CATEGORY_LGTeacher_$_test __attribute__ ((used, section ("__DATA,__objc_const"))) = 
{
	"LGTeacher",
	0, // &OBJC_CLASS_$_LGTeacher,
	(const struct _method_list_t *)&_OBJC_$_CATEGORY_INSTANCE_METHODS_LGTeacher_$_test,
	(const struct _method_list_t *)&_OBJC_$_CATEGORY_CLASS_METHODS_LGTeacher_$_test,
	0,
	(const struct _prop_list_t *)&_OBJC_$_PROP_LIST_LGTeacher_$_test,
};
```

我们可以看到 `LGTeacher+test` 分类在底层的实现是一个结构体，其名字为 `_OBJC_$_CATEGORY_LGTeacher_$_test` ，很明显这是一个按规则生成的符号，中间的 `LGTeacher` 是类名，后面的 `test` 是分类的名字。

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119121826.jpg)


我们的分类如上图所示，定义了属性、实例方法和类方法，刚好在底层对应了 

- `_OBJC_$_PROP_LIST_LGTeacher_$_test`
- `_OBJC_$_CATEGORY_INSTANCE_METHODS_LGTeacher_$_test`
- `_OBJC_$_CATEGORY_CLASS_METHODS_LGTeacher_$_test`

同时，我们在后面可以看到如下的代码：

```cpp
static struct _category_t *L_OBJC_LABEL_CATEGORY_$ [1] __attribute__((used, section ("__DATA, __objc_catlist,regular,no_dead_strip")))= {
	&_OBJC_$_CATEGORY_LGTeacher_$_test,
};
```

这表明分类是存储在 `__DATA` 段的 `__objc_catlist` section 里面的。

## 2.2 分类的定义

我们根据 `_category_t` 来到 `libObjc` 源码中进行查找，不过我们需要去掉一下 `_category_t` 的下划线，然后不难找到分类真正的定义所在：

```cpp
struct category_t {
    const char *name;
    classref_t cls;
    struct method_list_t *instanceMethods;
    struct method_list_t *classMethods;
    struct protocol_list_t *protocols;
    struct property_list_t *instanceProperties;
    // Fields below this point are not always present on disk.
    struct property_list_t *_classProperties;

    method_list_t *methodsForMeta(bool isMeta) {
        if (isMeta) return classMethods;
        else return instanceMethods;
    }

    property_list_t *propertiesForMeta(bool isMeta, struct header_info *hi);
};
```

根据刚才 `clang` 重写之后的内容，我们不难看出

- `name` : 是分类所关联的类，也就是类的名字，而不是分类的名字
- `cls` : 我们在前面可以看到 `clang` 重写后这个值为 0，但是后面有注释为 `&OBJC_CLASS_$_LGTeacher` ，也就是我们的类对象的定义，所以这里其实就是我们要扩展的类对象，只是在编译期这个值并不存在
- `instanceMethods` : 分类上存储的实例方法
- `classMethods` ：分类上存储的类方法
- `protocols` ：分类所实现的协议
- `instanceProperties` ：分类所定义的实例属性，不过我们一般在分类中添加属性都是通过关联对象来实现的
- `_classProperties` ：分类所定义的类属性。这里有一行注释：
> Fields below this point are not always present on disk.
> 下面的内容并不是一直在磁盘上保存


也就是说 `_classProperties` 其实是一个私有属性，但并不是一直都存在的。

# 三、分类的加载

我们现在知道了类分为了 `懒加载类` 和 `非懒加载类` ，它们的加载时机是不一样的，那么分类的加载又是怎么样的呢？我们还是同样的先分析没有实现 `load` 方法的分类的情况:

但是我们在分析前，还要搞清楚一点，分类必须依附于类而存在，如果只有分类，没有类，那么从逻辑上是说不通的，就算实现了，编译器也会忽略掉。而关于类是懒加载还是非懒加载的，所以这里我们还要再细分一次。

- 懒加载分类与懒加载类
- 懒加载分类和非懒加载类

## 3.1 没有实现 load 的分类
### 3.1.1 与懒加载类配合加载
我们先分析第一种情况，也就是类和分类都不实现 `load` 方法的情况。<br />首先，非懒加载类的流程上面我们已经探索过了，在向类**第一次发送消息**的时候，非懒加载类才会开始加载，而根据我们上一章类的加载探索内容，在 `realizeClassWithoutSwift` 方法的最后有一个 `methodizeClass` 方法，在这个方法里面会有一个 `Attach categories` 的地方：

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119121859.jpg)


但是我们断点之后发现这个时候通过 `unattachedCategoriesForClass` 方法并没有取到分类，我们此时不妨通过 `LLDB` 打印一下当前类里面是否已经把分类的内容附加上了。<br />前面的流程大家都很熟悉了，我们直接看 `cls` 的 `rw` 中的 `methods` 是否有内容：

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119121906.jpg)


此时 `LGTeacher` 类里面是没有方法的，这里读取 `rw` 却有一个结果，我们不难看出这是位于 `LGTeacher+test` 分类中的一个 `initialize` 方法，这个方法是我手动加个这个分类的。这样进一步证明了，如果是懒加载类，并且分类也是懒加载，那么分类的加载并不会来到 `unattachedCategoriesForClass` ，而是直接在编译时加载到了类的 `ro` 里面，然后在运行时被拷贝到了类的 `rw` 里面。这一点可以通过下面的 `LLDB` 打印来证明。

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119121919.jpg)


如果细心的读者可能会发现，不是在 `_read_images` 的最后那块有一个 `Discover categories` 吗，万一懒加载分类是在这里加载的呢？我们一试便知：

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119121925.jpg)


这里在 `Discover categories` 内部做了一下判断，如果是 `LGTeacher` 类进来了，就打印一下，结果发现并没有打印，说明分类也不是在这里被加载的。  

### 3.1.2 与非懒加载类配合加载

   同样的道理，当类为非懒加载类的时候，走的是 `_read_images` 里面的流程，这个时候我们的懒加载分类是在哪加载的呢？ 

我们直接在 `methodizeClass` 方法中打上断点，并做了一下简单的判断:

```cpp
    const char *cname = ro->name;
    const char *oname = "LGTeacher";
    if (strcmp(cname, oname) == 0) {
       printf("methodizeClass :%s \n",cname);
    }
```

结果可以看到：

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119121934.jpg)

分类还是不在这，同时通过 `LLDB` 打印，发现分类的方法已经在类的 `ro` 里面了，所以说分类的加载其实跟类的懒加载与否并没有关系，也就是说懒加载的分类都是在编译时期被加载的。

## 3.2 实现了 load 的分类
我们再接着分下下面两种情况：

- 非懒加载分类与懒加载类
- 非懒加载分类和非懒加载类

### 3.2.1 与懒加载类配合加载
其实懒加载和非懒加载的最大区别就是加载是否提前，而实现了 `+load` 方法的分类，面对的是懒加载的类，<br />而懒加载的类我们前面已经知道了，是在第一次发送消息的时候才会被加载的，那我们直接在<br />`lookupImpOrForward`  => `realizeClassMaybeSwiftAndLeaveLocked` => `realizeClassMaybeSwiftMaybeRelock` => `realizeClassWithoutSwift` => `methodizeClass` 流程中的 `methodizeClass` 打上断点，看下在这里分类会不会被加载：

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119121942.jpg)


这一次通过 `unattachedCategoriesForClass` 取出来值了，并且在这之前 `cls` 的 `ro` 中并没有分类的 `initialize` 方法：

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119121949.jpg)


但是我们注意观察此时的调用堆栈：

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119121955.jpg)


为什么走的不是发送消息的流程，而走的是 `load_images` 里面的 `prepare_load_methods` 方法呢？我们来到 `prepare_load_methods` 方法处：

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119122001.jpg)


可以看到，其实是在这里调用了 `realizeClassWithoutSwift` 方法来加载类的。而上面的 `_getObjc2NonlazyCategoryList` 方法显示就是获取的所有的非懒加载分类，然后遍历这些非懒加载分类，然后去加载这些分类所依赖的类。这个逻辑很好理解，非懒加载分类让我们的懒加载类实现提前了，所以说懒加载类**并不一定**只会在第一次消息发送的时候加载，还要取决于有没有非懒加载的分类，如果有非懒加载的分类，那么就走的是 `load_images` 里面的 `prepare_load_methods` 的 `realizeClassWithoutSwift` 。


### 3.2.2 与非懒加载类配合加载
非懒加载类的流程我们也十分熟悉了，在 `_read_images` 里面进行加载，而此时，分类也是非懒加载。我们还是在 `methodizeClass` 里面进行断点：

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119122009.jpg)


结果如上图所示，这次从 `unattachedCategoriesForClass` 方法取出来的是 `NULL` 值，显然分类不是在这个地方被加载的，我们回到 `_read_images` 方法，还记得那个 `Discover categories` 流程吗，我们打开里面的断点：

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119122014.jpg)


因为当前类已经在前面的非懒加载类加载流程中被加载完成，所以这里会来到 `remethodizeClass` 方法，我们进入其内部实现：

```cpp
static void remethodizeClass(Class cls)
{
    category_list *cats;
    bool isMeta;

    runtimeLock.assertLocked();

    isMeta = cls->isMetaClass();

    // Re-methodizing: check for more categories
    if ((cats = unattachedCategoriesForClass(cls, false/*not realizing*/))) {
        if (PrintConnecting) {
            _objc_inform("CLASS: attaching categories to class '%s' %s", 
                         cls->nameForLogging(), isMeta ? "(meta)" : "");
        }
        
        attachCategories(cls, cats, true /*flush caches*/);        
        free(cats);
    }
}
```

可以看到有一个 `attachCategories` 方法，断点也确实来到了这个地方， `attachCategories` 方法有一段注释:

> // Attach method lists and properties and protocols from categories to a class.
> // Assumes the categories in cats are all loaded and sorted by load order, 
> // oldest categories first.
> 
> 将分类的方法、属性和协议添加到类上
> 假设传入的分类列表都是按加载顺序加载完毕了
> 先加载的分类排在前面


其实 `attachCategories` 这个方法只会在实现了 `load` 方法的分类下才会被调用，而来到 `attachCategories` 之前又取决于类是否为懒加载，如果是懒加载，那么就在 `load_images` 里面去处理，如果是非懒加载，那么就在 `map_images` 里面去处理。

# 四、总结

我们今天探索的内容可能会有点绕，不过其实探索下来，我们只需要保持研究重点就很简单。分类的加载其实可以笼统的分为实现 `load` 方法和没有实现 `load` 方法：

- 没有实现 `load` 方法的分类由编译时确定
- 实现了 `load` 方法的分类由运行时去确定

这也说明分类的加载和类的加载是不一样的，而结合着类的懒加载与否，我们有以下的结论：

- 懒加载分类 + 懒加载类

> 类的加载在**第一次消息发送**的时候，而分类的加载则在**编译时**


- 懒加载分类 + 非懒加载类

> 类的加载在 `_read_images` 处，分类的加载还是在**编译时**


- 非懒加载分类 + 懒加载类

> 类的加载在 `load_images` 内部，分类的加载在类加载之后的 `methodizeClass` 
> 
> ![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119122041.jpg)



- 非懒加载分类 + 非懒加载类
> 类的加载在 `_read_images` 处，分类的加载在类加载之后的 `reMethodizeClass`
> 
> ![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119122050.jpg)

分类的加载探索完了，我们下一章将探索类拓展和关联对象，敬请期待~


# 五、参考资料

[objc category的秘密 - sunnyxx](http://blog.sunnyxx.com/2014/03/05/objc_category_secret/)


