# iOS 查漏补缺 - PerformSelector

> `performSelector` 系列的函数我们都不陌生，但是对于它不同的变种以及底层原理在很多时候还是容易分不清楚，所以笔者希望通过 `runtime` 源码以及 `GUNStep` 源码来一个个抽丝剥茧，把不同变种的 `performSelector` 理顺，并搞清楚每个方法的底层实现，如有错误，欢迎指正。
> 
> 本文的代码已放在 [Github](https://github.com/LeeJunhui/PerformSelectorIndepth) ，欢迎自取
<!-- more -->

# NSObject 下的 `PerformSelector`

* `performSelector:(SEL)aSelector` 

`performSelector` 方法是最简单的一个 `api`，使用方法如下

```objectivec
- (void)jh_performSelector
{
     [self performSelector:@selector(task)];
}

- (void)task
{
    NSLog(@"%s", __func__);
}

// 输出
2020-03-12 11:13:26.321254+0800 PerformSelectorIndepth[61807:828757] -[ViewController task]
```

`performSelector:` 方法只需要传入一个 `SEL`，在 `runtime` 底层实现为：

```objectivec
- (id)performSelector:(SEL)sel {
    if (!sel) [self doesNotRecognizeSelector:sel];
    return ((id(*)(id, SEL))objc_msgSend)(self, sel);
}
```


-------

* `performSelector:(SEL)aSelector withObject:(id)object` 

`performSelector:withObject:` 方法相比于上一个方法多了一个参数，使用起来如下：

```objectivec
- (void)jh_performSelectorWithObj
{
    [self performSelector:@selector(taskWithParam:) withObject:@{@"param": @"leejunhui"}];
}

- (void)taskWithParam:(NSDictionary *)param
{
    NSLog(@"%s", __func__);
    NSLog(@"%@", param);
}

// 输出
2020-03-12 11:12:34.473153+0800 PerformSelectorIndepth[61790:827408] -[ViewController taskWithParam:]
2020-03-12 11:12:34.473381+0800 PerformSelectorIndepth[61790:827408] {
    param = leejunhui;
}
```

`performSelector:withObject:` 方法底层实现如下:

```objectivec
- (id)performSelector:(SEL)sel withObject:(id)obj {
    if (!sel) [self doesNotRecognizeSelector:sel];
    return ((id(*)(id, SEL, id))objc_msgSend)(self, sel, obj);
}
```


-------


* `performSelector:(SEL)aSelector withObject:(id)object1 withObject:(id)object2` 

这个方法相比上一个方法又多了一个参数:

```objectivec
- (void)jh_performSelectorWithObj1AndObj2
{
    [self performSelector:@selector(taskWithParam1:param2:) withObject:@{@"param1": @"lee"} withObject:@{@"param2": @"junhui"}];
}

- (void)taskWithParam1:(NSDictionary *)param1 param2:(NSDictionary *)param2
{
    NSLog(@"%s", __func__);
    NSLog(@"%@", param1);
    NSLog(@"%@", param2);
}

// 输出
2020-03-12 11:17:52.889731+0800 PerformSelectorIndepth[61859:833076] -[ViewController taskWithParam1:param2:]
2020-03-12 11:17:52.889921+0800 PerformSelectorIndepth[61859:833076] {
    param1 = lee;
}
2020-03-12 11:17:52.890009+0800 PerformSelectorIndepth[61859:833076] {
    param2 = junhui;
}
```

`performSelector:withObject:withObject:` 方法底层实现如下:

```objectivec
- (id)performSelector:(SEL)sel withObject:(id)obj1 withObject:(id)obj2 {
    if (!sel) [self doesNotRecognizeSelector:sel];
    return ((id(*)(id, SEL, id, id))objc_msgSend)(self, sel, obj1, obj2);
}
```


-------


| 方法 | 底层实现 |
| --- | --- |
| performSelector: | ((id(*)(id, SEL))objc_msgSend)(self, sel) |
| performSelector:withObject: | ((id(*)(id, SEL, id))objc_msgSend)(self, sel, obj) |
| performSelector:withObject:withObject: | ((id(*)(id, SEL, id, id))objc_msgSend)(self, sel, obj1, obj2) |

这三个方法应该是使用频率很高的 `performSelector` 系列方法了，我们只需要记住这三个方法在底层都是执行的**消息发送**即可。


-------


# Runloop 相关的 `PerformSelector`

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200312163817.jpg)

如上图所示，在 `NSRunLoop` 头文件中，定义了两个的分类，分别是 

* `NSDelayedPerforming` 对应于 `NSObject` 
* `NSOrderedPerform` 对应于 `NSRunLoop`

## NSObject 分类 NSDelayedPerforming 

* `performSelector:WithObject:afterDelay:`

```objectivec
- (void)jh_performSelectorwithObjectafterDelay
{
    [self performSelector:@selector(taskWithParam:) withObject:@{@"param": @"leejunhui"} afterDelay:1.f];
}

- (void)taskWithParam:(NSDictionary *)param
{
    NSLog(@"%s", __func__);
    NSLog(@"%@", param);
}

// 输出
2020-03-12 11:25:01.475634+0800 PerformSelectorIndepth[61898:838345] -[ViewController taskWithParam:]
2020-03-12 11:25:01.475837+0800 PerformSelectorIndepth[61898:838345] {
    param = leejunhui;
}
```

>  This method sets up a timer to perform the aSelector message on the current thread’s run loop. The timer is configured to run in the default mode (NSDefaultRunLoopMode). When the timer fires, the thread attempts to dequeue the message from the run loop and perform the selector. It succeeds if the run loop is running and in the default mode; otherwise, the timer waits until the run loop is in the default mode.
> 
> 这个方法会在当前线程所对应的 runloop 中设置一个定时器来执行传入的 SEL。定时器需要在 NSDefaultRunLoopMode 模式下才会被触发。当定时器启动后，线程会尝试从 runloop 中取出 SEL 然后执行。
     如果 runloop 已经启动并且处于 NSDefaultRunLoopMode 的话，SEL 执行成功。否则，直到 runloop 处于 NSDefaultRunLoopMode 前，timer 都会一直等待
     
通过断点调试如下图所示，runloop 底层最终是通过 `__CFRUNLOOP_IS_CALLING_OUT_TO_A_TIMER_CALLBACK_FUNCTION__ ()`来触发任务的执行。

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200312163803.jpg)

因为 `NSRunLoop` 并没有开源，所以我们只能通过 `GNUStep` 来窥探底层实现细节，如下所示：

```objectivec
- (void) performSelector: (SEL)aSelector
	      withObject: (id)argument
	      afterDelay: (NSTimeInterval)seconds
{
  NSRunLoop		*loop = [NSRunLoop currentRunLoop];
  GSTimedPerformer	*item;

  item = [[GSTimedPerformer alloc] initWithSelector: aSelector
					     target: self
					   argument: argument
					      delay: seconds];
  [[loop _timedPerformers] addObject: item];
  RELEASE(item);
  [loop addTimer: item->timer forMode: NSDefaultRunLoopMode];
}


/*
 * The GSTimedPerformer class is used to hold information about
 * messages which are due to be sent to objects at a particular time.
 */
@interface GSTimedPerformer: NSObject
{
@public
  SEL		selector;
  id		target;
  id		argument;
  NSTimer	*timer;
}

- (void) fire;
- (id) initWithSelector: (SEL)aSelector
		 target: (id)target
	       argument: (id)argument
		  delay: (NSTimeInterval)delay;
- (void) invalidate;
@end
```

我们可以看到，在 `performSelector:WithObject:afterDelay:` 底层

* 获取当前线程的 `NSRunLoop` 对象。
* 通过传入的 `SEL` 、`argument` 和 `delay` 初始化一个 `GSTimedPerformer` 实例对象，`GSTimedPerformer` 类型里面封装了 `NSTimer` 对象。
* 然后把 `GSTimedPerformer` 实例加入到 `RunLoop` 对象的 `_timedPerformers` 成员变量中
* 释放掉 `GSTimedPerformer` 对象
* 以 `default mode` 将 `timer` 对象加入到 `runloop` 中


-------


*  `performSelector:WithObject:afterDelay:inModes`

`performSelector:WithObject:afterDelay:inModes` 方法相比上个方法多了一个 `modes` 参数，根据官方文档的定义，只有当 `runloop` 处于 `modes` 中的任意一个 `mode` 时，才会执行任务，如果 `modes` 为空，那么将不会执行任务。


```objectivec
- (void)jh_performSelectorwithObjectafterDelayInModes
{
    [self performSelector:@selector(taskWithParam:) withObject:@{@"param": @"leejunhui"} afterDelay:1.f inModes:@[NSRunLoopCommonModes]];
}

- (void)taskWithParam:(NSDictionary *)param
{
    NSLog(@"%s", __func__);
    NSLog(@"%@", param);
}

// 打印如下
2020-03-12 11:38:58.479152+0800 PerformSelectorIndepth[62006:851520] -[ViewController taskWithParam:]
2020-03-12 11:38:58.479350+0800 PerformSelectorIndepth[62006:851520] {
    param = leejunhui;
}
```

> 这里我们如果把 `modes` 参数改为 `UITrackingRunLoopMode`，那么就只有在 `scrollView` 发生滚动的时候才会触发 timer

我们再看一下 `GNUStep` 对应的实现：

```objectivec
- (void) performSelector: (SEL)aSelector
	      withObject: (id)argument
	      afterDelay: (NSTimeInterval)seconds
		 inModes: (NSArray*)modes
{
  unsigned	count = [modes count];

  if (count > 0)
    {
      NSRunLoop		*loop = [NSRunLoop currentRunLoop];
      NSString		*marray[count];
      GSTimedPerformer	*item;
      unsigned		i;

      item = [[GSTimedPerformer alloc] initWithSelector: aSelector
						 target: self
					       argument: argument
						  delay: seconds];
      [[loop _timedPerformers] addObject: item];
      RELEASE(item);
      if ([modes isProxy])
	{
	  for (i = 0; i < count; i++)
	    {
	      marray[i] = [modes objectAtIndex: i];
	    }
	}
      else
	{
          [modes getObjects: marray];
	}
      for (i = 0; i < count; i++)
	{
	  [loop addTimer: item->timer forMode: marray[i]];
	}
    }
}

@end
```

可以看到，相比于上一个方法的底层实现不同的是，这里会循环添加不同 `mode` 的 `timer` 对象到 `runloop` 中。


-------


* `cancelPreviousPerformRequestsWithTarget:` 和 `cancelPreviousPerformRequestsWithTarget:selector:object:`


`cancelPreviousPerformRequestsWithTarget:` 方法和 `cancelPreviousPerformRequestsWithTarget:selector:object:` 方法是两个类方法，它们的作用是取消执行之前通过 `performSelector:WithObject:afterDelay:` 方法注册的任务。使用起来如下所示:

```objectivec
- (void)jh_performSelectorwithObjectafterDelayInModes
{
    // 只有当 scrollView 发生滚动时，才会触发timer
//    [self performSelector:@selector(taskWithParam:) withObject:@{@"param": @"leejunhui"} afterDelay:1.f inModes:@[UITrackingRunLoopMode]];
    
    [self performSelector:@selector(taskWithParam:) withObject:@{@"param": @"leejunhui"} afterDelay:5.f inModes:@[NSRunLoopCommonModes]];
}

- (IBAction)cancelTask {
    NSLog(@"%s", __func__);
    [ViewController cancelPreviousPerformRequestsWithTarget:self selector:@selector(taskWithParam:) object:@{@"param": @"leejunhui"}];
    
    // [ViewController cancelPreviousPerformRequestsWithTarget:self];
}

// 输出
2020-03-12 11:52:33.549213+0800 PerformSelectorIndepth[62172:865289] -[ViewController cancelTask]
```

这里有一个区别，就是 `cancelPreviousPerformRequestsWithTarget:` 类方法会取消掉 `target` 上所有的通过 `performSelector:WithObject:afterDelay:` 实例方法注册的定时任务，而 `cancelPreviousPerformRequestsWithTarget:selector:object:` 只会通过传入的 `SEL` 取消匹配到的定时任务

在 `GNUStep` 中 `cancelPreviousPerformRequestsWithTarget:` 方法底层实现如下：

```objectivec
/*
 * Cancels any perform operations set up for the specified target
 * in the current run loop.
 */
+ (void) cancelPreviousPerformRequestsWithTarget: (id)target
{
    NSMutableArray	*perf = [[NSRunLoop currentRunLoop] _timedPerformers];
    unsigned		count = [perf count];
    
    if (count > 0)
    {
        GSTimedPerformer	*array[count];
        
        IF_NO_GC(RETAIN(target));
        [perf getObjects: array];
        while (count-- > 0)
        {
            GSTimedPerformer	*p = array[count];
            
            if (p->target == target)
            {
                [p invalidate];
                [perf removeObjectAtIndex: count];
            }
        }
        RELEASE(target);
    }
}

// GSTimedPerformer 实例方法
- (void) invalidate
{
    if (timer != nil)
    {
        [timer invalidate];
        DESTROY(timer);
    }
}

```

这里的逻辑其实很清晰：

* 取出当前 runloop 对象的成员变量 `_timedPerformers`
* 判断定时任务数组是否为空，不为空才会继续往下走
* 初始化一个局部的空的任务数组，然后通过 `getObjects` 从成员变量中取出任务
* 通过 while 循环遍历所有的任务，如果匹配到了对应的 `target`，则调用任务的 invalidate 方法，在这个方法内部会把定时器停掉然后销毁。接着还需要把成员变量 `_timedPerformers` 中对应的任务移除掉

另一个取消任务的方法底层实现如下:

```objectivec
/*
 * Cancels any perform operations set up for the specified target
 * in the current loop, but only if the value of aSelector and argument
 * with which the performs were set up match those supplied.<br />
 * Matching of the argument may be either by pointer equality or by
 * use of the [NSObject-isEqual:] method.
 */
+ (void) cancelPreviousPerformRequestsWithTarget: (id)target
                                        selector: (SEL)aSelector
                                          object: (id)arg
{
    NSMutableArray	*perf = [[NSRunLoop currentRunLoop] _timedPerformers];
    unsigned		count = [perf count];
    
    if (count > 0)
    {
        GSTimedPerformer	*array[count];
        
        IF_NO_GC(RETAIN(target));
        IF_NO_GC(RETAIN(arg));
        [perf getObjects: array];
        while (count-- > 0)
        {
            GSTimedPerformer	*p = array[count];
            
            if (p->target == target && sel_isEqual(p->selector, aSelector)
                && (p->argument == arg || [p->argument isEqual: arg]))
            {
                [p invalidate];
                [perf removeObjectAtIndex: count];
            }
        }
        RELEASE(arg);
        RELEASE(target);
    }
}

```

这里的实现不一样的地方就是除了判断 `target` 是否匹配外，还会判断 `SEL` 是否匹配，以及参数是否匹配。


-------


* `performSelector:WithObject:afterDelay:`
    * 在该方法所在线程的 runloop 处于 default mode 时，根据给定的时间触发给定的任务。底层原理是把一个 timer 对象以 **default mode** 加入到 runloop 对象中，等待唤醒。
* `performSelector:WithObject:afterDelay:inModes:`
    * 在该方法所在线程的 runloop 处于给定的任一 mode 时，根据给定的时间触发给定的任务。底层原理是循环把一个 timer 对象以给定的 mode 加入到 runloop 对象中，等待唤醒。
* `cancelPreviousPerformRequestsWithTarget:`
    * 取消 `target` 对象通过 `performSelector:WithObject:afterDelay:`  方法或 `performSelector:WithObject:afterDelay:inModes:` 方法注册的**所有定时任务**
* `cancelPreviousPerformRequestsWithTarget:selector:object:`
    * 取消 `target` 对象通过 `performSelector:WithObject:afterDelay:`  方法或 `performSelector:WithObject:afterDelay:inModes:` 方法注册的**指定的定时任务**

这四个方法是作为 `NSObject` 的 `NSDelayedPerforming` 分类存在于 `NSRunLoop` 源代码中，所以我们在使用的时候要注意一个细节，那就是执行这些方法的线程是否是主线程，如果是主线程，那么执行起来是没有问题的，但是，**如果是在子线程中执行这些方法，则需要开启子线程对应的 runloop 才能保证执行成功**。

```objectivec
- (void)jh_performSelectorwithObjectafterDelay
{
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        [self performSelector:@selector(taskWithParam:) withObject:@{@"param": @"leejunhui"} afterDelay:1.f];
    });
    
//    [self performSelector:@selector(taskWithParam:) withObject:@{@"param": @"leejunhui"} afterDelay:1.f];
}
    
- (void)taskWithParam:(NSDictionary *)param
{
    NSLog(@"%s", __func__);
    NSLog(@"%@", param);
}    


// 没有输出
```

如上所示的代码，通过 `GCD` 的异步执行函数在全局并发队列上执行任务，并没有任何打印输出，我们加入 runloop 的启动代码后结果将完全不一样：

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200312163806.jpg)

对于 `performSelector:WithObject:afterDelay:inModes` 方法，如果遇到这样的情况，也是一样的解决方案。

## NSRunLoop 的分类 NSOrderedPerform

* performSelector:target:argument:order:modes:

`performSelector:target:argument:order:modes:` 方法的调用者是 `NSRunLoop` 实例，然后需要传入要执行的 `SEL`，以及 `SEL` 对应的 `target`，和 `SEL` 要接收的参数 `argument`，最后是此次任务的优先级 `order`，以及一个 **运行模式集合** `modes`，目的是当 `runloop` 的 `currentMode` 处于这个运行模式集合中的其中任意一个 mode 时，就会按照优先级 `order` 来触发 `SEL` 的执行。具体使用如下：

```objectivec
- (void)jh_performSelectorTargetArgumentOrderModes
{
    NSRunLoop *runloop = [NSRunLoop currentRunLoop];
    [runloop performSelector:@selector(runloopTask5) target:self argument:nil order:5 modes:@[NSRunLoopCommonModes]];
    [runloop performSelector:@selector(runloopTask1) target:self argument:nil order:1 modes:@[NSRunLoopCommonModes]];
    [runloop performSelector:@selector(runloopTask3) target:self argument:nil order:3 modes:@[NSRunLoopCommonModes]];
    [runloop performSelector:@selector(runloopTask2) target:self argument:nil order:2 modes:@[NSRunLoopCommonModes]];
    [runloop performSelector:@selector(runloopTask4) target:self argument:nil order:4 modes:@[NSRunLoopCommonModes]];
}

- (void)runloopTask1
{
    NSLog(@"runloop 任务1");
}

- (void)runloopTask2
{
    NSLog(@"runloop 任务2");
}

- (void)runloopTask3
{
    NSLog(@"runloop 任务3");
}

- (void)runloopTask4
{
    NSLog(@"runloop 任务4");
}

- (void)runloopTask5
{
    NSLog(@"runloop 任务5");
}

// 输出
2020-03-12 14:23:27.088636+0800 PerformSelectorIndepth[62976:972980] runloop 任务1
2020-03-12 14:23:27.088760+0800 PerformSelectorIndepth[62976:972980] runloop 任务2
2020-03-12 14:23:27.088868+0800 PerformSelectorIndepth[62976:972980] runloop 任务3
2020-03-12 14:23:27.088964+0800 PerformSelectorIndepth[62976:972980] runloop 任务4
2020-03-12 14:23:27.089048+0800 PerformSelectorIndepth[62976:972980] runloop 任务5
```

可以看到输出结果就是按照我们传入的 `order` 参数作为任务执行的顺序。

`GUNStep` 中这个底层的底层实现如下:

```objectivec
- (void) performSelector: (SEL)aSelector
                  target: (id)target
                argument: (id)argument
                   order: (NSUInteger)order
                   modes: (NSArray*)modes
{
    unsigned		count = [modes count];
    
    if (count > 0)
    {
        NSString			*array[count];
        GSRunLoopPerformer	*item;
        
        item = [[GSRunLoopPerformer alloc] initWithSelector: aSelector
                                                     target: target
                                                   argument: argument
                                                      order: order];
        
        if ([modes isProxy])
        {
            unsigned	i;
            
            for (i = 0; i < count; i++)
            {
                array[i] = [modes objectAtIndex: i];
            }
        }
        else
        {
            [modes getObjects: array];
        }
        while (count-- > 0)
        {
            NSString	*mode = array[count];
            unsigned	end;
            unsigned	i;
            GSRunLoopCtxt	*context;
            GSIArray	performers;
            
            context = NSMapGet(_contextMap, mode);
            if (context == nil)
            {
                context = [[GSRunLoopCtxt alloc] initWithMode: mode
                                                        extra: _extra];
                NSMapInsert(_contextMap, context->mode, context);
                RELEASE(context);
            }
            performers = context->performers;
            
            end = GSIArrayCount(performers);
            for (i = 0; i < end; i++)
            {
                GSRunLoopPerformer	*p;
                
                p = GSIArrayItemAtIndex(performers, i).obj;
                if (p->order > order)
                {
                    GSIArrayInsertItem(performers, (GSIArrayItem)((id)item), i);
                    break;
                }
            }
            if (i == end)
            {
                GSIArrayInsertItem(performers, (GSIArrayItem)((id)item), i);
            }
            i = GSIArrayCount(performers);
            if (i % 1000 == 0 && i > context->maxPerformers)
            {
                context->maxPerformers = i;
                NSLog(@"WARNING ... there are %u performers scheduled"
                      @" in mode %@ of %@\n(Latest: [%@ %@])",
                      i, mode, self, NSStringFromClass([target class]),
                      NSStringFromSelector(aSelector));
            }
        }
        RELEASE(item);
    }
}

@interface GSRunLoopPerformer: NSObject
{
@public
    SEL		selector;
    id		target;
    id		argument;
    unsigned	order;
}
```

我们已经知道了 `performSelector:WithObject:afterDelay:` 方法底层实现使用一个包裹 timer 对象的数据结构的方式，而这里是使用了一个包裹了 `selector`、`target`、`argument` 以及优先级 `order` 的数据结构的方式来实现。同时在 context 上下文的成员变量 `performers` 中存储了要执行的任务队列，所以这里实际上就是一个简单的插入排序的过程。


-------


*  `cancelPerformSelector:target:argument:` 和 `cancelPerformSelectorsWithTarget:`

`cancelPerformSelector:target:argument:` 和 `cancelPerformSelectorsWithTarget:` 使用起来比较简单，一个需要传入 `selector`、`target` 和 `argument`，另一个只需要传入 `target`。它们的作用分别是根据给定的三个参数或 `target` 去 runloop 底层的 `performers` 任务队列中查找任务，找到了就从队列中移除掉。

而底层具体实现具体如下:

```objectivec
/**
 * Cancels any perform operations set up for the specified target
 * in the receiver.
 */
- (void) cancelPerformSelectorsWithTarget: (id) target
{
    NSMapEnumerator	enumerator;
    GSRunLoopCtxt		*context;
    void			*mode;
    
    enumerator = NSEnumerateMapTable(_contextMap);
    
    while (NSNextMapEnumeratorPair(&enumerator, &mode, (void**)&context))
    {
        if (context != nil)
        {
            GSIArray	performers = context->performers;
            unsigned	count = GSIArrayCount(performers);
            
            while (count--)
            {
                GSRunLoopPerformer	*p;
                
                p = GSIArrayItemAtIndex(performers, count).obj;
                if (p->target == target)
                {
                    GSIArrayRemoveItemAtIndex(performers, count);
                }
            }
        }
    }
    NSEndMapTableEnumeration(&enumerator);
}

/**
 * Cancels any perform operations set up for the specified target
 * in the receiver, but only if the value of aSelector and argument
 * with which the performs were set up match those supplied.<br />
 * Matching of the argument may be either by pointer equality or by
 * use of the [NSObject-isEqual:] method.
 */
- (void) cancelPerformSelector: (SEL)aSelector
                        target: (id) target
                      argument: (id) argument
{
    NSMapEnumerator	enumerator;
    GSRunLoopCtxt		*context;
    void			*mode;
    
    enumerator = NSEnumerateMapTable(_contextMap);
    
    while (NSNextMapEnumeratorPair(&enumerator, &mode, (void**)&context))
    {
        if (context != nil)
        {
            GSIArray	performers = context->performers;
            unsigned	count = GSIArrayCount(performers);
            
            while (count--)
            {
                GSRunLoopPerformer	*p;
                
                p = GSIArrayItemAtIndex(performers, count).obj;
                if (p->target == target && sel_isEqual(p->selector, aSelector)
                    && (p->argument == argument || [p->argument isEqual: argument]))
                {
                    GSIArrayRemoveItemAtIndex(performers, count);
                }
            }
        }
    }
    NSEndMapTableEnumeration(&enumerator);
}
```


-------

* `performSelector:target:argument:order:modes:` 
    * 在该方法所在线程的 runloop 处于给定的任一 mode 时，且处于下一次 runloop 消息循环的开头的时候触发给定的任务。底层原理是循环把一个类似于 timer 的对象加入到 runloop 的上下文的任务队列中，等待唤醒
* `cancelPerformSelector:target:argument:`
    * 取消 target 对象通过 `performSelector:target:argument:order:modes:` 方法方法注册的**指定的任务**
* `cancelPerformSelectorsWithTarget:`
    * 取消 target 对象通过 `performSelector:target:argument:order:modes:` 方法方法注册的**所有任务**

这里同样的也需要注意，**如果是在子线程中执行这些方法，则需要开启子线程对应的 runloop 才能保证执行成功**。

# Thread 相关的 `performSelector`

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/20200312163811.jpg)

如上图所示，在 `NSThread` 中定义了 `NSObject` 的分类 `NSThreadPerformAdditions`，其中定义了 5 个 `performSelector` 的方法。


-------


* `performSelector:onThread:withObject:waitUntilDone:` 
* `performSelector:onThread:withObject:waitUntilDone:modes:`

根据官方文档的解释，第一个方法相当于调用了第二个方法，然后 mode 传入的是 `kCFRunLoopCommonModes`。我们这里只研究第一个方法。

这个方法需要相比于 `performSeletor:withObject:` 多了两个参数，分别是要哪个线程执行任务以及是否阻塞当前线程。但是使用这个方法一定要小心，如下图所示是一个常见的错误用法：


![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/15839893768175.jpg)


这里报的错是 `target thread exited while waiting for the perform`，就是说已经退出的线程无法执行定时任务。
熟悉 `iOS` 多线程的同学都知道 `NSThread` 实例化之后的线程对象在 `start` 之后就会被系统回收，而之后调用的 `performSelector:onThread:withObject:waitUntilDone:` 方法又在一个**已经回收的线程**上执行任务，显然就会崩溃。这里的解决方案就是给这个子线程对应的 runloop 启动起来，让线程具有 『有事来就干活，没事干就睡觉』 的功能，具体代码如下:

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/15839895561944.jpg)

对于 `waitUntilDone` 参数，如果我们设置为 `YES`:

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/15839897039699.jpg)

如果设置为 `NO`:

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/15839898047089.jpg)


所以这里的 `waitUntilDone` 可以简单的理解为控制同步或异步执行。

在探索 `GNUStep` 对应实现之前，我们先熟悉一下 `GSRunLoopThreadInfo`

```objectivec
/* Used to handle events performed in one thread from another.
 */
@interface      GSRunLoopThreadInfo : NSObject
{
  @public
  NSRunLoop             *loop;
  NSLock                *lock;
  NSMutableArray        *performers;
#ifdef _WIN32
  HANDLE	        event;
#else
  int                   inputFd;
  int                   outputFd;
#endif	
}
```

`GSRunLoopThreadInfo` 是每个线程特有的一个属性，存储了线程和 runloop 之间的一些信息，可以通过下面的方式获取:

```objectivec
GSRunLoopThreadInfo *
GSRunLoopInfoForThread(NSThread *aThread)
{
    GSRunLoopThreadInfo   *info;
    
    if (aThread == nil)
    {
        aThread = GSCurrentThread();
    }
    if (aThread->_runLoopInfo == nil)
    {
        [gnustep_global_lock lock];
        if (aThread->_runLoopInfo == nil)
        {
            aThread->_runLoopInfo = [GSRunLoopThreadInfo new];
        }
        [gnustep_global_lock unlock];
    }
    info = aThread->_runLoopInfo;
    return info;
}
```

然后是另一个 `GSPerformHolder`: 

```objectivec
/**
 * This class performs a dual function ...
 * <p>
 *   As a class, it is responsible for handling incoming events from
 *   the main runloop on a special inputFd.  This consumes any bytes
 *   written to wake the main runloop.<br />
 *   During initialisation, the default runloop is set up to watch
 *   for data arriving on inputFd.
 * </p>
 * <p>
 *   As instances, each  instance retains perform receiver and argument
 *   values as long as they are needed, and handles locking to support
 *   methods which want to block until an action has been performed.
 * </p>
 * <p>
 *   The initialize method of this class is called before any new threads
 *   run.
 * </p>
 */
@interface GSPerformHolder : NSObject
{
    id			receiver;
    id			argument;
    SEL			selector;
    NSConditionLock	*lock;		// Not retained.
    NSArray		*modes;
    BOOL                  invalidated;
@public
    NSException           *exception;
}
+ (GSPerformHolder*) newForReceiver: (id)r
                           argument: (id)a
                           selector: (SEL)s
                              modes: (NSArray*)m
                               lock: (NSConditionLock*)l;
- (void) fire;
- (void) invalidate;
- (BOOL) isInvalidated;
- (NSArray*) modes;
@end
```

`GSPerformHolder` 封装了任务的细节(`receiver`, `argument`, `selector`)以及运行模式(`mode`)和一把条件锁( `NSConditionLock` )。

接着我们目光聚焦到源码 `performSelector:onThread:withObject:waitUntilDone:modes:` 具体实现上:

```objectivec
- (void) performSelector: (SEL)aSelector
                onThread: (NSThread*)aThread
              withObject: (id)anObject
           waitUntilDone: (BOOL)aFlag
                   modes: (NSArray*)anArray
{
    GSRunLoopThreadInfo   *info;
    NSThread	        *t;
    
    if ([anArray count] == 0)
    {
        return;
    }
    
    t = GSCurrentThread();
    if (aThread == nil)
    {
        aThread = t;
    }
    info = GSRunLoopInfoForThread(aThread);
    if (t == aThread)
    {
        /* Perform in current thread.
         */
        if (aFlag == YES || info->loop == nil)
        {
            /* Wait until done or no run loop.
             */
            [self performSelector: aSelector withObject: anObject];
        }
        else
        {
            /* Don't wait ... schedule operation in run loop.
             */
            [info->loop performSelector: aSelector
                                 target: self
                               argument: anObject
                                  order: 0
                                  modes: anArray];
        }
    }
    else
    {
        GSPerformHolder   *h;
        NSConditionLock	*l = nil;
        
        if ([aThread isFinished] == YES)
        {
            [NSException raise: NSInternalInconsistencyException
                        format: @"perform [%@-%@] attempted on finished thread (%@)",
             NSStringFromClass([self class]),
             NSStringFromSelector(aSelector),
             aThread];
        }
        if (aFlag == YES)
        {
            l = [[NSConditionLock alloc] init];
        }
        
        h = [GSPerformHolder newForReceiver: self
                                   argument: anObject
                                   selector: aSelector
                                      modes: anArray
                                       lock: l];
        [info addPerformer: h];
        if (l != nil)
        {
            [l lockWhenCondition: 1];
            [l unlock];
            RELEASE(l);
            if ([h isInvalidated] == NO)
            {
                /* If we have an exception passed back from the remote thread,
                 * re-raise it.
                 */
                if (nil != h->exception)
                {
                    NSException       *e = AUTORELEASE(RETAIN(h->exception));
                    
                    RELEASE(h);
                    [e raise];
                }
            }
        }
        RELEASE(h);
    }
}
```

* 声明一个`GSRunLoopThreadInfo` 对象和一条 `NSThread` 线程
* 判断运行模式数组参数是否为空
* 获取当前线程，将结果赋值于第一步声明的局部线程变量
* 判断如果传入的要执行任务的线程 aThread 如果为空，那么就把当前线程赋值于到 aThread 上
* 确保 aThread 不为空之后获取该线程对应的 `GSRunLoopThreadInfo` 对象并赋值于第一步声明的局部 info 变量
* 确保 info 有值后，判断是否是在当前线程上执行任务
* 如果是在当前线程上执行任务，接着判断是否要阻塞当前线程，或当前线程的 runloop 为空。
    * 如果是的话，则直接调用 `performSelector:withObject` 来执行任务
    * 如果不是的话，则通过线程对应的 runloop 对象调用 `performSelector:target:argument:order:modes:` 来执行任务
* 如果不是在当前线程上执行任务，声明一个 `GSPerformHolder` 局部变量，声明一把空的条件锁 `NSConditionLock`
    * 判断要执行任务的线程是否已经被回收，如果已被回收，则抛出异常
    * 如果未被回收
        * 判断是否要阻塞当前线程，如果传入的参数需要阻塞，则初始化条件锁
        * 根据传入的参数及条件锁初始化  `GSPerformHolder` 实例
        * 然后在 info 中加入 `GSPerformHolder` 实例
        * 然后判断条件锁如果不为空，赋予条件锁何时加锁的条件，然后解锁条件锁，然后释放条件锁
        * 判断 `GSPerformHolder` 局部变量是否已经被释放，如果没有被释放，抛出异常


-------


* `performSelectorOnMainThread:withObject:waitUntilDone:`
* `performSelectorOnMainThread:withObject:waitUntilDone:modes:`

顾名思义，这两个方法其实就是在主线程上执行任务，根据传入的参数决定是否阻塞主线程，以及在哪些运行模式下执行任务。使用方法如下:

```objectivec
- (void)jh_performSelectorOnMainThreadwithObjectwaitUntilDone
{
    [self performSelectorOnMainThread:@selector(threadTask:) withObject:@{@"param": @"leejunhui"} waitUntilDone:NO];
//    [self performSelectorOnMainThread:@selector(threadTask:) withObject:@{@"param": @"leejunhui"} waitUntilDone:NO modes:@[NSRunLoopCommonModes]];
}

- (void)threadTask:(NSDictionary *)param
{
    NSLog(@"%s", __func__);
    NSLog(@"%@", [NSThread currentThread]);
}

// 输出
2020-03-12 16:14:31.783962+0800 PerformSelectorIndepth[63614:1057033] -[ViewController threadTask:]
2020-03-12 16:14:31.784126+0800 PerformSelectorIndepth[63614:1057033] <NSThread: 0x600002e76dc0>{number = 1, name = main}
```

因为是在主线程上执行，所以并不需要手动开启 runloop。我们来看下这两个方法在 `GNUStep` 中底层实现:

```objectivec
- (void) performSelectorOnMainThread: (SEL)aSelector
                          withObject: (id)anObject
                       waitUntilDone: (BOOL)aFlag
                               modes: (NSArray*)anArray
{
    /* It's possible that this method could be called before the NSThread
     * class is initialised, so we check and make sure it's initiailised
     * if necessary.
     */
    if (defaultThread == nil)
    {
        [NSThread currentThread];
    }
    [self performSelector: aSelector
                 onThread: defaultThread
               withObject: anObject
            waitUntilDone: aFlag
                    modes: anArray];
}

- (void) performSelectorOnMainThread: (SEL)aSelector
                          withObject: (id)anObject
                       waitUntilDone: (BOOL)aFlag
{
    [self performSelectorOnMainThread: aSelector
                           withObject: anObject
                        waitUntilDone: aFlag
                                modes: commonModes()];
}
```

不难看出，这里其实就是调用的 `performSelector:onThread:withObject:waitUntilDone:modes` 方法，但是有一个细节需要注意，就是有可能在 `NSThread` 类被初始化之前，就调用了 `performSelectorOnMainThread` 方法，所以需要手动调用一下 `[NSThread currentThread]`。


-------

* `performSelectorInBackground:withObject:`

最后要探索的是 `performSelectorInBackground:withObject:` 方法，这个方法用法如下：

```objectivec
- (void)jh_performSelectorOnBackground
{
    [self performSelectorInBackground:@selector(threadTask:) withObject:@{@"param": @"leejunhui"}];
}

- (void)threadTask:(NSDictionary *)param
{
    NSLog(@"%s", __func__);
    NSLog(@"%@", [NSThread currentThread]);
}

// 输出
2020-03-12 16:19:36.751675+0800 PerformSelectorIndepth[63660:1061569] -[ViewController threadTask:]
2020-03-12 16:19:36.751990+0800 PerformSelectorIndepth[63660:1061569] <NSThread: 0x6000027a0ac0>{number = 6, name = (null)}
```

根据输出我们可知，这里显然是开了一条子线程来执行任务，我们看一下 `GNUStep` 的底层实现:

```objectivec
- (void) performSelectorInBackground: (SEL)aSelector
                          withObject: (id)anObject
{
    [NSThread detachNewThreadSelector: aSelector
                             toTarget: self
                           withObject: anObject];
}
```

可以看到在底层其实是调用的 `NSThread` 的类方法来执行传入的任务。关于 `NSThread` 细节我们后面会进行探索。

-------


* `performSelector:onThread:withObject:waitUntilDone:` 和 `performSelector:onThread:withObject:waitUntilDone:modes:`
    * 在该方法所在线程的 runloop 处于给定的任一 mode 时，判断是否阻塞当前线程，并且处于下一次 runloop 消息循环的开头的时候触发给定的任务。
* `performSelectorOnMainThread:withObject:waitUntilDone:` 和 `performSelectorOnMainThread:withObject:waitUntilDone:modes:`
    * 当主线程的 runloop 处于给定的任一 mode 时，判断是否阻塞主线程，并且处于下一次 runloop 消息循环的开头的时候触发给定的任务。
* `performSelectorInBackground:withObject: `
    * 在子线程上执行给定的任务。底层是通过 `NSThread` 的 `detachNewThread` 实现。



-------

# 总结

![](https://raw.githubusercontent.com/LeeJunhui/blog_images/master/performSelector.png)
