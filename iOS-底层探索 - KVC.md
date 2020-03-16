# 一、KVC 初探

`Key Value Coding` 也即 `KVC` 是 `iOS` 开发中一个很重要的概念，中文翻译过来是 `键值编码` ，关于这个概念的具体定义可以在 `Apple` 的[官方文档](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/KeyValueCoding/index.html#//apple_ref/doc/uid/10000107-SW1)处找到。

> Key-value coding is a mechanism enabled by the NSKeyValueCoding informal protocol that objects adopt to provide indirect access to their properties.
> 【译】`KVC` 是通过 `NSKeyValueCoding` 这个非正式协议启用的一种机制，而遵循了这个协议的对象就提供了对其属性的间接访问。
<!-- more -->
我们通常使用访问器方法来访问对象的属性，即使用 `getter` 来获取属性值，使用 `setter` 来设置属性值。而在 `Objective-C` 中，我们还可以直接通过实例变量的方式来获取属性值和设置属性值。如下面的代码所示：

```Objective-C
// JHPerson.h
@interface JHPerson : NSObject
{
    @public
    NSString *myName;
}

@property (nonatomic, copy) NSString *name;
@property (nonatomic, assign) NSInteger age;
@end

// ViewController.m
- (void)viewDidLoad {
    [super viewDidLoad];
    
    JHPerson *person = [[JHPerson alloc] init];
    person.name      = @"leejunhui";
    person.age       = 20;
    person->myName   = @"leejunhui";
    NSLog(@"%@ - %ld - %@",person.name, person.age,person->myName);
}
```

这种方式我们再熟悉不过了，关于属性会由编译器自动生成 `getter` 和 `setter` 以及对应的实例变量前面我们已经探索过了，我们可以在 `ro` 中来找到它们的踪影，感兴趣的读者可以翻阅前面的文章。

> 这里再明确下实例变量、成员变量、属性之间的区别：
> 在 @interface 括号里面声明的变量统称为 **成员变量**
> 而成员变量实际上由两部分组成：**实例变量** + **基本数据类型变量**
> 而**属性** = **成员变量** + **getter方法** + **setter方法**

那其实这里分两种情况，自己实现和编译器帮我们实现。

## 1.1 自己实现 `getter` 和 `setter`


这里我们以 `JHPerson` 类的 `name` 属性为例，我们分别重写 `name` 的 `getter` 和 `setter` 方法，这里还有个注意点，我们需要在 `@interface` 中声明一下实例变量 `_name`，具体代码如下所示：

```Objective-C
// JHPerson.h
@interface JHPerson : NSObject
{
    @public
    NSString *myName;
    NSString *_name;
}

@property (nonatomic, copy) NSString *name;
@property (nonatomic, assign) NSInteger age;
@end

// JHPerson.m
@implementation JHPerson

- (NSString *)name
{
    return _name;
}

- (void)setName:(NSString *)name
{
    _name = name;
}

@end
```

接着，我们在 `main.m` 中使用点语法对 `name` 进行赋值，然后打印 `name` 的值:

```Objective-C
#import <Foundation/Foundation.h>
#import "JHPerson.h"

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        JHPerson *person = [[JHPerson alloc] init];
        person.name      = @"leejunhui";
        NSLog(@"person 姓名为：%@", person.name);
    }
    return 0;
}
```

打印结果如下：

```bash
-[JHPerson setName:] - leejunhui
-[JHPerson name] - leejunhui
person 姓名为：leejunhui
```

显然，这里的结果就表明了 `person.name      = @"leejunhui";` 其实是调用了 `JHPerson` 类的 `setName` 方法，而 `NSLog(@"person 姓名为：%@", person.name);` 则是调用了 `name` 方法。

这块的逻辑我相信读者应该都比较熟悉了，接下来我们再分析编译器自动生成 `getter` 和 `setter` 的场景。

## 1.2 编译器自动实现 `getter` 和 `setter`

我们探索前先思考一个问题，按照我们现在的认知，如果我们不去重写属性的 `getter` 和 `setter` 方法以及声明对应的实例变量，那么编译器就会帮我们做这件事，那么是不是说有多少个属性，就会生成多少个对应的 `getter` 和 `setter` 呢？显然，编译器不会这么傻，这样做不论是从性能上还是设计上都十分笨拙，我们在 `libObjc` 源码中可以找到这么一个源文件：`objc-accessors.mm`，这个文件中有许多从字面意思上看起来像是设置属性的方法，如下图所示：

![img](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200215014508.jpg)


我们聚焦这个方法: `objc_setProperty_nonatomic_copy`，为什么呢？因为 `name` 属性声明为 `@property (nonatomic, copy) NSString *name;`，二者都包含 `nonatomic` 和 `copy` 关键字，我们不妨在 `objc_setProperty_nonatomic_copy` 方法处打上断点，注意，此时我们需要注释掉我们刚才自己添加的 `getter` 和 `setter` 方法。

![img](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200215014453.jpg)

![img](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200215014503.jpg)


Bingo~，`objc_setProperty_nonatomic_copy` 方法果然被调用了，并且我们赋的值也是对的，我们来到这个方法内部实现：

```cpp
void objc_setProperty_nonatomic_copy(id self, SEL _cmd, id newValue, ptrdiff_t offset)
{
    reallySetProperty(self, _cmd, newValue, offset, false, true, false);
}
```

可以看到这里又包裹了一层，真正的实现为 `reallySetProperty`：

这个方法不是很复杂，我们简单过一下这个方法的参数。

> 1.首先是这个方法的 `offset` 参数，前面我们已经探索过关于*内存偏移*的内容，这里不再赘述。我们知道，对象的 `isa` 指针占 `8` 个字节，还寄的我们的 `JHPerson` 类的声明中有一个实例变量 `myName` 吗，这是一个字符串类型的实例变量，也占用 `8` 个字节，所以这里的 `offset` 为 `16`，意思就是偏移 `16` 个字节来设置属性 `name`。
![img](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200215014457.jpg)

> 2.然后是 `atomic` 参数，这个参数取决于属性声明时是 `atomic` 还是 `nonatomic`，这个关键字表示是操作的原子性，而网上很多资料都说 `atomic` 是来保证对象的多线程安全，其实不然，它只是能保证你访问的时候给你返回一个完好无损的 `Value` 而已，[Realm官方对此相关的解释](https://link.jianshu.com/?t=https://realm.io/news/tmi-objective-c-property-attributes/)，举个例子：
> > 如果线程 A 调了 getter，与此同时线程 B 、线程 C 都调了 setter——那最后线程 A get 到的值，有3种可能：可能是 B、C set 之前原始的值，也可能是 B set 的值，也可能是 C set 的值。同时，最终这个属性的值，可能是 B set 的值，也有可能是 C set 的值。所以 `atomic` 并不能保证对象的线程安全。也就是说 `atomic` 所说的线程安全只是保证了`getter` 和 `setter` 存取方法的线程安全，并不能保证整个对象是线程安全的。
> 
> `nonatomic` 关键字就没有这个保证了，`nonatomic` 返回你的对象可能就不是完整的`value` 。因此，在多线程的环境下原子操作是非常必要的，否则有可能会引起错误的结果。但仅仅使用 `atomic` 并不会使得对象线程安全，我们还要为对象线程添加 `lock` 来确保线程的安全。
> 
> **`nonatomic` 对象 `setter` 和 `getter` 方法的实现**:
> ```Objetive-C
> - (void)setCurrentImage:(UIImage *)currentImage
>{
>    if (_currentImage != currentImage) {
>        [_currentImage release];
>        _currentImage = [currentImage retain];
>
>    }
>}
>- (UIImage *)currentImage
>{
>    return _currentImage;
>}

> ```
> 
> **`atomic` 对象 `setter` 和 `getter` 方法的实现**:
> ```Objetive-C
> - (void)setCurrentImage:(UIImage *)currentImage
>{
>    @synchronized(self) {
>        if (_currentImage != currentImage) {
>            [_currentImage release];
>            _currentImage = [currentImage retain];
>
>        }
>    }
>}
>- (UIImage *)currentImage
>{
    @synchronized(self) {
        return _currentImage;
    }
>}
> ```

> 3.最后是 `copy` 和 `mutableCopy` 参数，说到 `copy` 关键字不妨来复习下 `iOS` 中的属性标识符以及相应的变量标识符。


-------


在 `ARC` 中与内存管理有关的变量标识符，有下面几种：
* `__strong`
* `__weak`
* `__unsafe_unretained`
* `__autoreleasing`

| 变量标识符 | 作用 |
| --- | --- |
| `__strong` | 默认使用的标识符。只有还有一个强指针指向某个对象，这个对象就会一直存活 |
| `__weak` | 声明这个引用**不会保持**被引用对象的存活，如果对象没有强引用了，弱引用会被**置为 `nil`** |
| `__unsafe_unretained` | 声明这个引用**不会保持**被引用对象的存活，如果对象没有强引用了，它不**会被置为 nil**。如果它引用的对象被回收掉了，该指针就变成了**野指针** |
| `__autoreleasing` | 用于标示使用引用传值的参数（id *），在函数返回时会被自动释放掉

变量标识符的用法如下：

```Objective-C
Number* __strong num = [[Number alloc] init];
```

注意 `__strong` 的位置应该放到 `*` 和变量名中间，放到其他的位置严格意义上说是不正确的，只不过编译器不会报错。


-------


**属性标识符**

```Objective-C
@property (atomic/nonatomic/assign/retain/strong/weak/unsafe_unretained/copy) Number* num
```

| 属性标识符 | 作用 |
| --- | --- |
| `atomic` | 表明该属性的读写操作是原子性的，但不保证对象的多线程安全 |
| `nonatomic` | 表明该属性的读写操作是非原子性的，性能强于`atomic`，因为没有锁的开销 |
| `assign` | 表明 `setter` 仅仅是一个**简单的赋值操作**，通常用于**基本的数值类型**，例如 `CGFloat` 和 `NSInteger` |
| `strong` | 表明属性定义一个**拥有者关系**。当给属性设定一个新值的时候，首先这个值进行 `retain` ，旧值进行 `release`，然后进行赋值操作 |
| `weak` | 表明属性定义了一个**非拥有者关系**。当给属性设定一个新值的时候，这个值不会进行 `retain`，旧值也不会进行 `release`， 而是进行类似 `assign` 的操作。不过当属性指向的对象被销毁时，该属性会被**置为nil**。 |
| `unsafe_unretained` | 语义和 `assign` 类似，不过是**用于对象类型**的，表示一个非拥有(`unretained`)的，同时也不会在对象被销毁时置为 `nil` 的(`unsafe`)关系。
| `copy` | 类似于 `strong`，不过在赋值时进行 `copy` 操作而不是 `retain` 操作。通常在需要保留某个不可变对象（ `NSString` 最常见），并且**防止它被意外改变**时使用。

> **错误使用属性标识符的后果**
> 如果我们给一个原始类型设置 `strong\weak\copy` ，编译器会直接报错：
> > Property with 'retain (or strong)' attribute must be of object type
> 
> 设置为 `unsafe_unretained` 倒是可以通过编译，只是用起来跟 `assign` 也没有什么区别。
> 反过来，我们给一个 `NSObject` 属性设置为 assign，编译器会报警：
> > Assigning retained object to unsafe property; object will be released after assignment
> 
> 正如警告所说的，对象在赋值之后被立即释放，对应的属性也就成了野指针，运行时跑到属性有关操作会直接崩溃掉。和设置成 `unsafe_unretained` 是一样的效果（设置成 `weak` 不会崩溃）。
> 
> **`unsafe_unretained` 的用处**
> `unsafe_unretained` 差不多是实际使用最少的一个标识符了，在使用中它的用处主要有下面几点：
> 1.兼容性考虑。`iOS4` 以及之前还没有引入 `weak`，这种情况想表达弱引用的语义只能使用 `unsafe_unretained`。这种情况现在已经很少见了。
> 2.性能考虑。使用 `weak` 对性能有一些影响，因此对性能要求高的地方可以考虑使用 `unsafe_unretained` 替换 `weak`。一个例子是 [YYModel 的实现](https://github.com/ibireme/YYModel/blob/master/YYModel/NSObject%2BYYModel.m)，为了追求更高的性能，其中大量使用 `unsafe_unretained` 作为变量标识符。


-------

```cpp
static inline void reallySetProperty(id self, SEL _cmd, id newValue, ptrdiff_t offset, bool atomic, bool copy, bool mutableCopy)
{
    if (offset == 0) {
        object_setClass(self, newValue);
        return;
    }

    id oldValue;
    id *slot = (id*) ((char*)self + offset);

    if (copy) {
        newValue = [newValue copyWithZone:nil];
    } else if (mutableCopy) {
        newValue = [newValue mutableCopyWithZone:nil];
    } else {
        if (*slot == newValue) return;
        newValue = objc_retain(newValue);
    }

    if (!atomic) {
        oldValue = *slot;
        *slot = newValue;
    } else {
        spinlock_t& slotlock = PropertyLocks[slot];
        slotlock.lock();
        oldValue = *slot;
        *slot = newValue;        
        slotlock.unlock();
    }

    objc_release(oldValue);
}
```

* 我们把目光转移到 `reallySetProperty` 中来，这里先判断的 `offset` 是否为 `0`。
    * 如果为 `0`，直接调用方法 `object_setClass` 设置当前对象的 `class`，显然就是设置对象的 `isa` 指针。
* 声明一个临时变量 `oldValue`。
* 将 `self` 先强转为字符串指针，然后进行内存平移得到要设置的属性的内存偏移值，然后将其强转为 `id*` 类型。
* 判断要设置的属性的标识符是否需要进行 `copy` 操作
    * 如果需要，则对传进来的 `newValue` 也就是要设置的属性值发送 `copyWithZone` 消息，**这一步的目的是拿到 `newValue` 的副本，然后覆写 `newValue`，使得传入的 `newValue` 之后再发生了改变都不会影响到属性值**。
* 判断要设置的属性的标识符是否需要进行 `mutableCopy` 操作
    * 如果需要，则对传进来的 `newValue` 也就是要设置的属性值发送 `mutableCopyWithZone` 消息
* 如果要设置的属性既不执行 `copy` 也不执行 `mutableCopy`，那么就先判断要设置的值是否相等
    * 如果相等，说明新值和旧值相等，直接返回
    * 如果不等，则对新值发送 `objc_retain` 消息进行 `retain` 操作，然后将返回值覆写到 `newValue` 上
* 接着判断属性赋值操作是否是原子操作
    * 如果不是原子操作，则将属性赋值给临时变量 `oldValue`，然后将新值赋上去
    * 如果是原子操作，则对赋值操作进行加锁操作保证数据完整性，防止赋值过程中数据发生变化，这也就印证了 `atomic` 是保证属性的读写操作线程安全
    * ![img](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200215014511.jpg)

* 最后对 `oldValue` 也就是旧值进行内存的释放

> PS: **并不是所有属性的自动 `setter` 都会来到 `objc_setProperty`** 
> ![img](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200215014515.jpg)
> 那么，具体是哪些情况下的属性才会来到这里呢？我们不妨做一下简单的测试

```Objective-C
// JHTest.h
@interface JHTest
@property (nonatomic, strong) NSMutableArray *arrayNonatomicAndStrong;
@property (nonatomic, copy)   NSMutableArray *arrayNonatomicAndCopy;
@property (nonatomic, strong) NSString *stringNonatomicAndStrong;
@property (nonatomic, copy)   NSString *stringNonatomicAndCopy;
@property (nonatomic, assign) int ageNonatomicAndAssign;
@property (nonatomic, weak) NSString *stringNonatomicAndWeak;
@property (nonatomic, retain) NSString *stringNonatomicAndRetain;

@property (atomic, strong) NSMutableArray *arrayAtomicAndStrong;
@property (atomic, copy)   NSMutableArray *arrayAtomicAndCopy;
@property (atomic, strong) NSString *stringAtomicAndStrong;
@property (atomic, copy)   NSString *stringAtomicAndCopy;
@property (atomic, assign) int ageAtomicAndAssign;
@property (atomic, weak) NSString *stringAtomicAndWeak;
@property (atomic, retain) NSString *stringAtomicAndRetain;
@end

// main.m
JHTest *test = [[JHTest alloc] init];
NSMutableArray *testMutableArray = @[].mutableCopy;
        
test.arrayNonatomicAndStrong = testMutableArray;
test.arrayNonatomicAndCopy = testMutableArray;
test.stringNonatomicAndStrong = @"呵呵哒";
test.stringNonatomicAndCopy = @"呵呵哒";
test.ageNonatomicAndAssign = 18;
test.stringNonatomicAndWeak = @"呵呵哒";  
test.stringNonatomicAndRetain = @"呵呵哒"; 

test.arrayAtomicAndStrong = testMutableArray;
test.arrayAtomicAndCopy = testMutableArray;
test.stringAtomicAndStrong = @"呵呵哒";
test.stringAtomicAndCopy = @"呵呵哒";
test.ageAtomicAndAssign = 18; 
test.stringAtomicAndWeak = @"呵呵哒";  
test.stringAtomicAndRetain = @"呵呵哒";       
```

我们通过断点调试，每执行到一个属性的时候，看断点是否会来到 `reallySetProperty`，测试结果如下:

| 属性 | 是否进入`reallySetProperty` |
| --- | --- |
| arrayNonatomicAndStrong | 否 |
| arrayNonatomicAndCopy | 是 |
| stringNonatomicAndStrong | 否 |
| stringNonatomicAndCopy | 是 |
| ageNonatomicAndAssign | 否 |
| stringNonatomicAndWeak | 否 |
| stringNonatomicAndRetain | 否 |

| 属性 | 是否进入`reallySetProperty` |
| --- | --- |
| arrayAtomicAndStrong | 是 |
| arrayAtomicAndCopy | 是 |
| stringAtomicAndStrong | 是 |
| stringAtomicAndCopy | 是 |
| ageAtomicAndAssign | 否 | 
| stringAtomicAndWeak | 否 |
| stringAtomicAndRetain | 是 |

从这两组测试结果不难看出，因为 `reallySetProperty` 内部实际上进行了原子性的写操作以及 `copy` 或 `mutableCopy` 的操作和 `retain` 操作，而对于属性标识符为 `nonatomic` 并且非 `copy` 的属性来说，其实并不需要进行原子操作以及 `copy` 或 `mutableCopy` 操作。
我们前面所展示的属性标识符对应作用的内容在这里也印证了只有当属性需要进行 `copy` 或 `mutableCopy` 操作或原子操作时或 `retain` 操作才会被编译器优化来到 `objc_setProperty_xxx => reallySetProperty` 的流程。换句话说，在 `Clang` 编译的时候，编译器肯定会对属性进行判断，对有需要的属性才触发这一流程。

我们用一个表格来总结：


| 底层方法 | 对应属性标识符 |
| --- | --- |
| objc_setProperty_nonatomic_copy | nonatomic + copy |
| objc_setProperty_atomic_copy | atomic + copy |
| objc_setProperty_atomic | atomic + retain/strong |

-------


我们分析完 `reallySetProperty` 后不禁有一个疑问，那就是系统是在哪一步调用了 `objc_setProperty_xxx` 之类的方法呢？答案就是 `LLVM`。我们可以在 `LLVM` 的源码中进行搜索关键字 `objc_setProperty`：

![img](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200215014519.jpg)

我们可以看到在 `clang` 编译器前端的 `RewriteModernObjC` 命名空间下的 `RewritePropertyImplDecl` 方法中：

![img](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200215014523.jpg)

然后我们在 `CodeGen` 目录下的匿名命名空间下的 `ObjcCommonTypesHelper` 的 `getOptimizedSetPropertyFn` 处可以看到以下代码：

![img](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200215014527.jpg)

我们接着以 `getOptimizedSetPropertyFn` 为关键字来搜索：

```cpp
  llvm::FunctionCallee GetOptimizedPropertySetFunction(bool atomic,
                                                       bool copy) override {
    return ObjCTypes.getOptimizedSetPropertyFn(atomic, copy);
  }
```

然后我们搜索 `GetOptimizedPropertySetFunction`：

![img](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200215014536.jpg)

关于 `LLVM` 这块我们先探索到这里，接下来让我们回顾一下 `KVC` 常用的几种使用场景。

# 二、深入 KVC

## 2.1 访问对象属性

1. 通过 `valueForKey:` 和 `setValue:ForKey:` 来**间接的**获取和设置属性值

```Objective-C
        JHPerson *person = [[JHPerson alloc] init];
        [person setValue:@"leejunhui" forKey:@"name"];
        NSLog(@"person 的姓名为: %@", [person valueForKey:@"name"]);
        
        // 打印如下
        person 的姓名为: leejunhui
```

> * `valueForKey`: - Returns the value of a property named by the key parameter. If the property named by the key cannot be found according to the rules described in **Accessor Search Patterns**, then the object sends itself a valueForUndefinedKey: message. The default implementation of valueForUndefinedKey: raises an NSUndefinedKeyException, but subclasses may override this behavior and handle the situation more gracefully.
> 【译】`valueForKey`: 返回由 `key` 参数命名的属性的值。如果根据**访问者搜索模式**中描述的规则找不到由 `key` 命名的属性，则该对象将向自身发送 `valueForUndefinedKey:` 消息。`valueForUndefinedKey:`的默认实现会抛出 `NSUndefinedKeyException` 异常，但是子类可以重写此行为并更优雅地处理这种情况。
> 
> * `setValue:forKey:`: Sets the value of the specified key relative to the object receiving the message to the given value. The default implementation of setValue:forKey: automatically unwraps NSNumber and NSValue objects that represent scalars and structs and assigns them to the property. See Representing Non-Object Values for details on the wrapping and unwrapping semantics.
> If the specified key corresponds to a property that the object receiving the setter call does not have, the object sends itself a setValue:forUndefinedKey: message. The default implementation of setValue:forUndefinedKey: raises an NSUndefinedKeyException. However, subclasses may override this method to handle the request in a custom manner.
> 【译】`setValue:forKey:`: 将该消息接收者的指定 `key` 的值设置为给定值。默认实现会自动把表示标量和结构体的 `NSNumber` 和 `NSValue` 对象解包然后赋值给属性。如果指定 `key` 所对应的属性没有对应的 `setter` 实现，则该对象将向自身发送 `setValue:forUndefinedKey:` 消息，而该消息的默认实现会抛出一个 `NSUndefinedKeyException` 的异常。但是子类可以重写此方法以自定义方式处理请求。

2.`valueForKeyPath:` 和 `setValue:ForKeyPath:`
**Storyboard 或 xib 中使用 KVC**

![img](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200215014545.jpg)

如上图所示，`Storyboard` 中的一个视图的属性菜单可以设置该视图的 `Key Path` ，这就引出了基于**路由**的另外一种 `KVC` 方式，那就是 `valueForKeyPath:` 和 `setValue:ForKeyPath:`

> A key path is a string of dot-separated keys used to specify a sequence of object properties to traverse. The property of the first key in the sequence is relative to the receiver, and each subsequent key is evaluated relative to the value of the previous property. Key paths are useful for drilling down into a hierarchy of objects with a single method call.
> 【译】`keypath` 是一个以点分隔开来的字符串，表示了要遍历的对象属性序列。序列中第一个 `key` 相对于接受者，而后续的每个 `key` 都与前一级 `key` 相关联。`keypath` 对于单个方法调用来深入对象内部结构来说很有用。

通过 `layer.cornerRadius` 这个 `Key Path`，实现了对左侧 `View` 的 `layer` 属性的 `cornerRadius` 属性的访问。

> * `valueForKeyPath:` - Returns the value for the specified key path relative to the receiver. Any object in the key path sequence that is not key-value coding compliant for a particular key—that is, for which the default implementation of valueForKey: cannot find an accessor method—receives a valueForUndefinedKey: message.
>
> 【译】`valueForKeyPath:` : 返回相对于接受者的指定 `key path` 上的值。`key path` 路径序列中不符合特定键的键值编码的任何对象（即 `valueForKey:` 的默认实现无法找到访问器方法的对象）都会接收到 `valueForUndefinedKey:` 消息。
> * `setValue:forKeyPath:` - Sets the given value at the specified key path relative to the receiver. Any object in the key path sequence that is not key-value coding compliant for a particular key receives a setValue:forUndefinedKey: message.
> 
> 【译】`setValue:forKeyPath:`: 将该消息接收者的指定 `key path` 的值设置为给定值。`key path` 路径序列中不符合特定键的键值编码的任何对象都将收到`setValue:forUndefinedKey:` 消息

```Objective-C
// JHPerson.h
@property (nonatomic, strong) JHAccount *account;

// JHAccount.h
@property (nonatomic, copy) NSString *balance;

// main.m
person.account = [[JHAccount alloc] init];
[person setValue:@"666" forKeyPath:@"account.balance"];
NSLog(@"person 的账户余额为: %@", [person valueForKeyPath:@"account.balance"]);

// 打印输出
person 的账户余额为: 666
```


3.`dictionaryWithValuesForKeys:` 和 `setValuesForKeysWithDictionary:`

> * `dictionaryWithValuesForKeys:` - Returns the values for an array of keys relative to the receiver. The method calls valueForKey: for each key in the array. The returned NSDictionary contains values for all the keys in the array.
> 
> 【译】返回相对于接收者的 `key` 数组的值。该方法会为数组中的每个 `key` 调用`valueForKey:`。 返回的 `NSDictionary` 包含数组中所有键的值。
> * `setValuesForKeysWithDictionary:` - Sets the properties of the receiver with the values in the specified dictionary, using the dictionary keys to identify the properties. The default implementation invokes setValue:forKey: for each key-value pair, substituting nil for NSNull objects as required.
> 
> 【译】使用字典键标识属性，然后使用字典中的对应值来设置该消息接收者的属性值。默认实现会对每一个键值对调用 `setValue:forKey:`。设置时需要将 `nil` 替换成 `NSNull`。

```Objective-C
[person setValuesForKeysWithDictionary:@{@"name": @"junhui", @"age": @(18)}];
NSLog(@"%@", [person dictionaryWithValuesForKeys:@[@"name", @"age"]]);       
        
 // 打印输出
{
    age = 18;
    name = junhui;
}       
```

> Collection objects, such as NSArray, NSSet, and NSDictionary, can’t contain nil as a value. Instead, you represent nil values using the NSNull object. NSNull provides a single instance that represents the nil value for object properties. The default implementations of dictionaryWithValuesForKeys: and the related setValuesForKeysWithDictionary: translate between NSNull (in the dictionary parameter) and nil (in the stored property) automatically.
> 集合对象（例如 `NSArray`，`NSSet` 和 `NSDictionary`）不能包含 `nil` 作为值。 而是使用 `NSNull` 对象表示 `nil` 值。`NSNull` 提供了单个实例表示对象属性的nil值。`dictionaryWithValuesForKeys:` 和 `setValuesForKeysWithDictionary:` 的默认实现会自动在 `NSNull`（在 `dictionary` 参数中）和 `nil`（在存储的属性中）之间转换。
> ![img](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200215014549.jpg)

## 2.2 访问集合属性

我们先看下面这样的一份代码，首先给 `JHPerson` 类增加一个属性 `array`，类型为不可变数组，然后修改这个属性：

```Objective-C
// JHPerson.h
@property (nonatomic, strong) NSArray *array;

// main.m
person.array = @[@"1", @"2", @"3"];
NSArray *tempArray = @[@"0", @"1", @"2"];
[person setValue:tempArray forKey:@"array"];
NSLog(@"%@", [person valueForKeyPath:@"array"]);        

// 打印输出
(
    0,
    1,
    2
)
```

虽然这种方式能达到效果，但其实还有一种更好的方式：

```Objective-C
// main.m
NSMutableArray *mutableArray = [person mutableArrayValueForKey:@"array"];
mutableArray[0] = @"-1";
NSLog(@"%@", [person valueForKeyPath:@"array"]);

// 打印输出
 (
    "-1",
    1,
    2
)
```

这里我们用到了一个叫做 `mutableArrayValueForKey:` 的实例方法，这个方法会通过传入的 `key` 返回对应属性的一个可变数组的代理对象。

其实对集合对象来说，我们使用上一节的各种读取和设置方法都可以，但是对于操作集合对象内部的元素来说，更高效的方式是使用 `KVC` 提供的**可变代理方法**。`KVC` 为我们提供了三种不同的可变代理方法：

* `mutableArrayValueForKey:` 和 `mutableArrayValueForKeyPath:`
    * These return a proxy object that behaves like an NSMutableArray object.
    * 【译】返回的代理对象表现为一个 `NSMutableArray` 对象
* `mutableSetValueForKey:` 和 `mutableSetValueForKeyPath:`
    * These return a proxy object that behaves like an NSMutableSet object.
    * 【译】返回的代理对象表现为一个 `NSMutableSet` 对象
* `mutableOrderedSetValueForKey:` and `mutableOrderedSetValueForKeyPath:`
    * These return a proxy object that behaves like an NSMutableOrderedSet object.
    * 【译】返回的代理对象表现为一个 `NSMutableOrderedSet` 对象

## 2.3 集合运算符

在使用 `valueForKeyPath:` 的时候，可以使用集合运算符来实现一些高效的运算操作。

> A collection operator is one of a small list of keywords preceded by an at sign (@) that specifies an operation that the getter should perform to manipulate the data in some way before returning it.
> 【译】一个集合运算符是一小部分关键字其后带有一个at符号（@），该符号指定 `getter` 在返回数据之前以某种方式处理数据应执行的操作。

集合运算符的结构如下图所示：

![img](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200215015436.jpg)

简单解释一下:

* left key path: 指向的要进行运算的集合，如果是直接给集合发送的 `valueForKeyPath:` 消息，`left key path` 可以省略
* right key path: 表示的是对集合中具体哪个属性进行运算操作，除了 `@count` 运算符外，所有的集合运算符的 `right key path` 都不能省略

而集合运算符可以分为三大类：

* 聚合操作符
    * `@avg`: 返回操作对象指定属性的**平均值**
    * `@count`: 返回操作对象指定**属性的个数**
    * `@max`: 返回操作对象指定属性的**最大值**
    * `@min`: 返回操作对象指定属性的**最小值**
    * `@sum`: 返回操作对象指定**属性值之和**
* 数组操作符
    *  `@distinctUnionOfObjects`: 返回操作对象**指定属性的集合--去重** 
    *  `@unionOfObjects`: 返回操作对象**指定属性的集合**
* 嵌套操作符
    * `@distinctUnionOfArrays`: 返回操作对象(嵌套集合)**指定属性的集合--去重**，返回的是 `NSArray`
    * `@unionOfArrays`: 返回操作对象(集合)**指定属性的集合**
    * `@distinctUnionOfSets`: 返回操作对象(嵌套集合)**指定属性的集合--去重**，返回的是 `NSSet`

## 2.4 访问非对象属性

非对象属性分为两类，一类是基本数据类型也就是所谓的**标量**(scalar)，一类是结构体(struct)。

### 2.4.1 访问标量属性

![img](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200215015453.jpg)

如图所示，常用的基本数据类型需要在设置属性的时候包装成 `NSNumber` 类型，然后在读取值的时候使用各自对应的读取方法，如 `double` 类型的标量读取的时候使用 `doubleValue`

### 2.4.2 访问结构体

![img](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200215015441.jpg)

结构体的话就需要转换成 `NSValue` 类型，如上图所示。
除了 `NSPoint`, `NSRange`, `NSRect`, 和 `NSSize`，对于自定义的结构体，也需要进行 `NSValue` 的转换操作，举个🌰:

```Objective-C
typedef struct {
    float x, y, z;
} ThreeFloats;
 
@interface MyClass
@property (nonatomic) ThreeFloats threeFloats;
@end

// 获取结构体属性
NSValue* result = [myClass valueForKey:@"threeFloats"];

// 设置结构体属性
ThreeFloats floats = {1., 2., 3.};
NSValue* value = [NSValue valueWithBytes:&floats objCType:@encode(ThreeFloats)];
[myClass setValue:value forKey:@"threeFloats"];

// 提取结构体属性
ThreeFloats th;
[reslut getValue:&th];
```

## 2.5 属性验证

`KVC` 支持属性验证，而这一特性是通过`validateValue:forKey:error:` (或` validateValue:forKeyPath:error:`) 方法来实现的。这个验证方法的默认实现是去收到这个验证消息的对象(或`keyPath`中最后的对象)中根据 `key` 查找是否有对应的 `validate<Key>:error:` 方法实现，如果没有，验证默认成功，返回 `YES`。
而由于 `validate<Key>:error:` 方法通过引用接收值和错误参数，所以会有以下三种结果：

* 验证成功，返回 `YES`，对属性值不做任何改动。
* 验证失败，返回 `NO`，但对属性值不做改动，如果调用者提供了 `NSError` 的话，就把错误引用设置为指示错误原因的NSError对象。
* 验证失败，返回 `YES`，创建一个新的，有效的属性值作为替代。在返回之前，该方法将值引用修改为指向新值对象。 进行修改时，即使值对象是可变的，该方法也总是创建一个新对象，而不是修改旧对象。

```Objective-C
Person* person = [[Person alloc] init];
NSError* error;
NSString* name = @"John";
if (![person validateValue:&name forKey:@"name" error:&error]) {
    NSLog(@"%@",error);
}
```

那么是否系统会自动进行属性验证呢？
通常，`KVC` 或其默认实现均未定义任何机制来自动的执行属性验证，也就是说需要在适合你的应用的时候自己提供属性验证方法。
某些其他 `Cocoa` 技术在某些情况下会自动执行验证。 例如，保存 `managed object context` 时，`Core Data`会自动执行验证。另外，在 `macOS` 中，`Cocoa Binding`允许你指定验证应自动进行。

## 2.6 `KVC` 取值和设值原理

### 2.6.1 基本 `getter` 

`valueForKey:` 方法会在调用者传入 `key` 之后会在对象中按下列的步骤进行模式搜索：

* 1.以 `get<Key>`, `<key>`, `is<Key>` 以及 `_<key>` 的顺序查找对象中是否有对应的方法。
    * 如果找到了，将方法返回值带上跳转到第 5 步
    * 如果没有找到，跳转到第 2 步
* 2.查找是否有 `countOf<Key>` 和 `objectIn<Key>AtIndex:` 方法(对应于 `NSArray` 类定义的原始方法)以及 `<key>AtIndexes:` 方法(对应于 `NSArray` 方法 `objectsAtIndexes:`)
    * 如果找到其中的第一个(`countOf<Key>`)，再找到其他两个中的至少一个，则创建一个响应所有 `NSArray` 方法的代理集合对象，并返回该对象。(翻译过来就是要么是 `countOf<Key>` + `objectIn<Key>AtIndex:`，要么是 `countOf<Key>` + `<key>AtIndexes:`，要么是 `countOf<Key>` + `objectIn<Key>AtIndex:` + `<key>AtIndexes:`)
    * 如果没有找到，跳转到第 3 步
* 3.查找名为 `countOf<Key>`，`enumeratorOf<Key>` 和 `memberOf<Key>` 这三个方法(对应于NSSet类定义的原始方法）
    * 如果找到这三个方法，则创建一个响应所有 `NSSet` 方法的代理集合对象，并返回该对象
    * 如果没有找到，跳转到第 4 步
* 4.判断类方法 `accessInstanceVariablesDirectly` 结果
    * 如果返回 `YES`，则以 `_<key>`, `_is<Key>`, `<key>`, `is<Key>` 的顺序查找成员变量，如果找到了，将成员变量带上跳转到第 5 步，如果没有找到则跳转到第 6 步
    * 如果返回 `NO`，跳转到第 6 步
* 5.判断取出的属性值
    * 如果属性值是对象，直接返回
    * 如果属性值不是对象，但是可以转化为 `NSNumber` 类型，则将属性值转化为 `NSNumber` 类型返回
    * 如果属性值不是对象，也不能转化为 `NSNumber` 类型，则将属性值转化为 `NSValue` 类型返回
* 6.调用 `valueForUndefinedKey:`。 默认情况下，这会引发一个异常，但是 `NSObject` 的子类可以提供特定于 `key` 的行为。

这里可以用简单的流程图来表示

![img](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200215015445.jpg)

### 2.6.2 基本 `setter`

`setValue:forKey:` 方法默认实现会在调用者传入 `key` 和 `value`(如果是非对象类型，则指的是解包之后的值) 之后会在对象中按下列的步骤进行模式搜索：

* 1.以 `set<Key>:`, `_set<Key>` 的顺序在对象中查找是否有这样的方法，如果找到了，则把属性值传给方法来完成属性值的设置。
* 2.判断类方法 `accessInstanceVariablesDirectly` 结果
    * 如果返回 `YES`，则以 `_<key>`, `_is<Key>`, `<key>`, `is<Key>` 的顺序查找成员变量，如果找到了，则把属性值传给方法来完成属性值的设置。
    * 如果返回 `NO`，跳转到第 3 步
* 3.调用 `setValue：forUndefinedKey:`。 默认情况下，这会引发一个异常，但是`NSObject` 的子类可以提供特定于 `key` 的行为。

![img](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200215015449.jpg)

# 三、自定义 `KVC`

了解了 `KVC` 底层原理之后，我们是否可以自己来实现一下 `KVC` 呢？这里我们要先明确一下 `iOS` 中对于属性的分类：

* **Attributes**: 简单属性，比如基本数据类型，字符串和布尔值，而诸如 `NSNumber` 和其它一些不可变类型比如 `NSColor` 也可以被认为是简单属性
* **To-one relationships**: 这些是具有自己属性的可变对象属性。即对象的属性可以更改，而无需更改对象本身。例如，一个 `Account` 对象可能具有一个 `owner` 属性，该属性是 `Person` 对象的实例，而 `Person` 对象本身具有 `address` 属性。`owner` 的地址可以更改，但却而无需更改 `Account` 持有的 `owner` 属性。也就是说 `Account` 的 `owner` 属性未被更改，只是 `address` 被更改了。
* **To-many relationships**: 这些是集合对象属性。尽管也可以使用自定义集合类，但是通常使用 `NSArray` 或 `NSSet` 的实例来持有此集合。

我们通过代码来演示上述三种类型的属性：

```Objective-C
// Person.h
@interface Person
@property (nonatomic, copy) NSString *name; // Attributes 
@property (nonatomic, strong) Account *account; // To-one relationships
@property (nonatomic, strong) NSArray *subjects; // To-many relationships
@end

// Account.h
@interface Account
@property (nonatomic, assign) NSInteger balance; 
@end
```

我们实现聚焦于最常用的 `valueForKey:` 方法的声明，我们发现该方法是位于 `NSKeyValueCoding` 这个分类里面的，这种设计模式可以实现解耦的功能。

![img](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200215015438.jpg)

打个比方，我们在实际开发中会在 `AppDelegate` 源文件里面去做各种诸如第三方组件的注册和初始化，时间久了，随着项目功能不断迭代，堆积在 `AppDelegate` 中的代码就会越来越多，导致难以维护。这个时候如果采取把这些初始化和注册逻辑放在不同的 `AppDelegate` 的分类中就可以大大减轻 `AppDelegate` 自身维护的成本，同时，也让整个业务流更加清晰。

## 3.1 自定义设值

那么，我们如果要自定义 `KVC` 实现的话，也应该按照这种设计模式来操作。我们直接新建一个 `NSObject` 的分类，然后我们先着眼于 `setValue:ForKey:` 方法，为了避免与系统自带的 `KVC` 方法冲突，我们加一个前缀

```Objective-C
// NSObject+JHKVC.h
@interface NSObject (JHKVC)
- (void)jh_setValue:(nullable id)value forKey:(NSString *)key;
@end
```

然后要实现这个方法，根据我们前面探索的 `setValue:ForKey:` 流程，我们判断一下传入的 `key` 是否为空:

```Objective-C
    // 1.判断 key
    if (key == nil  || key.length == 0) return;
```

* 如果 `key` 为 `nil` 或者 `key` 长度为 0 ，直接退出。

接着我们要判断是否存在 `setKey`，`_setKey`，这里有个小插曲，因为苹果官方文档上只说了这两种方法，但其实，`iOS` 底层还处理了 `setIsKey`，这是因为 `key` 可以被重写成 `isKey` 的形式，所以这里我们就再加上对 `setIsKey` 的判断。

```Objective-C
    // 2.判断 setKey,_setKey,setIsKey 是否存在，如果存在，直接调用相应的方法来设置属性值
    NSString *Key = key.capitalizedString;
    NSString *setKey = [NSString stringWithFormat:@"set%@:",Key];
    NSString *_setKey = [NSString stringWithFormat:@"_set%@:",Key];
    NSString *setIsKey = [NSString stringWithFormat:@"setIs%@:",Key];
    
    if ([self jh_performSelectorWithMethodName:setKey value:value]) {
        NSLog(@"*********%@**********",setKey);
        return;
    }else if ([self jh_performSelectorWithMethodName:_setKey value:value]) {
        NSLog(@"*********%@**********",_setKey);
        return;
    }else if ([self jh_performSelectorWithMethodName:setIsKey value:value]) {
        NSLog(@"*********%@**********",setIsKey);
        return;
    }
```

* 这里为了方便，先将 `key` 进行一下首字母大写化，然后拼接三个不同的 `set` 方法名，然后判断响应的方法能否实现，如果实现了就直接调用响应的方法来设置属性值

> 这里先通过 `respondsToSelector` 来判断当前对象是否能响应传入的方法，如果能响应，则执行方法
> ```Objective-C
> - (BOOL)jh_performSelectorWithMethodName:(NSString *)methodName value:(id)value{
>  
>     if ([self respondsToSelector:NSSelectorFromString(methodName)]) {
>         
> #pragma clang diagnostic push
> #pragma clang diagnostic ignored "-Warc-performSelector-leaks"
>         [self performSelector:NSSelectorFromString(methodName) withObject:value];
> #pragma clang diagnostic pop
>         return YES;
>     }
>     return NO;
> }
> ```

这里如果按照系统的 `KVC` 设值流程，应该还有对 `NSArray`，`NSSet` 之类的处理，为了简化，就暂时忽略掉这些流程。我们直接往下面走，下一个流程应该就是判断类方法 `accessInstanceVariablesDirectly` 了:

```Objective-C
    // 3.判断是否能直接读取成员变量
    if (![self.class accessInstanceVariablesDirectly] ) {
        @throw [NSException exceptionWithName:@"JHUnknownKeyException" reason:[NSString stringWithFormat:@"****[%@ valueForUndefinedKey:]: this class is not key value coding-compliant for the key name.****",self] userInfo:nil];
    }
```

如果可以读取成员变量，那么就需要我们按照 `_key`，`_isKey`, `key`, `isKey` 的顺序去查找了：

```Objective-C
    // 4.按照 _key,is_key,key,isKey 顺序查询实例变量
    NSMutableArray *mArray = [self getIvarListName];
    NSString *_key = [NSString stringWithFormat:@"_%@",key];
    NSString *_isKey = [NSString stringWithFormat:@"_is%@",Key];
    NSString *isKey = [NSString stringWithFormat:@"is%@",Key];
    if ([mArray containsObject:_key]) {
        // 4.2 获取相应的 ivar
       Ivar ivar = class_getInstanceVariable([self class], _key.UTF8String);
        // 4.3 对相应的 ivar 设置值
       object_setIvar(self , ivar, value);
       return;
    }else if ([mArray containsObject:_isKey]) {
       Ivar ivar = class_getInstanceVariable([self class], _isKey.UTF8String);
       object_setIvar(self , ivar, value);
       return;
    }else if ([mArray containsObject:key]) {
       Ivar ivar = class_getInstanceVariable([self class], key.UTF8String);
       object_setIvar(self , ivar, value);
       return;
    }else if ([mArray containsObject:isKey]) {
       Ivar ivar = class_getInstanceVariable([self class], isKey.UTF8String);
       object_setIvar(self , ivar, value);
       return;
    }
```

* 这里要先读取到当前对象上所有的实例变量，然后匹配四种情况

> ```Objective-C
> - (NSMutableArray *)getIvarListName{
>     // 初始化数组容器
>     NSMutableArray *mArray = [NSMutableArray arrayWithCapacity:1];
>     unsigned int count = 0;
>     // 获取到当前类的成员变量
>     Ivar *ivars = class_copyIvarList([self class], &count);
>     // 遍历所有的成员变量
>     for (int i = 0; i<count; i++) {
>         Ivar ivar = ivars[i];
>         const char *ivarNameChar = ivar_getName(ivar);
>         // 将静态字符串指针转换为 NSString 类型  
>         NSString *ivarName = [NSString stringWithUTF8String:ivarNameChar];
>         NSLog(@"ivarName == %@",ivarName);
>         [mArray addObject:ivarName];
>     }
>     // 释放掉成员变量指针数组
>     free(ivars);
>     return mArray;
> }
> ```

这里用到了 `Runtime` 的两个 `api`，`class_copyIvarList` 和 `ivar_getName`
 
> ```Objective-C
> Ivar  _Nonnull * class_copyIvarList(Class cls, unsigned int *outCount);
> ``` 
> 返回类结构中成员变量的指针数组，但是不包括父类中声明的成员变量。该数组包含 `*outCount`指针，后跟一个 `NULL` 终止符。使用完毕后您必须使用 `free()` 释放成员变量的指针数组。如果该类未声明任何实例变量，或者 `cls` 为Nil，则返回 `NULL`，并且 `*outCount` 为 0。
> 
> ```Objective-C
> const char * ivar_getName(Ivar v);
> ```
> 返回成员变量的名称

```Objective-C
    // 5.如果前面的流程都失败了，则抛出异常
    @throw [NSException exceptionWithName:@"JHUnknownKeyException" reason:[NSString stringWithFormat:@"****[%@ %@]: setValue:forUndefinedKey:%@.****",self,NSStringFromSelector(_cmd),key] userInfo:nil];
```

* 最后抛出 `setValue:forUndefinedKey` 的异常

至此，我们的 `setValue:forKey:` 流程就结束了，当然，整个内容和系统真正的 `KVC` 比起来还差得很远，包括线程安全、可变数组之类的都没涉及，不过这不是重点，我们只需要举一反三即可。

## 3.2 自定义取值

接着我们需要自定义的是 `valueForKey:`，我们声明如下的方法：

```Objective-C
- (nullable id)jh_valueForKey:(NSString *)key;
```

然后同样的，根据我们前面探索的 `valueForKey:` 底层流程，还是要先判断 `key`:

```Objective-C
    // 1.判断 key
    if (key == nil  || key.length == 0) {
        return nil;
    }
```

* 如果 `key` 为 `nil` 或者 `key` 长度为 0 ，直接退出。

然后就是判断是否有相应的 `getter` 方法，查找顺序是按照 `getKey`, `key`, `isKey`, `_key`:

```Objective-C
    // 2.判断 getKey,key,isKey,_key 是否存在，如果存在，直接调用相应的方法来返回属性值
    NSString *Key = key.capitalizedString;
    NSString *getKey = [NSString stringWithFormat:@"get%@:",Key];
    NSString *isKey = [NSString stringWithFormat:@"is%@:",Key];
    NSString *_key = [NSString stringWithFormat:@"_%@:",Key];
    
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Warc-performSelector-leaks"
    if ([self respondsToSelector:NSSelectorFromString(getKey)]) {
        return [self performSelector:NSSelectorFromString(getKey)];
    } else if ([self respondsToSelector:NSSelectorFromString(key)]){
        return [self performSelector:NSSelectorFromString(key)];
    } else if ([self respondsToSelector:NSSelectorFromString(isKey)]){
        return [self performSelector:NSSelectorFromString(isKey)];
    } else if ([self respondsToSelector:NSSelectorFromString(_key)]){
        return [self performSelector:NSSelectorFromString(_key)];
    }
#pragma clang diagnostic pop
```

如果这四种 `getter` 方法都没有找到，那么同样的就需要读取类方法：

```Objective-C
    // 3.判断是否能直接读取成员变量
    if (![self.class accessInstanceVariablesDirectly] ) {
        @throw [NSException exceptionWithName:@"JHUnknownKeyException" reason:[NSString stringWithFormat:@"****[%@ valueForUndefinedKey:]: this class is not key value coding-compliant for the key name.****",self] userInfo:nil];
    }
```

如果可以读取成员变量，那么就需要我们按照 `_key`，`_isKey`, `key`, `isKey` 的顺序去查找了：

```Objective-C
    // 4.按照 _key,_iskey,key,isKey 顺序查询实例变量
    NSMutableArray *mArray = [self getIvarListName];
    _key = [NSString stringWithFormat:@"_%@",key];
    NSString *_isKey = [NSString stringWithFormat:@"_is%@",Key];
    isKey = [NSString stringWithFormat:@"is%@",Key];
    if ([mArray containsObject:_key]) {
        Ivar ivar = class_getInstanceVariable([self class], _key.UTF8String);
        return object_getIvar(self, ivar);;
    }else if ([mArray containsObject:_isKey]) {
        Ivar ivar = class_getInstanceVariable([self class], _isKey.UTF8String);
        return object_getIvar(self, ivar);;
    }else if ([mArray containsObject:key]) {
        Ivar ivar = class_getInstanceVariable([self class], key.UTF8String);
        return object_getIvar(self, ivar);;
    }else if ([mArray containsObject:isKey]) {
        Ivar ivar = class_getInstanceVariable([self class], isKey.UTF8String);
        return object_getIvar(self, ivar);;
    }
```

```Objective-C
    // 5.抛出异常
    @throw [NSException exceptionWithName:@"JHUnknownKeyException" reason:[NSString stringWithFormat:@"****[%@ %@]: valueForUndefinedKey:%@.****",self,NSStringFromSelector(_cmd),key] userInfo:nil];
```

* 最后抛出 `valueForUndefinedKey:` 的异常

取值过程的自定义也结束了，其实这里也有不严谨的地方，比如取得属性值返回的时候需要根据属性值类型来判断是否要转换成 `NSNumber` 或 `NSValue`，以及对 `NSArray` 和 `NSSet` 类型的判断。

# 四、总结

`KVC` 探索完了，其实我们探索的大部分内容都是基于苹果的官方文档，我们在探索 `iOS` 底层的时候，文档思维十分重要，有时候说不定在文档的某个角落里就隐藏着追寻的答案。`KVC` 用起来不难，理解起来也不难，但是这不意味着我们可以轻视它。在 `iOS 13` 之前，我们可以通过 `KVC` 去获取和设置系统的私有属性，但从 `iOS 13` 之后，这种方式被禁用掉了。建议对 `KVC` 理解还不透彻的读者去多几遍官方文档，相信我，你会有新的收获。最后，我们简单总结一下本文的内容。

* `KVC` 是一种 `NSKeyValueCoding` 隐式协议所提供的机制。
* `KVC` 通过 `valueForKey:` 和 `valueForKeyPath:` 来取值，不考虑集合类型的话具体的取值过程如下:
    * 以 `get<Key>`, `<key>`, `is<Key>`, `_<key>` 的顺序查找方法
    * 如果找不到方法，则通过类方法 `accessInstanceVariablesDirectly` 判断是否能读取成员变量来返回属性值
    * 以 `_<key>`, `_is<Key>`, `<key>`, `is<Key>` 的顺序查找成员变量
* `KVC` 通过 `setValueForKey:` 和 `setValueForKeyPath:` 来取值，不考虑集合类型的话具体的设置值过程如下:
    * 以 `set<Key>`, `_set<Key>`的顺序查找方法
    * 如果找不到方法，则通过类方法 `accessInstanceVariablesDirectly` 判断是否能通过成员变量来返回设置值
    * 以 `_<key>`, `_is<Key>`, `<key>`, `is<Key>` 的顺序查找成员变量

# 参考资料

[Apple 开发者文档 - KVC](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/KeyValueCoding/index.html#//apple_ref/doc/uid/10000107-SW1)

[iOS atomic 和 nonatomic 的区别](https://www.jianshu.com/p/66b77270e363)

[Objective-C 内存管理](https://hit-alibaba.github.io/interview/iOS/ObjC-Basic/MM.html)

[iOS底层原理总结篇 -- 深入理解 KVC\KVO 实现机制](https://juejin.im/post/5c2189dee51d454517589c8b#heading-13)