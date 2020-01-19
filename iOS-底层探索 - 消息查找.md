![iOS 底层探索 - 消息查找](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119101759.jpg)


# `objc_msgSend` 汇编补充

我们知道，之所以使用汇编来实现 `objc_msgSend` 有两个原因:

- 因为 C 无法通过写一个函数来保留未知的参数并且跳转到一个任意的函数指针。
- `objc_msgSend` 必须足够快。

## `objc_msgSend` 流程

- ENTRY _objc_msgSend
- 对消息接收者进行判断、处理 (id self, sel _cmd)
- taggedPointer 判断处理
- `GetClassFromIsa_p16` isa 指针处理拿到 class
- CacheLookup 查找缓存
- `cache_t` 处理 `bucket` 以及内存哈希处理
  - 找不到递归下一个 `bucket`
  - 找到了就返回 `{imp, sel} = *bucket->imp`\
  - 遇到意外就重试
  - 找不到就跳到 `junpMiss`
- `__objc_msgSend_uncached` 找不到缓存 `imp`
- `STATIC ENTRY __objc_msgSend_uncached`
- `MethodTableLookup` 方法表查找
  - `save parameters registers`
  - `self` 以及 `_cmd` 准备
  - `_class_lookupMethodAndLoadCache3` 调用

# 通过汇编找到下一流程

我们在探索 `objc_msgSend` 的时候，当找不到缓存的时候，会来到一个地方叫做 `objc_msgSend_uncached`，然后会来到 `MethodTableLookup`，然后会有一个核心的查找方法 `__class_lookupMethodAndLoadCache3`。但是我们知道其实已经要进入 C/C++ 的流程了，所以我们还可以汇编来定位。<br />我们打开 `Always Show Disassembly`选项

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119100507.jpg)


然后我们进入 `objc_msgSend` 内部

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119100515.jpg)


然后我们进入 `_objc_msgSend_uncached` 的内部

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119100525.jpg)


我们会来到 `_class_lookupMethodAndLoadCache3`，这就是真正的方法查找实现。


# 代码分析方法查找流程

## 对象方法测试

- 对象的实例方法 - 自己有
- 对象的实例方法 - 自己没有 - 找父类的
- 对象的实例方法 - 自己没有 - 父类也没有 - 找父类的父类 - NSObject
- 对象的实例方法 - 自己没有 - 父类也没有 - 找父类的父类 - NSObject也没有 - 崩溃

## 3.2 类方法测试

- 类方法 - 自己有
- 类方法 - 自己没有 - 找父类的
- 类方法 - 自己没有 - 父类也没有 - 找父类的父类 - NSObject
- 类方法 - 自己没有 - 父类也没有 - 找父类的父类 - NSObject也没有 - 崩溃
- 类方法 - 自己没有 - 父类也没有 - 找父类的父类 - NSObject也没有 - 但是有对象方法


# 源码分析方法查找流程

我们直接定位到 `_class_lookupMethodAndLoadCache3` 源码处:

```c
IMP _class_lookupMethodAndLoadCache3(id obj, SEL sel, Class cls)
{
    return lookUpImpOrForward(cls, sel, obj, 
                              YES/*initialize*/, NO/*cache*/, YES/*resolver*/);
}
```

接着我们进入 `lookUpImpOrForward`，这里注意一下， `cache` 是传的 `NO`，因为来到这里已经说明缓存不存在，所以需要进行方法查找。

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200119100821.jpg)

## lookUpImpOrForward

我们接着定位到 `lookUpImpOrForward` 的源码处：

```c
IMP lookUpImpOrForward(Class cls, SEL sel, id inst, 
                       bool initialize, bool cache, bool resolver)
```

由该方法的参数我们可以知道，`lookUpImpOrForward` 应该是个公共方法，`initialize` 和 `cache` 分别代表是否避免 `+initialize` 和是否从缓存中查找。

```c
// Optimistic cache lookup
    if (cache) {
        imp = cache_getImp(cls, sel);
        if (imp) return imp;
    }
```

- 如果 `cache` 为 `YES`，那么就直接调用 `cache_getImp` 来从 `cls` 的缓存中获取 `sel` 对应的 `IMP`，如果找到了就返回。

```c
if (!cls->isRealized()) {
        realizeClass(cls);
    }
```

- 判断当前要查找的 `cls` 是否已经完成了准备工作，如果没有，则需要进行一下类的 `realize`。

## 从当前类上查找

```c
// Try this class's method lists.
    {
        Method meth = getMethodNoSuper_nolock(cls, sel);
        if (meth) {
            log_and_fill_cache(cls, meth->imp, sel, inst, cls);
            imp = meth->imp;
            goto done;
        }
    }
```

- 上面的方法很显然，是从类的方法列表中查找 `IMP`。这里加两个大括号的目的是形成**局部作用域**，让命名不会不想冲突。通过 `getMethodNoSuper_nolock` 查找 `Method`，找到了之后就调用 `log_and_fill_cache` 进行缓存的填充，然后返回 `imp`。

### getMethodNoSuper_nolock

```c
static method_t *
getMethodNoSuper_nolock(Class cls, SEL sel)
{
    runtimeLock.assertLocked();

    assert(cls->isRealized());
    // fixme nil cls? 
    // fixme nil sel?

    for (auto mlists = cls->data()->methods.beginLists(), 
              end = cls->data()->methods.endLists(); 
         mlists != end;
         ++mlists)
    {
        method_t *m = search_method_list(*mlists, sel);
        if (m) return m;
    }

    return nil;
}

static method_t *search_method_list(const method_list_t *mlist, SEL sel)
{
    int methodListIsFixedUp = mlist->isFixedUp();
    int methodListHasExpectedSize = mlist->entsize() == sizeof(method_t);
    
    if (__builtin_expect(methodListIsFixedUp && methodListHasExpectedSize, 1)) {
        return findMethodInSortedMethodList(sel, mlist);
    } else {
        // Linear search of unsorted method list
        for (auto& meth : *mlist) {
            if (meth.name == sel) return &meth;
        }
    }

#if DEBUG
    // sanity-check negative results
    if (mlist->isFixedUp()) {
        for (auto& meth : *mlist) {
            if (meth.name == sel) {
                _objc_fatal("linear search worked when binary search did not");
            }
        }
    }
#endif

    return nil;
}
```

`getMethodNoSuper_nolock` 实现很简单，就是从 `cls` 的 `data()` 中进行遍历，然后对遍历到的 `method_list_t` 结构体指针再次调用 `search_method_list` 与 `sel` 进行匹配。这里的 `findMethodInSortedMethodList` 我们再接着往下探索。

### findMethodInSortedMethodList

```c
static method_t *findMethodInSortedMethodList(SEL key, const method_list_t *list)
{
    assert(list);

    const method_t * const first = &list->first;
    const method_t *base = first;
    const method_t *probe;
    uintptr_t keyValue = (uintptr_t)key;
    uint32_t count;
    
    for (count = list->count; count != 0; count >>= 1) {
        probe = base + (count >> 1);
        
        uintptr_t probeValue = (uintptr_t)probe->name;
        
        if (keyValue == probeValue) {
            // `probe` is a match.
            // Rewind looking for the *first* occurrence of this value.
            // This is required for correct category overrides.
            while (probe > first && keyValue == (uintptr_t)probe[-1].name) {
                probe--;
            }
            return (method_t *)probe;
        }
        
        if (keyValue > probeValue) {
            base = probe + 1;
            count--;
        }
    }
    
    return nil;
}
```

`findMethodInSortedMethodList` 的核心逻辑是**二分查找**，这种算法的前提是**有序**的集合。

## 从父类中查找

源码如下:

```c
// Try superclass caches and method lists.
    {
        unsigned attempts = unreasonableClassCount();
        for (Class curClass = cls->superclass;
             curClass != nil;
             curClass = curClass->superclass)
        {
            // Halt if there is a cycle in the superclass chain.
            if (--attempts == 0) {
                _objc_fatal("Memory corruption in class list.");
            }
            
            // Superclass cache.
            imp = cache_getImp(curClass, sel);
            if (imp) {
                if (imp != (IMP)_objc_msgForward_impcache) {
                    // Found the method in a superclass. Cache it in this class.
                    log_and_fill_cache(cls, imp, sel, inst, curClass);
                    goto done;
                }
                else {
                    // Found a forward:: entry in a superclass.
                    // Stop searching, but don't cache yet; call method 
                    // resolver for this class first.
                    break;
                }
            }
            
            // Superclass method list.
            Method meth = getMethodNoSuper_nolock(curClass, sel);
            if (meth) {
                log_and_fill_cache(cls, meth->imp, sel, inst, curClass);
                imp = meth->imp;
                goto done;
            }
        }
    }
```

```c
// Superclass cache.
            imp = cache_getImp(curClass, sel);
```

- 在父类中查找的时候，和在当前类查找有一点不同的是需要检查缓存。

```c
if (imp != (IMP)_objc_msgForward_impcache) {
                    // Found the method in a superclass. Cache it in this class.
                    log_and_fill_cache(cls, imp, sel, inst, curClass);
                    goto done;
                }
                else {
                    // Found a forward:: entry in a superclass.
                    // Stop searching, but don't cache yet; call method 
                    // resolver for this class first.
                    break;
                }
```

- 如果在父类中找到了 `IMP`，同时判断是否是消息转发的入口，如果不是消息转发，那么就把找到的 `IMP` 通过 `log_and_fill_cache` 缓存到当前类的缓存中；如果是消息转发，就退出循环。

```c
// Superclass method list.
            Method meth = getMethodNoSuper_nolock(curClass, sel);
            if (meth) {
                log_and_fill_cache(cls, meth->imp, sel, inst, curClass);
                imp = meth->imp;
                goto done;
            }
```

- 如果父类缓存中没有找到，那么就查找父类的方法列表，这里和上面在当前类中的方法列表中查找是异曲同工之妙，就不再赘述了。

## 方法解析

```c
// No implementation found. Try method resolver once.

    if (resolver  &&  !triedResolver) {
        runtimeLock.unlock();
        _class_resolveMethod(cls, sel, inst);
        runtimeLock.lock();
        // Don't cache the result; we don't hold the lock so it may have 
        // changed already. Re-do the search from scratch instead.
        triedResolver = YES;
        goto retry;
    }
```

如果在类和父类中都没有找到，`Runtime` 给了我们一个机会来进行**动态方法解析**。

```c
/***********************************************************************
* _class_resolveMethod
* Call +resolveClassMethod or +resolveInstanceMethod.
* Returns nothing; any result would be potentially out-of-date already.
* Does not check if the method already exists.
**********************************************************************/
void _class_resolveMethod(Class cls, SEL sel, id inst)
{
    if (! cls->isMetaClass()) {
        // try [cls resolveInstanceMethod:sel]

        _class_resolveInstanceMethod(cls, sel, inst);
    } 
    else {
        // try [nonMetaClass resolveClassMethod:sel]
        // and [cls resolveInstanceMethod:sel]
        _class_resolveClassMethod(cls, sel, inst);
        if (!lookUpImpOrNil(cls, sel, inst, 
                            NO/*initialize*/, YES/*cache*/, NO/*resolver*/)) 
        {
            _class_resolveInstanceMethod(cls, sel, inst);
        }
    }
}
```

我们来分析一下 `_class_resolveMethod`:

```c
if (! cls->isMetaClass()) {
        // try [cls resolveInstanceMethod:sel]

        _class_resolveInstanceMethod(cls, sel, inst);
    } 
    else {
        // try [nonMetaClass resolveClassMethod:sel]
        // and [cls resolveInstanceMethod:sel]
        _class_resolveClassMethod(cls, sel, inst);
        if (!lookUpImpOrNil(cls, sel, inst, 
                            NO/*initialize*/, YES/*cache*/, NO/*resolver*/)) 
        {
            _class_resolveInstanceMethod(cls, sel, inst);
        }
    }
```

- 判断当前类是否是元类，如果不是的话，调用 `_class_resolveInstanceMethod`。
- 如果是元类的话，说明要查找的是类方法，调用 `_class_resolveClassMethod`。

### _class_resolveInstanceMethod

首先我们分析动态解析对象方法:

```c
static void _class_resolveInstanceMethod(Class cls, SEL sel, id inst)
{
    if (! lookUpImpOrNil(cls->ISA(), SEL_resolveInstanceMethod, cls, 
                         NO/*initialize*/, YES/*cache*/, NO/*resolver*/)) 
    {
        // Resolver not implemented.
        return;
    }

    BOOL (*msg)(Class, SEL, SEL) = (typeof(msg))objc_msgSend;
    bool resolved = msg(cls, SEL_resolveInstanceMethod, sel);

    // Cache the result (good or bad) so the resolver doesn't fire next time.
    // +resolveInstanceMethod adds to self a.k.a. cls
    IMP imp = lookUpImpOrNil(cls, sel, inst, 
                             NO/*initialize*/, YES/*cache*/, NO/*resolver*/);

    if (resolved  &&  PrintResolving) {
        if (imp) {
            _objc_inform("RESOLVE: method %c[%s %s] "
                         "dynamically resolved to %p", 
                         cls->isMetaClass() ? '+' : '-', 
                         cls->nameForLogging(), sel_getName(sel), imp);
        }
        else {
            // Method resolver didn't add anything?
            _objc_inform("RESOLVE: +[%s resolveInstanceMethod:%s] returned YES"
                         ", but no new implementation of %c[%s %s] was found",
                         cls->nameForLogging(), sel_getName(sel), 
                         cls->isMetaClass() ? '+' : '-', 
                         cls->nameForLogging(), sel_getName(sel));
        }
    }
}

IMP lookUpImpOrNil(Class cls, SEL sel, id inst, 
                   bool initialize, bool cache, bool resolver)
{
    IMP imp = lookUpImpOrForward(cls, sel, inst, initialize, cache, resolver);
    if (imp == _objc_msgForward_impcache) return nil;
    else return imp;
}
```

这里还有一个注意点：

```c
bool resolved = msg(cls, SEL_resolveInstanceMethod, sel);
```

对当前 `cls` 发送 `SEL_resolveInstanceMethod` 消息，如果返回的是 `YES`，那说明当前类是实现了动态方法解析。

由上面的代码可知动态方法解析到最后会回到 `lookUpImpOrForward`。注意这里的传参:<br />`cache` 是 `YES`，`resolver` 是 `NO`，什么意思呢?

> Cache the result (good or bad) so the resolver doesn't fire next time.
> 缓存查找的结果，所以解析器下一次就不会被触发，其实本质上就是**打破递归**。

### _class_resolveClassMethod

我们接着分析动态解析类方法:

```c
static void _class_resolveClassMethod(Class cls, SEL sel, id inst)
{
    assert(cls->isMetaClass());

    if (! lookUpImpOrNil(cls, SEL_resolveClassMethod, inst, 
                         NO/*initialize*/, YES/*cache*/, NO/*resolver*/)) 
    {
        // Resolver not implemented.
        return;
    }

    BOOL (*msg)(Class, SEL, SEL) = (typeof(msg))objc_msgSend;
    bool resolved = msg(_class_getNonMetaClass(cls, inst), 
                        SEL_resolveClassMethod, sel);

    // Cache the result (good or bad) so the resolver doesn't fire next time.
    // +resolveClassMethod adds to self->ISA() a.k.a. cls
    IMP imp = lookUpImpOrNil(cls, sel, inst, 
                             NO/*initialize*/, YES/*cache*/, NO/*resolver*/);

    if (resolved  &&  PrintResolving) {
        if (imp) {
            _objc_inform("RESOLVE: method %c[%s %s] "
                         "dynamically resolved to %p", 
                         cls->isMetaClass() ? '+' : '-', 
                         cls->nameForLogging(), sel_getName(sel), imp);
        }
        else {
            // Method resolver didn't add anything?
            _objc_inform("RESOLVE: +[%s resolveClassMethod:%s] returned YES"
                         ", but no new implementation of %c[%s %s] was found",
                         cls->nameForLogging(), sel_getName(sel), 
                         cls->isMetaClass() ? '+' : '-', 
                         cls->nameForLogging(), sel_getName(sel));
        }
    }
}
```

这里有一个注意点：传进来的 `cls` 必须是**元类**，因为类方法存在元类的缓存或方法列表中。

```c
// 对象方法动态解析
bool resolved = msg(cls, SEL_resolveInstanceMethod, sel);

// 类方法动态解析
bool resolved = msg(_class_getNonMetaClass(cls, inst), 
                        SEL_resolveClassMethod, sel);
```

这里 `msg` 方法的第一个参数就明显不同，解析对象方法的时候传的是当前类，而解析类方法的时候传的是 `_class_getNonMetaClass(cls, inst)` 的结果。我们进入 `_class_getNonMetaClass` 内部:

```c
Class _class_getNonMetaClass(Class cls, id obj)
{
    mutex_locker_t lock(runtimeLock);
    cls = getNonMetaClass(cls, obj);
    assert(cls->isRealized());
    return cls;
}
```

接着进入 `getNonMetaClass`，这个方法的目的就是**通过元类获取类**，我们去除一些干扰信息:

```c
static Class getNonMetaClass(Class metacls, id inst)
{
    static int total, named, secondary, sharedcache;
    realizeClass(metacls);

    total++;

    // 如果已经不是元类的，那就直接返回
    if (!metacls->isMetaClass()) return metacls;

    // metacls really is a metaclass

    // 根元类的特殊情况，这里回忆一下，根元类的isa指向的是自己
    // where inst == inst->ISA() == metacls is possible
    if (metacls->ISA() == metacls) {
        Class cls = metacls->superclass;
        assert(cls->isRealized());
        assert(!cls->isMetaClass());
        assert(cls->ISA() == metacls);
        if (cls->ISA() == metacls) return cls;
    }

    // 如果实例不为空
    if (inst) {
        Class cls = (Class)inst;
        realizeClass(cls);
        // cls 可能是一个子类，这里通过实例获取到类对象，
        // 然后通过一个 while 循环来遍历判断类对象的 isa 是否是元类
        // 如果是元类的话，就跳出循环；如果不是接着获取类对象的父类
        // cls may be a subclass - find the real class for metacls
        while (cls  &&  cls->ISA() != metacls) {
            cls = cls->superclass;
            realizeClass(cls);
        }
        // 说明已经找到了当前元类所匹配的类
        if (cls) {
            assert(!cls->isMetaClass());
            assert(cls->ISA() == metacls);
            return cls;
        }
#if DEBUG
        _objc_fatal("cls is not an instance of metacls");
#else
        // release build: be forgiving and fall through to slow lookups
#endif
    }

    // 尝试命名查询
    {
        Class cls = getClass(metacls->mangledName());
        if (cls->ISA() == metacls) {
            named++;
            if (PrintInitializing) {
                _objc_inform("INITIALIZE: %d/%d (%g%%) "
                             "successful by-name metaclass lookups",
                             named, total, named*100.0/total);
            }

            realizeClass(cls);
            return cls;
        }
    }

    // 尝试 NXMapGet
    {
        Class cls = (Class)NXMapGet(nonMetaClasses(), metacls);
        if (cls) {
            secondary++;
            if (PrintInitializing) {
                _objc_inform("INITIALIZE: %d/%d (%g%%) "
                             "successful secondary metaclass lookups",
                             secondary, total, secondary*100.0/total);
            }

            assert(cls->ISA() == metacls);            
            realizeClass(cls);
            return cls;
        }
    }

    // try any duplicates in the dyld shared cache
    // 尝试从 dyld 动态共享缓存库中查询
    {
        Class cls = nil;

        int count;
        Class *classes = copyPreoptimizedClasses(metacls->mangledName(),&count);
        if (classes) {
            for (int i = 0; i < count; i++) {
                if (classes[i]->ISA() == metacls) {
                    cls = classes[i];
                    break;
                }
            }
            free(classes);
        }

        if (cls) {
            sharedcache++;
            if (PrintInitializing) {
                _objc_inform("INITIALIZE: %d/%d (%g%%) "
                             "successful shared cache metaclass lookups",
                             sharedcache, total, sharedcache*100.0/total);
            }

            realizeClass(cls);
            return cls;
        }
    }

    _objc_fatal("no class for metaclass %p", (void*)metacls);
}
```

## 消息转发

```c
// No implementation found, and method resolver didn't help. 
    // Use forwarding.

    imp = (IMP)_objc_msgForward_impcache;
    cache_fill(cls, sel, imp, inst);
```

如果**动态消息解析**仍然失败，那么就会来到消息查找的最后一步了，**消息转发**。

此时会返回一个类型为 `_objc_msgForward_impcache` 的 `IMP`，然后填充到 `cls` 中的 `cache_t` 里面。至此，我们的消息查找流程就此结束了。

# 总结

- 方法查找或者说消息查找，起始于 `_class_lookupMethodAndLoadCache3`。
- `_class_lookupMethodAndLoadCache3` 的核心实现是 `lookUpImpOrForward`。
- 从 `_class_lookupMethodAndLoadCache3` 进入的话，是忽略缓存直接从方法列表中查找。
- 查找之前会确保类已经完成诸如 属性、方法、协议等内容的 `attach`。
- 先从当前类的方法列表中查找，找到了返回，找不到交给父类。
- 先从父类的缓存中查找，如果找到返回，如果没有查找方法列表，找到了返回，找不到进行**动态方法解析**。
- 根据当前是类还是元类来进行**对象方法动态解析**和**类方法动态解析**。
- 如果解析成功，则返回，如果失败，进入消息转发流程。

我们今天一起探索了消息查找的底层，下一章我们将会沿着今天的方向再往下探索方法转发的流程。敬请期待~