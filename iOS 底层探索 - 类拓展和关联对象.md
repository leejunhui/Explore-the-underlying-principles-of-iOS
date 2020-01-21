
![iOS 底层探索 - 类拓展_关联对象](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200121143007.jpg)

前面我们探索了 `iOS` 中类和分类的加载，关于类这一块的内容，我们还有一些坑没有填，比如类拓展和关联对象，今天让我们一起填下这块的坑。

# 一、类拓展

## 1.1 什么是类拓展?

关于类拓展的具体定义，大家可以直接参考 [Apple 对于类拓展的说明](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ProgrammingWithObjectiveC/CustomizingExistingClasses/CustomizingExistingClasses.html)。

> A class extension bears some similarity to a category, but it can only be added to a class for which you have the source code at compile time (the class is compiled at the same time as the class extension).
> 
> 类拓展和分类很相似，但是前提是你拥有原始类的源码，并且是在**编译时**被附加到类上的。（类和类扩展同时编译）

类拓展的结构：

```Objective-C
@interface ClassName ()
 
@end
```

> Because no name is given in the parentheses, class extensions are often referred to as anonymous categories.
> 因为括号中没有填写任何内容，所以类扩展也被称为**匿名的分类**。

我们在 `Xcode` 中创建 `Objective` 类型的文件的时候，可以选择空文件、分类、协议以及类扩展。

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200120150028.jpg)

如果我们选择 `Extension` 选项，`Xcode` 会帮我们生成一个 `NSObject + 扩展名` 的头文件出来，也就是说类扩展的命名方式为 `类名_扩展名.h`

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200120150038.jpg)

而这样的操作其实我们很少做，我们一般都是在 `.m` 文件中声明一下当前类的拓展，基本上我们都会在类扩展去声明一些**私有**的属性、方法。比如在 `.h` 文件中声明一个只读的属性，然后在 `.m` 文件的类拓展中去重写这个属性为可读可写。

我们不妨使用 `LLDB` 打印看一下类拓展究竟是不是在编译时就被附加到了类上面了呢？

## 1.2 类拓展是编译时确定的吗？

我们在 `objc-756` 源码中的 `objc-debug` 项目下新建一个类 `Person`，然后给这个类添加一个属性 `name`，然后在 `.m` 文件中的类拓展中添加一个属性 `mName` 和方法 `extM_method`，接着再创建一个 `Person` 的类拓展 `Person+Extension.h` 文件：

```objective-c
// Person.h
#import <Foundation/Foundation.h>

NS_ASSUME_NONNULL_BEGIN

@interface Person : NSObject
@property (nonatomic, copy) NSString *name;
@end

NS_ASSUME_NONNULL_END

// Person.m
#import "Person.h"
#import "Person+Extension.h"

@interface Person ()
@property (nonatomic, copy) NSString *mName;

- (void)extM_method;
@end

@implementation Person

+ (void)load{
    NSLog(@"%s",__func__);
}

- (void)extM_method{
    NSLog(@"%s",__func__);
}

- (void)extH_method{
    NSLog(@"%s",__func__);
}

@end

// Person+Extension.h
#import <AppKit/AppKit.h>
#import "Person.h"

NS_ASSUME_NONNULL_BEGIN

@interface Person ()
@property (nonatomic, copy) NSString *ext_name;
@property (nonatomic, copy) NSString *ext_subject;

- (void)extH_method;
@end

NS_ASSUME_NONNULL_END
```

接着我们在 `main.m` 中来测试一下：

```objective-c
#import <Foundation/Foundation.h>
#import <objc/runtime.h>
#import "Person.h"

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        Person *p = [Person alloc];
        NSLog(@"%@ - %p", p, p);
    }
    return 0;
}
```

我们在 `Person` 实例化对象 `p` 这一行打上断点，然后运行项目。接着在控制台进行 `LLDB` 打印:


> 因为对象的属性以及方法都存储在类对象上面，而由于类结构里面的 `ro` 是编译时就确定了其内容，所以我们只需要打印出类对象的 `ro` 结构中
> 是否有类拓展中的 `mName` 属性和 `extM_method` 方法
> 是否有类扩展中的 `ext_name` 和 `ext_subject` 属性以及 `extH_method` 方法


## 1.3 LLDB 验证

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200120150035.jpg)

* 通过 `x/4gx` 命令打印出 `LGPerson` 类对象的内存地址，以 16 进制方式打印，打印 4 段

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200120150031.jpg)

* 因为类对象的内存地址起始为 `isa`，紧接着是 `superclass`，然后是 `cache_t`。我们前面已经分析过，在默认的 `arm64` 处理器架构下，`isa` 占 8 个字节，`superclass` 占 8 个字节，而 `cache_t` 的三个属性加起来是 8 + 4 + 4 = 16 个字节，所以要想拿到   `bits` 需要进行 8 + 8 + 16 = 32 字节的内存平移，但是这里是 16 进制，所以需要移动 0x20 个内存地址，也就是 `0x100002420 + 0x20 = 0x100002440`

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200120150042.jpg)

* 因为类对象的 `data()` 属性会返回 `bits.data()`，所以这里直接打印刚才取到的 `bits` 的 `data()` 属性，而 `bits` 的 `data()` 属性其实返回的是 `rw`。

```cpp
struct objc_class : objc_object {
    class_rw_t *data() { 
        return bits.data();
    }
}
    
struct class_data_bits_t {
    class_rw_t* data() {
        return (class_rw_t *)(bits & FAST_DATA_MASK);
    }
}    
```

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200120150044.jpg)

* 接着打印 `rw` 的属性 `ro`，然后我们先尝试读取 `baseMethodList` 属性，该属性存储的是编译时确定的类的所有的方法。

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200120150047.jpg)


* 因为 `baseMethodList` 属性是一个 `List` 类型的容器，我们直接使用 `get(index)` 来获取其 `index` 处的值，结果我们所要寻找的 `extH_method` 和 `extM_method` 出现了，不过还没结束，我们还没验证类拓展中声明的两个属性，让我们打印一下 `ro` 的 `baseProperties`

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200120150051.jpg)

* 我们很清楚的看到，`mName`，`ext_name` 和 `ext_subject` 都被找到了，那么是不是就是说类拓展就是编译时确定的了呢？我们还漏掉了这三个属性的 `getter` 和 `setter` 了，让我们回过头再去 `baseMethodList` 中查找一下

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200120150055.jpg)

* Bingo! 我们类拓展定义的属性的 `getter` 和 `setter` 方法也生成了，至此，我们就完全确定了类拓展在编译时就会被加载到类的 `ro` 中。

> 这里有个注意点，就是如果我们没有在类的头文件或者源文件中引入单独的类拓展头文件，那么这个单独的类拓展的头文件里面的属性和方法将不会被加载到类上面来。

## 1.4 类拓展和分类的区别


| 研究对象 | 加载时机 | 操作对象 | 能否通过@property声明属性生成 getter 和 setter |
| --- | --- | --- | --- |
| 分类(实现了load方法) | 运行时 | rw | 不能，需要借助关联对象来实现 |
| 分类(没有实现load方法) | 编译时 | ro | 不能，需要借助关联对象来实现 |
| 类拓展 | 编译时 | ro | 可以 |

# 二、关联对象

上一节我们探索了类拓展以及类拓展与分类的区别，我们知道，类拓展中可以声明属性，编译器会帮助我们生成属性对应的 `getter` 和 `setter` 方法，但是分类通过 `@property` 的方式来声明属性却不能生成 `getter` 和 `setter` 方法。而其实 `iOS` 中有一种方式可以为分为增加具有 `getter` 和 `setter` 的属性，那就是 - 关联对象 `Associated Objects` 。

## 2.1 关联对象定义

关联对象的官方定义可以在 [苹果官方文档](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjectiveC/Chapters/ocAssociativeReferences.html#//apple_ref/doc/uid/TP30001163-CH24) 上找到。

> Associative references, available starting in OS X v10.6, simulate the addition of object instance variables to an existing class. Using associative references, you can add storage to an object without modifying the class declaration. This may be useful if you do not have access to the source code for the class, or if for binary-compatibility reasons you cannot alter the layout of the object.

> 关联引用，是从 `OS X 10.6` 开始启用的，模拟了将对象实例变量添加到已经存在的类中。通过使用关联引用，你可以在不修改类声明的前提下为对象添加内容。如果你无权访问该类的源代码，或者由于二进制兼容性原因而无法更改该对象的布局，则这可能很有用。

> Associations are based on a key. For any object you can add as many associations as you want, each using a different key. An association can also ensure that the associated object remains valid for at least the lifetime of the source object.

> 关联引用机制基于 `key`。对于任何对象，你都可以根据需要添加任意数量的关联引用，每个关联都使用不同的 `key`。关联引用还可以确保关联的对象至少在源对象的声明周期内保持有效。

而关于关联对象的最佳实践可以参考 [NSHipster - Associated Objects](https://nshipster.com/associated-objects/) 一文。

从苹果官方文档可以看到，关联引用其实不是只能在分类中使用，只不过对于我们日常开发来说，分类中使用关联引用还是更常用的场景。相信大多数开发者都知道怎么使用关联引用，的确，关联引用使用起来很简单，不外乎两个方法：

```Objective-C
// 设置关联对象
objc_setAssociatedObject()

// 获取关联对象
objc_getAssociatedObject()
```

我们如果要给一个分类中的属性设置关联对象，需要重写属性的 `setter` 方法，然后使用 `objc_setAssociatedObject`：

```Objective-C
- (void)setXXX:(关联值数据类型)关联值
    objc_setAssociatedObject(self, 关联的key, 关联值, 关联对象内存管理策略);
}
```

然后还需要重写 `getter` 方法，然后使用 `objc_getAssociatedObject`：

```Objective-C
- (关联值数据类型)关联值{
    return objc_getAssociatedObject(self, 关联的key);
}
```

这其中的关联对象内存管理策略如下表所示：


| 关联策略 | 等同的 @property | 描述 |
| --- | --- | --- |
| OBJC_ASSOCIATION_ASSIGN | @property (assign) 或 @property (unsafe_unretained) <span class="Apple-tab-span" style="white-space:pre"></span> 指定一个关联对象的弱引用。 | 指定一个关联对象的弱引用。 |
| OBJC_ASSOCIATION_RETAIN_NONATOMIC | @property (nonatomic, strong) | 指定一个关联对象的强引用，不能被原子化使用。 |
| OBJC_ASSOCIATION_COPY_NONATOMIC | @property (nonatomic, copy) | 指定一个关联对象的copy引用，不能被原子化使用。 |
| OBJC_ASSOCIATION_RETAIN | @property (atomic, strong) | 指定一个关联对象的强引用，能被原子化使用。 |
| OBJC_ASSOCIATION_COPY | @property (atomic, copy) | 指定一个关联对象的copy引用，能被原子化使用。 |

## 2.2 关联对象底层原理

> 关于关联对象的底层原理，这里有一篇灯塔 `draveness` 的博文 [关联对象 AssociatedObject 完全解析](https://draveness.me/ao) 十分值得一读。

当然，如果也可以跟随笔者一起探索下关联对象的底层原理。我们不妨从最直观的 `objc_setAssociatedObject` 方法开始切入：

## 2.3 objc_setAssociatedObject

```Objective-C
// objc-runtime.mm
void objc_setAssociatedObject(id object, const void *key, id value, objc_AssociationPolicy policy) {
    _object_set_associative_reference(object, (void *)key, value, policy);
}
```

`objc_setAssociatedObject` 方法的实现又包裹了一层，其实现为 `_object_set_associative_reference`

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200121141312.jpg)

而 `_object_set_associative_reference` 方法的实现非常长，这里就分段来进行探索吧。

```cpp
    // This code used to work when nil was passed for object and key. Some code
    // probably relies on that to not crash. Check and handle it explicitly.
    // rdar://problem/44094390
    if (!object && !value) return;
```

> 根据注释我们可以知道，当传入的 `object` 和 `key` 同时为 `nil` 的时候，直接返回。这样的处理是为了避免传入空值时而导致崩溃。

```cpp
// objc-references.mm
if (object->getIsa()->forbidsAssociatedObjects())
        _objc_fatal("objc_setAssociatedObject called on instance (%p) of class %s which does not allow associated objects", object, object_getClassName(object));

// objc-runtime-new.h        
bool forbidsAssociatedObjects() {
    return (data()->flags & RW_FORBIDS_ASSOCIATED_OBJECTS);
}
```

> 判断要进行关联的对象是否禁用掉了关联引用，这里是通过对象的 `isa` 的 `rw` 的 `flags` 属性与上一个宏 `RW_FORBIDS_ASSOCIATED_OBJECTS`来判断的。

```cpp
    // retain the new value (if any) outside the lock.
    ObjcAssociation old_association(0, nil);
```

> 初始化一个 `ObjcAssociation` 对象，用于持有原有的关联对象

```cpp
id new_value = value ? acquireValue(value, policy) : nil;
```

> 判断传入的关联对象值是否存在，如果存在就调用 `acquireValue` 方法来获取值，我们可以进入 `acquireValue` 方法看一下:
```cpp
static id acquireValue(id value, uintptr_t policy) {
    switch (policy & 0xFF) {
    case OBJC_ASSOCIATION_SETTER_RETAIN:
        return objc_retain(value);
    case OBJC_ASSOCIATION_SETTER_COPY:
        return ((id(*)(id, SEL))objc_msgSend)(value, SEL_copy);
    }
    return value;
}
```

> 可以看到 `acquireValue` 会根据关联策略来进行 `retain` 或 `copy` 消息的发送

```cpp
        AssociationsManager manager;
        AssociationsHashMap &associations(manager.associations());
        disguised_ptr_t disguised_object = DISGUISE(object);
```

> 初始化一个 `AssociationsManager` 对象，然后获取一个 `AssociationsHashMap` 哈希表，然后通过 `DISGUISE` 方法作为去哈希表查找的 `key`。这里的 `DISGUISE` 其实进行了按位取反的操作。
```cpp
inline disguised_ptr_t DISGUISE(id value) { return ~uintptr_t(value); }
```

如果传入的关联对象值存在，说明是进行赋值操作；如果传入的关联对象值不存在，说明是进行置空操作。这里我们先看一下赋值操作的流程:

```cpp
if (new_value) {
            // break any existing association.
            AssociationsHashMap::iterator i = associations.find(disguised_object);
            if (i != associations.end()) {
                // secondary table exists
                ObjectAssociationMap *refs = i->second;
                ObjectAssociationMap::iterator j = refs->find(key);
                if (j != refs->end()) {
                    old_association = j->second;
                    j->second = ObjcAssociation(policy, new_value);
                } else {
                    (*refs)[key] = ObjcAssociation(policy, new_value);
                }
            } else {
                // create the new association (first time).
                ObjectAssociationMap *refs = new ObjectAssociationMap;
                associations[disguised_object] = refs;
                (*refs)[key] = ObjcAssociation(policy, new_value);
                object->setHasAssociatedObjects();
            }
        }
```

> 1.通过上一步按位取反之后的结果，在 `AssociationsHashMap` 哈希表中查询，这里是通过迭代器的方式进行查询，查询的结果是 `ObjcAssociation` 对象，这个结构也是一个哈希表，其内部存储的是 `_object_set_associative_reference` 方法传入的 `key` 为键，`ObjcAssociation` 对象为值的键值对
> 2.如果没有查询到，说明之前在当前类上**没有设置过**关联对象。则需要初始化一个 `ObjectAssociationMap` 出来，然后通过 `setHasAssociatedObjects` 设置当前对象的 `isa` 的 `has_assoc` 属性为 `true`
> 3.如果查询到了，说明之前在当前类上**设置过**关联对象，接着需要看 `key` 是否存在，如果 `key` 存在，那么就需要更新原有的关联对象；如果 `key` 不存在，则需要新增一个关联对象

```cpp
            // setting the association to nil breaks the association.
            AssociationsHashMap::iterator i = associations.find(disguised_object);
            if (i !=  associations.end()) {
                ObjectAssociationMap *refs = i->second;
                ObjectAssociationMap::iterator j = refs->find(key);
                if (j != refs->end()) {
                    old_association = j->second;
                    refs->erase(j);
                }
            }
```

> 因为来到这里的条件是 `new_value` 为 `nil`，也就代表着要删除关联对象，内部的逻辑和上面的流程大同小异，不过最后多了一步在 `ObjectAssociationMap` 擦除 `key` 对应的节点

```cpp
    // release the old value (outside of the lock).
    if (old_association.hasValue()) ReleaseValue()(old_association);
```

> 最后会判断 `old_association` 是否有值，如果有的话就释放掉，当然前提是旧的关联对象的策略是 `OBJC_ASSOCIATION_SETTER_RETAIN`
> ```cpp
> struct ReleaseValue {
    void operator() (ObjcAssociation &association) {
        releaseValue(association.value(), association.policy());
    }
};
static void releaseValue(id value, uintptr_t policy) {
    if (policy & OBJC_ASSOCIATION_SETTER_RETAIN) {
        return objc_release(value);
    }
}
> ```

## 2.4 objc_getAssociatedObject

`objc_setAssociatedObject` 方法分析完了，我们接着看另外一个重要的方法 `objc_getAssociatedObject`:

```cpp
id objc_getAssociatedObject(id object, const void *key) {
    return _object_get_associative_reference(object, (void *)key);
}
```

> 可以看到，跟 `objc_setAssociatedObject` 一样，`objc_getAssociatedObject` 这里又包裹了一层，其实现为 `_object_get_associative_reference`，而这个方法相比于上一节的 `_object_set_associative_reference` 要简单一些，我们就直接贴出完整的代码

```cpp
id _object_get_associative_reference(id object, void *key) {
    id value = nil;
    uintptr_t policy = OBJC_ASSOCIATION_ASSIGN;
    {
        AssociationsManager manager;
        AssociationsHashMap &associations(manager.associations());
        disguised_ptr_t disguised_object = DISGUISE(object);
        AssociationsHashMap::iterator i = associations.find(disguised_object);
        if (i != associations.end()) {
            ObjectAssociationMap *refs = i->second;
            ObjectAssociationMap::iterator j = refs->find(key);
            if (j != refs->end()) {
                ObjcAssociation &entry = j->second;
                value = entry.value();
                policy = entry.policy();
                if (policy & OBJC_ASSOCIATION_GETTER_RETAIN) {
                    objc_retain(value);
                }
            }
        }
    }
    if (value && (policy & OBJC_ASSOCIATION_GETTER_AUTORELEASE)) {
        objc_autorelease(value);
    }
    return value;
}
```

> 1.先初始化一个空的 `value`，以及一个策略为 `OBJC_ASSOCIATION_ASSIGN` 的 `policy`
> 2.初始化一个 `AssociationsManager` 关联对象管理类，接着拿到 `AssociationsHashMap` 对象，这个对象在 `AssociationsManager` 底层是静态的
> 3.然后以 `DISGUISE(object)` 按位取反之后的结果为键去查询 `AssociationsHashMap`
> 4.如果在 `AssociationsHashMap` 中扎到了，接着以 `key` 为键去 `ObjectAssociationMap` 中查询 `ObjcAssociation`
> 如果在 `ObjectAssociationMap` 中查询到了 `ObjcAssociation`，则把值和策略赋值给方法入口声明的两个临时变量，然后判断获取到的关联对象的策略是否为 `OBJC_ASSOCIATION_GETTER_RETAIN`，如果是的话，需要对关联值进行 `retain` 操作
> 5.最后判断如果关联值是否存在且策略为 `OBJC_ASSOCIATION_GETTER_AUTORELEASE`，是的话就需要调用 `objc_autorelease` 来释放关联值
> 6.最后返回关联值

## 2.5 objc_removeAssociatedObjects

`objc_removeAssociatedObjects` 方法我们平时可能用的不多，从字面含义来看，这个方法应该是用来删除关联对象。我们来到它的定义处：

```cpp
void objc_removeAssociatedObjects(id object) 
{
    if (object && object->hasAssociatedObjects()) {
        _object_remove_assocations(object);
    }
}
```

> 这里会判断 `object` 存在且有关联对象才会进入真正的实现 `_object_remove_assocations`，该实现也不是很复杂，我们还是直接贴出代码

```cpp
void _object_remove_assocations(id object) {
    vector< ObjcAssociation,ObjcAllocator<ObjcAssociation> > elements;
    {
        AssociationsManager manager;
        AssociationsHashMap &associations(manager.associations());
        if (associations.size() == 0) return;
        disguised_ptr_t disguised_object = DISGUISE(object);
        AssociationsHashMap::iterator i = associations.find(disguised_object);
        if (i != associations.end()) {
            // copy all of the associations that need to be removed.
            ObjectAssociationMap *refs = i->second;
            for (ObjectAssociationMap::iterator j = refs->begin(), end = refs->end(); j != end; ++j) {
                elements.push_back(j->second);
            }
            // remove the secondary table.
            delete refs;
            associations.erase(i);
        }
    }
    // the calls to releaseValue() happen outside of the lock.
    for_each(elements.begin(), elements.end(), ReleaseValue());
}
```

> 这里会将对象包含的所有关联对象加入到一个 `vector` 中，然后对所有的 `ObjcAssociation` 对象调用 `ReleaseValue()` 方法，释放不再被需要的值。

# 三、总结

* 类拓展是一种匿名的分类，加载时机为编译时
* 类拓展可以添加属性和方法以及实例变量，分类只能添加方法，属性，但是需要借助关联对象来生成 `getter` 和 `setter`，而且分类不能声明实例变量
* 关联对象在底层其实是 `ObjcAssociation` 对象的结构
* 全局有一个 `AssociationsManager` 管理类存储了一个静态的哈希表 `AssociationsHashMap`，这个哈希表存储的是以对象指针为键，以该对象所有的关联对象为值，而对象所有的关联对象又是以 `ObjectAssociationMap` 来存储的
* `ObjectAssociationMap` 存储结构为 `key` 为键，`ObjcAssociation` 为值
* 快速判断一个对象是否存在关联对象，可以直接取对象 `isa` 的 `has_assoc`

# 四、参考资料

[Apple - 类拓展](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ProgrammingWithObjectiveC/CustomizingExistingClasses/CustomizingExistingClasses.html)

[Apple - 关联对象](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjectiveC/Chapters/ocAssociativeReferences.html#//apple_ref/doc/uid/TP30001163-CH24) 

[NSHipster - Associated Objects](https://nshipster.com/associated-objects/) 

[Draveness - 关联对象 AssociatedObject 完全解析](https://draveness.me/ao)
