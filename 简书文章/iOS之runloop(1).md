`runloop` 这和`runtime`一样重要，那么我们现在探究一下`runloop`究竟是怎么运行的。

`runloop`其实就是APP运行的根本，我们写的代码执行完了，为什么app没有退出？就是因为`runloop`的存在。那么`runloop`究竟是什么呢？
##### RunLoop 的作用：
保证程序不退出保活线程，它就是一个死循环："接受事件 -> ...等待 -> 处理"
负责监听管理事件，触摸事件（用户交互事件）、时钟事件（Timer）、网络事件（网络请求回调接收等）以及系统事件内核事件；
如果没有事件则进入睡眠状态
也就是有了`runloop`，才保证我们的APP一直运行下去，永不退出。
伪代码如下：
```
 function loop() {
    initialize();
    do {
        var message = get_next_message();
        process_message(message);
    } while (message != quit);
}
```
当循环执行完成结束的时候，就是我们APP退出的时候。

看一下系统中的`runloop`到底是怎么运行的呢？![runloop](https://upload-images.jianshu.io/upload_images/783986-db957039adb511c8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```
/// 全局的Dictionary，key 是 pthread_t， value 是 CFRunLoopRef
static CFMutableDictionaryRef loopsDic;
/// 访问 loopsDic 时的锁
static CFSpinLock_t loopsLock;
 
/// 获取一个 pthread 对应的 RunLoop。
CFRunLoopRef _CFRunLoopGet(pthread_t thread) {
    OSSpinLockLock(&loopsLock);
    
    if (!loopsDic) {
        // 第一次进入时，初始化全局Dic，并先为主线程创建一个 RunLoop。
        loopsDic = CFDictionaryCreateMutable();
        CFRunLoopRef mainLoop = _CFRunLoopCreate();
        CFDictionarySetValue(loopsDic, pthread_main_thread_np(), mainLoop);
    }
    
    /// 直接从 Dictionary 里获取。
    CFRunLoopRef loop = CFDictionaryGetValue(loopsDic, thread));
    
    if (!loop) {
        /// 取不到时，创建一个
        loop = _CFRunLoopCreate();
        CFDictionarySetValue(loopsDic, thread, loop);
        /// 注册一个回调，当线程销毁时，顺便也销毁其对应的 RunLoop。
        _CFSetTSD(..., thread, loop, __CFFinalizeRunLoop);
    }
    
    OSSpinLockUnLock(&loopsLock);
    return loop;
}
 
CFRunLoopRef CFRunLoopGetMain() {
    return _CFRunLoopGet(pthread_main_thread_np());
}
 
CFRunLoopRef CFRunLoopGetCurrent() {
    return _CFRunLoopGet(pthread_self());
}
```
从上面的代码可以看出，线程和 RunLoop 之间是一一对应的，其关系是保存在一个全局的 Dictionary 里。线程刚创建时并没有 RunLoop，如果你不主动获取，那它一直都不会有。RunLoop 的创建是发生在第一次获取时，RunLoop 的销毁是发生在线程结束时。你只能在一个线程的内部获取其 RunLoop（主线程除外）

看一下内部处理：
```
{
    /// 1. 通知Observers，即将进入RunLoop
    /// 此处有Observer会创建AutoreleasePool: _objc_autoreleasePoolPush();
    __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopEntry);
    do {
 
        /// 2. 通知 Observers: 即将触发 Timer 回调。
        __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopBeforeTimers);
        /// 3. 通知 Observers: 即将触发 Source (非基于port的,Source0) 回调。
        __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopBeforeSources);
        __CFRUNLOOP_IS_CALLING_OUT_TO_A_BLOCK__(block);
 
        /// 4. 触发 Source0 (非基于port的) 回调。
        __CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__(source0);
        __CFRUNLOOP_IS_CALLING_OUT_TO_A_BLOCK__(block);
 
        /// 6. 通知Observers，即将进入休眠
        /// 此处有Observer释放并新建AutoreleasePool: _objc_autoreleasePoolPop(); _objc_autoreleasePoolPush();
        __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopBeforeWaiting);
 
        /// 7. sleep to wait msg.
        mach_msg() -> mach_msg_trap();
        
 
        /// 8. 通知Observers，线程被唤醒
        __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopAfterWaiting);
 
        /// 9. 如果是被Timer唤醒的，回调Timer
        __CFRUNLOOP_IS_CALLING_OUT_TO_A_TIMER_CALLBACK_FUNCTION__(timer);
 
        /// 9. 如果是被dispatch唤醒的，执行所有调用 dispatch_async 等方法放入main queue 的 block
        __CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__(dispatched_block);
 
        /// 9. 如果如果Runloop是被 Source1 (基于port的) 的事件唤醒了，处理这个事件
        __CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE1_PERFORM_FUNCTION__(source1);
 
 
    } while (...);
 
    /// 10. 通知Observers，即将退出RunLoop
    /// 此处有Observer释放AutoreleasePool: _objc_autoreleasePoolPop();
    __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopExit);
}
```
YY大牛的图：
![RunLoop_0](https://upload-images.jianshu.io/upload_images/783986-faee8ae805e2297c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
每个mode都独立分开的，mode是容器，区分不同环境的不同任务。
![RunLoop_1](https://upload-images.jianshu.io/upload_images/783986-300fa7e60706a0e0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



### AutoreleasePool

App启动后，苹果在主线程 RunLoop 里注册了两个 Observer，其回调都是 _wrapRunLoopWithAutoreleasePoolHandler()。

第一个 Observer 监视的事件是 Entry(即将进入Loop)，其回调内会调用 _objc_autoreleasePoolPush() 创建自动释放池。其 order 是-2147483647，优先级最高，保证创建释放池发生在其他所有回调之前。

第二个 Observer 监视了两个事件： BeforeWaiting(准备进入休眠) 时调用_objc_autoreleasePoolPop() 和 _objc_autoreleasePoolPush() 释放旧的池并创建新池；Exit(即将退出Loop) 时调用 _objc_autoreleasePoolPop() 来释放自动释放池。这个 Observer 的 order 是 2147483647，优先级最低，保证其释放池子发生在其他所有回调之后。

在主线程执行的代码，通常是写在诸如事件回调、Timer回调内的。这些回调会被 RunLoop 创建好的 AutoreleasePool 环绕着，所以不会出现内存泄漏，开发者也不必显示创建 Pool 了。

### 事件响应

苹果注册了一个 Source1 (基于 mach port 的) 用来接收系统事件，其回调函数为 __IOHIDEventSystemClientQueueCallback()。

当一个硬件事件(触摸/锁屏/摇晃等)发生后，首先由 IOKit.framework 生成一个 IOHIDEvent 事件并由 SpringBoard 接收。这个过程的详细情况可以参考[这里](http://iphonedevwiki.net/index.php/IOHIDFamily)。SpringBoard 只接收按键(锁屏/静音等)，触摸，加速，接近传感器等几种 Event，随后用 mach port 转发给需要的App进程。随后苹果注册的那个 Source1 就会触发回调，并调用 _UIApplicationHandleEventQueue() 进行应用内部的分发。

_UIApplicationHandleEventQueue() 会把 IOHIDEvent 处理并包装成 UIEvent 进行处理或分发，其中包括识别 UIGesture/处理屏幕旋转/发送给 UIWindow 等。通常事件比如 UIButton 点击、touchesBegin/Move/End/Cancel 事件都是在这个回调中完成的。

### 手势识别

当上面的 _UIApplicationHandleEventQueue() 识别了一个手势时，其首先会调用 Cancel 将当前的 touchesBegin/Move/End 系列回调打断。随后系统将对应的 UIGestureRecognizer 标记为待处理。

苹果注册了一个 Observer 监测 BeforeWaiting (Loop即将进入休眠) 事件，这个Observer的回调函数是 _UIGestureRecognizerUpdateObserver()，其内部会获取所有刚被标记为待处理的 GestureRecognizer，并执行GestureRecognizer的回调。

当有 UIGestureRecognizer 的变化(创建/销毁/状态改变)时，这个回调都会进行相应处理。


看一下__CFRunLoop
```
//回调函数
struct _block_item {
    struct _block_item *_next;
    CFTypeRef _mode;	// CFString or CFSet
    void (^_block)(void);
};
// reset for runs of the run loop 重新设置正在运行的loop
typedef struct _per_run_data {
    uint32_t a;
    uint32_t b;
    uint32_t stopped;
    uint32_t ignoreWakeUps;
} _per_run_data;

struct __CFRunLoop {
    CFRuntimeBase _base;
//互斥锁
    pthread_mutex_t _lock;			/* locked for accessing mode list */
    __CFPort _wakeUpPort;			// used for CFRunLoopWakeUp 
//是否正在使用
    Boolean _unused;
    volatile _per_run_data *_perRunData;              // reset for runs of the run loop
//当前线程
    pthread_t _pthread;
    uint32_t _winthread;
//modes
    CFMutableSetRef _commonModes;
    CFMutableSetRef _commonModeItems;
//当前mode
    CFRunLoopModeRef _currentMode;
    CFMutableSetRef _modes;
//
    struct  *_blocks_head;_block_item
//回调函数
    struct _block_item *_blocks_tail;
    CFTypeRef _counterpart;
};

```
使用`CFRunLoopObserverCreateWithHandler`创建obs
源码：
```
//CFAllocatorRef 指定系统分配函数内存  默认kCFAllocatorDefault
//repeats 是否重复
//order kaxprioritykey的优先级
//void (^block) (CFRunLoopObserverRef observer, CFRunLoopActivity activity) 回调函数
CFRunLoopObserverRef CFRunLoopObserverCreateWithHandler(CFAllocatorRef allocator, CFOptionFlags activities, Boolean repeats, CFIndex order,
                                                      void (^block) (CFRunLoopObserverRef observer, CFRunLoopActivity activity)) {
    CFRunLoopObserverContext blockContext;//block 结构体
    blockContext.version = 0;//版本
    blockContext.info = (void *)block;//回调block
    blockContext.retain = (const void *(*)(const void *info))_Block_copy;//手动copy
    blockContext.release = (void (*)(const void *info))_Block_release;//手动释放
    blockContext.copyDescription = NULL;//NULL
    return CFRunLoopObserverCreate(allocator, activities, repeats, order, _runLoopObserverWithBlockContext, &blockContext);//返回  CFRunLoopObserver
}
```
这里就是系统创建了一个context并且赋值了，然后返回了`CFRunLoopObserverCreate(allocator, activities, repeats, order, _runLoopObserverWithBlockContext, &blockContext);//返回  CFRunLoopObserver`。那么再看下`CFRunLoopObserverCreate`:
```
//创建 CFRunLoopObserverRef
//allocator 内存创建 一般使用 kCFAllocatorDefault
//activities Run Loop Observer Activities 一般监测所有 kCFRunLoopAllActivities
//repeats true 重复监测，false监测一次
//order 优先级
//callout 回调函数
//context 上下文
CFRunLoopObserverRef CFRunLoopObserverCreate(CFAllocatorRef allocator, CFOptionFlags activities, Boolean repeats, CFIndex order, CFRunLoopObserverCallBack callout, CFRunLoopObserverContext *context) {
    CHECK_FOR_FORK();
    //obs
    CFRunLoopObserverRef memory;
    UInt32 size;
    //内存大小
    size = sizeof(struct __CFRunLoopObserver) - sizeof(CFRuntimeBase);
    //创建obs
    memory = (CFRunLoopObserverRef)_CFRuntimeCreateInstance(allocator, __kCFRunLoopObserverTypeID, size, NULL);
    //创建失败 返回NULL
    if (NULL == memory) {
	return NULL;
    }
    __CFSetValid(memory);
    __CFRunLoopObserverUnsetFiring(memory);
    //设置 obs 是否重复
    if (repeats) {
	__CFRunLoopObserverSetRepeats(memory);
    } else {
	__CFRunLoopObserverUnsetRepeats(memory);
    }
    //初始化 obs->lock
    __CFRunLoopLockInit(&memory->_lock);
    //初始化 rl = NULL
    memory->_runLoop = NULL;
    //初始化 count = 0
    memory->_rlCount = 0;
    //obs的监测类型
    memory->_activities = activities;
    memory->_order = order;
    //callBack
    memory->_callout = callout;
    if (context) {
        //上下文的数据复制给obs
        if (context->retain) {
            memory->_context.info = (void *)context->retain(context->info);
        } else {
            memory->_context.info = context->info;
        }
        memory->_context.retain = context->retain;
        memory->_context.release = context->release;
        memory->_context.copyDescription = context->copyDescription;
    } else {
        ///没有上下文 直接 数据清空操作
        memory->_context.info = 0;
        memory->_context.retain = 0;
        memory->_context.release = 0;
        memory->_context.copyDescription = 0;
    }
    //返回obs
    return memory;
}
```
这段代码有创建互斥锁，我们在看下锁的参数：
```
//pthread_mutex_t 初始化
CF_INLINE void __CFRunLoopLockInit(pthread_mutex_t *lock) {
    pthread_mutexattr_t mattr;
    pthread_mutexattr_init(&mattr);
    //锁的类型是PTHREAD_MUTEX_RECURSIVE
    pthread_mutexattr_settype(&mattr, PTHREAD_MUTEX_RECURSIVE);
    int32_t mret = pthread_mutex_init(lock, &mattr);
    pthread_mutexattr_destroy(&mattr);
    if (0 != mret) {
    }
}
```
锁的类型是`PTHREAD_MUTEX_RECURSIVE`,那么这是什么意思呢？
对于互斥锁的参数，请看下边解释：
如果[互斥锁](https://baike.baidu.com/item/%E4%BA%92%E6%96%A5%E9%94%81)类型为 PTHREAD_MUTEX_NORMAL，则不提供死锁检测。尝试重新锁定互斥锁会导致死锁。如果某个线程尝试解除锁定的互斥锁不是由该线程锁定或未锁定，则将产生不确定的行为。

如果互斥锁类型为 `PTHREAD_MUTEX_ERRORCHECK`，则会提供错误检查。如果某个线程尝试重新锁定的互斥锁已经由该线程锁定，则将返回错误。如果某个线程尝试解除锁定的互斥锁不是由该线程锁定或者未锁定，则将返回错误。

如果互斥锁类型为 `PTHREAD_MUTEX_RECURSIVE`，则该互斥锁会保留锁定计数这一概念。线程首次成功获取互斥锁时，锁定计数会设置为 1。线程每重新锁定该[互斥锁](https://baike.baidu.com/item/%E4%BA%92%E6%96%A5%E9%94%81)一次，锁定计数就增加 1。线程每解除锁定该互斥锁一次，锁定计数就减小 1。 锁定计数达到 0 时，该互斥锁即可供其他线程获取。如果某个线程尝试解除锁定的互斥锁不是由该线程锁定或者未锁定，则将返回错误。

如果互斥锁类型是 `PTHREAD_MUTEX_DEFAULT`，则尝试以[递归](https://baike.baidu.com/item/%E9%80%92%E5%BD%92)方式锁定该互斥锁将产生不确定的行为。对于不是由调用线程锁定的互斥锁，如果尝试解除对它的锁定，则会产生不确定的行为。如果尝试解除锁定尚未锁定的互斥锁，则会产生不确定的行为。

除了`obs`，还有`runlooptimer`，看下它的数据结构：
```
struct __CFRunLoopTimer {
    CFRuntimeBase _base;
    uint16_t _bits;
    pthread_mutex_t _lock;//锁
    CFRunLoopRef _runLoop;//loop
    CFMutableSetRef _rlModes;//modes
    CFAbsoluteTime _nextFireDate;//下次更新时间
    CFTimeInterval _interval;		/* immutable */
    CFTimeInterval _tolerance;          /* mutable */
    uint64_t _fireTSR;			/* TSR units */
    CFIndex _order;			/* immutable */优先级
    CFRunLoopTimerCallBack _callout;	/* immutable */回调函数
    CFRunLoopTimerContext _context;	/* immutable, except invalidation */上下文
};
```
那么我们看下`CFRunLoopTimerCreate`的创建：
```
/*
 CFAllocatorRef 内存分配
 CFAbsoluteTime 启动时间
 CFTimeInterval 循环间隔
 CFOptionFlags 监测活动类型
 CFIndex 优先级
 CFRunLoopTimerCallBack 回调函数
 CFRunLoopTimerContext 上下文
 */
CFRunLoopTimerRef CFRunLoopTimerCreate(CFAllocatorRef allocator, CFAbsoluteTime fireDate, CFTimeInterval interval, CFOptionFlags flags, CFIndex order, CFRunLoopTimerCallBack callout, CFRunLoopTimerContext *context) {
    CHECK_FOR_FORK();
    //rlt
    CFRunLoopTimerRef memory;
    //占用内存大小
    UInt32 size;
    size = sizeof(struct __CFRunLoopTimer) - sizeof(CFRuntimeBase);
    memory = (CFRunLoopTimerRef)_CFRuntimeCreateInstance(allocator, __kCFRunLoopTimerTypeID, size, NULL);
    if (NULL == memory) {
        //若初始化失败 return null；
	return NULL;
    }
    __CFSetValid(memory);
    //设置 rlt 不启动
    __CFRunLoopTimerUnsetFiring(memory);
    //初始化 互斥锁
    __CFRunLoopLockInit(&memory->_lock);
    //rlf = null
    memory->_runLoop = NULL;
    //保存当前mode
    memory->_rlModes = CFSetCreateMutable(kCFAllocatorSystemDefault, 0, &kCFTypeSetCallBacks);
    memory->_order = order;
    //初始化间隔
    if (interval < 0.0) interval = 0.0;
    memory->_interval = interval;
    memory->_tolerance = 0.0;
    if (TIMER_DATE_LIMIT < fireDate) fireDate = TIMER_DATE_LIMIT;
    memory->_nextFireDate = fireDate;
    memory->_fireTSR = 0ULL;
    uint64_t now2 = mach_absolute_time();
    CFAbsoluteTime now1 = CFAbsoluteTimeGetCurrent();
    //设置 开始的时间
    if (fireDate < now1) {//小于当前时间
        memory->_fireTSR = now2;
    } else if (TIMER_INTERVAL_LIMIT < fireDate - now1) {
        memory->_fireTSR = now2 + __CFTimeIntervalToTSR(TIMER_INTERVAL_LIMIT);
    } else {
        memory->_fireTSR = now2 + __CFTimeIntervalToTSR(fireDate - now1);
    }
    //回调函数
    memory->_callout = callout;
    if (NULL != context) {
	if (context->retain) {
	    memory->_context.info = (void *)context->retain(context->info);
	} else {
	    memory->_context.info = context->info;
	}
        memory->_context.retain = context->retain;
        memory->_context.release = context->release;
        memory->_context.copyDescription = context->copyDescription;
    } else {//没有上下文  则清空数据
        memory->_context.info = 0;
        memory->_context.retain = 0;
        memory->_context.release = 0;
        memory->_context.copyDescription = 0;
    }
    return memory;
}
```

最后一个`source`
```
struct __CFRunLoopSource {
    CFRuntimeBase _base;
    uint32_t _bits;
    pthread_mutex_t _lock;
    CFIndex _order;			/* immutable */
    CFMutableBagRef _runLoops;
    union {
	CFRunLoopSourceContext version0;	/* immutable, except invalidation */
        CFRunLoopSourceContext1 version1;	/* immutable, except invalidation */
    } _context;
};



CFRunLoopSourceRef CFRunLoopSourceCreate(CFAllocatorRef allocator, CFIndex order, CFRunLoopSourceContext *context) {
    CHECK_FOR_FORK();
    //obs
    CFRunLoopSourceRef memory;
    //内存大小
    uint32_t size;
    //上下文 为空
    if (NULL == context) CRASH("*** NULL context value passed to CFRunLoopSourceCreate(). (%d) ***", -1);
    
    size = sizeof(struct __CFRunLoopSource) - sizeof(CFRuntimeBase);
    memory = (CFRunLoopSourceRef)_CFRuntimeCreateInstance(allocator, __kCFRunLoopSourceTypeID, size, NULL);
    if (NULL == memory) {
	return NULL;
    }
    __CFSetValid(memory);
    //默认不 设置发射信号
    __CFRunLoopSourceUnsetSignaled(memory);
    //初始化锁
    __CFRunLoopLockInit(&memory->_lock);
    memory->_bits = 0;
    memory->_order = order;
    memory->_runLoops = NULL;
    size = 0;
    //根据version 计算内存大小
    switch (context->version) {
        case 0:
            size = sizeof(CFRunLoopSourceContext);
        break;
        case 1:
            size = sizeof(CFRunLoopSourceContext1);
        break;
    }
    //拷贝上下文到内存
    objc_memmove_collectable(&memory->_context, context, size);
    if (context->retain) {
        memory->_context.version0.info = (void *)context->retain(context->info);
    }
    return memory;
}
```
`obs`、`timer`、`source`创建都完成，那么我们怎么添加到`runloop`中呢？
在我们添加之前需要检测一下是否已经添加过了。
```
Boolean CFRunLoopContainsObserver(CFRunLoopRef rl, CFRunLoopObserverRef rlo, CFStringRef modeName) {
    CHECK_FOR_FORK();
    CFRunLoopModeRef rlm;
    //默认不包含
    Boolean hasValue = false;
    //互斥锁锁住rl
    __CFRunLoopLock(rl);
    if (modeName == kCFRunLoopCommonModes) {
        //如果modeItems 不为空
        if (NULL != rl->_commonModeItems) {
            //rl->_commonModeItems 是否包含rlo
            hasValue = CFSetContainsValue(rl->_commonModeItems, rlo);
        }
    } else {
        rlm = __CFRunLoopFindMode(rl, modeName, false);
        if (NULL != rlm && NULL != rlm->_observers) {
            //rlm->_observers 是否包含_observers
            hasValue = CFArrayContainsValue(rlm->_observers, CFRangeMake(0, CFArrayGetCount(rlm->_observers)), rlo);
        }
        if (NULL != rlm) {
            //如果查到了 就解锁
        __CFRunLoopModeUnlock(rlm);
        }
    }
    __CFRunLoopUnlock(rl);
    return hasValue;
}
```
当没有添加过的时候，我们就`CFRunLoopAddObserver`添加上。
```
void CFRunLoopAddObserver(CFRunLoopRef rl, CFRunLoopObserverRef rlo, CFStringRef modeName) {
    CHECK_FOR_FORK();
    CFRunLoopModeRef rlm;
    //检查是否 已经销毁
    if (__CFRunLoopIsDeallocating(rl)) return;
    if (!__CFIsValid(rlo) || (NULL != rlo->_runLoop && rlo->_runLoop != rl)) return;
    //加锁
    __CFRunLoopLock(rl);
    if (modeName == kCFRunLoopCommonModes) {
        CFSetRef set = rl->_commonModes ? CFSetCreateCopy(kCFAllocatorSystemDefault, rl->_commonModes) : NULL;
        if (NULL == rl->_commonModeItems) {
            rl->_commonModeItems = CFSetCreateMutable(kCFAllocatorSystemDefault, 0, &kCFTypeSetCallBacks);
        }
        //设置一个默认的 modeItems
        CFSetAddValue(rl->_commonModeItems, rlo);
        if (NULL != set) {
            CFTypeRef context[2] = {rl, rlo};
            /* add new item to all common-modes */
            //添加item 到COmmonModes
            CFSetApplyFunction(set, (__CFRunLoopAddItemToCommonModes), (void *)context);
            CFRelease(set);
        }
    } else {
        //根据name超找runloop
        rlm = __CFRunLoopFindMode(rl, modeName, true);
        if (NULL != rlm && NULL == rlm->_observers) {
            //obs 都为空则创建一个默认参数的obs
            rlm->_observers = CFArrayCreateMutable(kCFAllocatorSystemDefault, 0, &kCFTypeArrayCallBacks);
        }
        if (NULL != rlm && !CFArrayContainsValue(rlm->_observers, CFRangeMake(0, CFArrayGetCount(rlm->_observers)), rlo)) {
                Boolean inserted = false;
                for (CFIndex idx = CFArrayGetCount(rlm->_observers); idx--; ) {
                    CFRunLoopObserverRef obs = (CFRunLoopObserverRef)CFArrayGetValueAtIndex(rlm->_observers, idx);
                    if (obs->_order <= rlo->_order) {
                        //根据优先级大小插入到idx=1 并退出
                        CFArrayInsertValueAtIndex(rlm->_observers, idx + 1, rlo);
                        inserted = true;
                        break;
                    }
                }
                if (!inserted) {
                    //若没插入成功则插入到索引为0的位置
                CFArrayInsertValueAtIndex(rlm->_observers, 0, rlo);
                }
            rlm->_observerMask |= rlo->_activities;
            __CFRunLoopObserverSchedule(rlo, rl, rlm);
        }
        if (NULL != rlm) {//解锁
	    __CFRunLoopModeUnlock(rlm);
        }
    }//解锁
    __CFRunLoopUnlock(rl);
}

```
runloop主函数：
```
void CFRunLoopRun(void) {	/* DOES CALLOUT */
    int32_t result;
    do {
        result = CFRunLoopRunSpecific(CFRunLoopGetCurrent(), kCFRunLoopDefaultMode, 1.0e10, false);
        CHECK_FOR_FORK();
    } while (kCFRunLoopRunStopped != result && kCFRunLoopRunFinished != result);
}
```
中间只是执行了一个`CFRunLoopRunSpecific`，那么我们看一下这个函数执行了什么东西？

```
static int32_t __CFRunLoopRun(CFRunLoopRef rl, CFRunLoopModeRef rlm, CFTimeInterval seconds, Boolean stopAfterHandle, CFRunLoopModeRef previousMode) {
    //开始运行时间
    uint64_t startTSR = mach_absolute_time();
//判断是否 loop已经stop
    if (__CFRunLoopIsStopped(rl)) {
        __CFRunLoopUnsetStopped(rl);
        return kCFRunLoopRunStopped;
    } else if (rlm->_stopped) {
        rlm->_stopped = false;
        return kCFRunLoopRunStopped;
    }
    //port
    mach_port_name_t dispatchPort = MACH_PORT_NULL;
    Boolean libdispatchQSafe = pthread_main_np() && ((HANDLE_DISPATCH_ON_BASE_INVOCATION_ONLY && NULL == previousMode) || (!HANDLE_DISPATCH_ON_BASE_INVOCATION_ONLY && 0 == _CFGetTSD(__CFTSDKeyIsInGCDMainQ)));
    if (libdispatchQSafe && (CFRunLoopGetMain() == rl) && CFSetContainsValue(rl->_commonModes, rlm->_name)) dispatchPort = _dispatch_get_main_queue_port_4CF();
    
#if USE_DISPATCH_SOURCE_FOR_TIMERS
    mach_port_name_t modeQueuePort = MACH_PORT_NULL;
    if (rlm->_queue) {
        modeQueuePort = _dispatch_runloop_root_queue_get_port_4CF(rlm->_queue);
        if (!modeQueuePort) {
            CRASH("Unable to get port for run loop mode queue (%d)", -1);
        }
    }
#endif
    
    dispatch_source_t timeout_timer = NULL;
    struct __timeout_context *timeout_context = (struct __timeout_context *)malloc(sizeof(*timeout_context));
    //超时
    if (seconds <= 0.0) { // instant timeout
        seconds = 0.0;
        timeout_context->termTSR = 0ULL;
    } else if (seconds <= TIMER_INTERVAL_LIMIT) {
        //全局队列
        dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, DISPATCH_QUEUE_OVERCOMMIT);
        //timer
        timeout_timer = dispatch_source_create(DISPATCH_SOURCE_TYPE_TIMER, 0, 0, queue);
            dispatch_retain(timeout_timer);
        timeout_context->ds = timeout_timer;
        timeout_context->rl = (CFRunLoopRef)CFRetain(rl);
        timeout_context->termTSR = startTSR + __CFTimeIntervalToTSR(seconds);
        dispatch_set_context(timeout_timer, timeout_context); // source gets ownership of context
        dispatch_source_set_event_handler_f(timeout_timer, __CFRunLoopTimeout);
            dispatch_source_set_cancel_handler_f(timeout_timer, __CFRunLoopTimeoutCancel);
            uint64_t ns_at = (uint64_t)((__CFTSRToTimeInterval(startTSR) + seconds) * 1000000000ULL);
            dispatch_source_set_timer(timeout_timer, dispatch_time(1, ns_at), DISPATCH_TIME_FOREVER, 1000ULL);
        //启动定时器 超时时间 forever
            dispatch_resume(timeout_timer);
    } else { // infinite timeout
        //初始化超时时间
        seconds = 9999999999.0;
        timeout_context->termTSR = UINT64_MAX;
    }

    Boolean didDispatchPortLastTime = true;
    int32_t retVal = 0;
    do {
        uint8_t msg_buffer[3 * 1024];
#if DEPLOYMENT_TARGET_MACOSX || DEPLOYMENT_TARGET_EMBEDDED || DEPLOYMENT_TARGET_EMBEDDED_MINI
        mach_msg_header_t *msg = NULL;
        mach_port_t livePort = MACH_PORT_NULL;
#endif
        //当前mode的 portset
        __CFPortSet waitSet = rlm->_portSet;

        __CFRunLoopUnsetIgnoreWakeUps(rl);
// 开始处理 timer
        if (rlm->_observerMask & kCFRunLoopBeforeTimers) __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeTimers);
        //开始处理 sources
        if (rlm->_observerMask & kCFRunLoopBeforeSources) __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeSources);

        __CFRunLoopDoBlocks(rl, rlm);
//处理source0
        Boolean sourceHandledThisLoop = __CFRunLoopDoSources0(rl, rlm, stopAfterHandle);
        if (sourceHandledThisLoop) {
            __CFRunLoopDoBlocks(rl, rlm);
        }

        Boolean poll = sourceHandledThisLoop || (0ULL == timeout_context->termTSR);
//第2次执行  第一次didDispatchPortLastTime = true
//        主线程中 dispatchPort = _dispatch_get_main_queue_port_4CF()
        if (MACH_PORT_NULL != dispatchPort && !didDispatchPortLastTime) {
#if DEPLOYMENT_TARGET_MACOSX || DEPLOYMENT_TARGET_EMBEDDED || DEPLOYMENT_TARGET_EMBEDDED_MINI
            msg = (mach_msg_header_t *)msg_buffer;
            if (__CFRunLoopServiceMachPort(dispatchPort, &msg, sizeof(msg_buffer), &livePort, 0)) {
                goto handle_msg;
            }
#endif
        }

        didDispatchPortLastTime = false;
//通知 开始 waiting
	if (!poll && (rlm->_observerMask & kCFRunLoopBeforeWaiting)) __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeWaiting);
        //睡眠
	__CFRunLoopSetSleeping(rl);
	// 休眠之后 什么也不做
        // 每个循环将本地端口 设置到激活端口
        // mode可以重新运行，而我们m不希望这些端口得到服务。

        __CFPortSetInsert(dispatchPort, waitSet);
        
	__CFRunLoopModeUnlock(rlm);
	__CFRunLoopUnlock(rl);

#if DEPLOYMENT_TARGET_MACOSX || DEPLOYMENT_TARGET_EMBEDDED || DEPLOYMENT_TARGET_EMBEDDED_MINI
#if USE_DISPATCH_SOURCE_FOR_TIMERS
        do {
            if (kCFUseCollectableAllocator) {
                objc_clear_stack(0);
                memset(msg_buffer, 0, sizeof(msg_buffer));
            }
            msg = (mach_msg_header_t *)msg_buffer;
            __CFRunLoopServiceMachPort(waitSet, &msg, sizeof(msg_buffer), &livePort, poll ? 0 : TIMEOUT_INFINITY);
            
            if (modeQueuePort != MACH_PORT_NULL && livePort == modeQueuePort) {
                //清空内部队列，如果其中一个标示timefire，则终端并维护计时器
                while (_dispatch_runloop_root_queue_perform_4CF(rlm->_queue));
                    if (rlm->_timerFired) {
                        // Leave livePort as the queue port, and service timers below
                        rlm->_timerFired = false;
                        break;
                    } else {
                        if (msg && msg != (mach_msg_header_t *)msg_buffer) free(msg);
                    }
            } else {
                // Go ahead and leave the inner loop.
                break;
            }
        } while (1);
#else
        if (kCFUseCollectableAllocator) {
            objc_clear_stack(0);
            memset(msg_buffer, 0, sizeof(msg_buffer));
        }
        msg = (mach_msg_header_t *)msg_buffer;
        __CFRunLoopServiceMachPort(waitSet, &msg, sizeof(msg_buffer), &livePort, poll ? 0 : TIMEOUT_INFINITY);
#endif
        
        __CFRunLoopLock(rl);
        __CFRunLoopModeLock(rlm);

        //删除端口 在每个runloop，每个mode可以重新运行，而我们不希望这些端口继续服务，另外不想让他们离开，如果函数返回则返回。

        __CFPortSetRemove(dispatchPort, waitSet);
        
        __CFRunLoopSetIgnoreWakeUps(rl);

        // user callouts now OK again
	__CFRunLoopUnsetSleeping(rl);
        //开始唤醒
	if (!poll && (rlm->_observerMask & kCFRunLoopAfterWaiting)) __CFRunLoopDoObservers(rl, rlm, kCFRunLoopAfterWaiting);
//标签：handle_msg
        handle_msg:;
        __CFRunLoopSetIgnoreWakeUps(rl);
        
        if (MACH_PORT_NULL == livePort) {
            CFRUNLOOP_WAKEUP_FOR_NOTHING();
            // handle nothing
        } else if (livePort == rl->_wakeUpPort) {
            CFRUNLOOP_WAKEUP_FOR_WAKEUP();
            // do nothing on Mac OS
        }
#if USE_DISPATCH_SOURCE_FOR_TIMERS
        else if (modeQueuePort != MACH_PORT_NULL && livePort == modeQueuePort) {
            CFRUNLOOP_WAKEUP_FOR_TIMER();
            if (!__CFRunLoopDoTimers(rl, rlm, mach_absolute_time())) {
                // 计算下次fire的时间
                __CFArmNextTimerInMode(rlm, rl);
            }
        }
#endif
#if USE_MK_TIMER_TOO
        else if (rlm->_timerPort != MACH_PORT_NULL && livePort == rlm->_timerPort) {
            CFRUNLOOP_WAKEUP_FOR_TIMER();
            // On Windows, we have observed an issue where the timer port is set before the time which we requested it to be set. For example, we set the fire time to be TSR 167646765860, but it is actually observed firing at TSR 167646764145, which is 1715 ticks early. The result is that, when __CFRunLoopDoTimers checks to see if any of the run loop timers should be firing, it appears to be 'too early' for the next timer, and no timers are handled.
            // In this case, the timer port has been automatically reset (since it was returned from MsgWaitForMultipleObjectsEx), and if we do not re-arm it, then no timers will ever be serviced again unless something adjusts the timer list (e.g. adding or removing timers). The fix for the issue is to reset the timer here if CFRunLoopDoTimers did not handle a timer itself. 9308754
            if (!__CFRunLoopDoTimers(rl, rlm, mach_absolute_time())) {
                // Re-arm the next timer
                __CFArmNextTimerInMode(rlm, rl);
            }
        }
#endif
        else if (livePort == dispatchPort) {
            CFRUNLOOP_WAKEUP_FOR_DISPATCH();
            __CFRunLoopModeUnlock(rlm);
            __CFRunLoopUnlock(rl);
            _CFSetTSD(__CFTSDKeyIsInGCDMainQ, (void *)6, NULL);
            __CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__(msg);
            _CFSetTSD(__CFTSDKeyIsInGCDMainQ, (void *)0, NULL);
	        __CFRunLoopLock(rl);
	        __CFRunLoopModeLock(rlm);
 	        sourceHandledThisLoop = true;
            didDispatchPortLastTime = true;
        } else {
            CFRUNLOOP_WAKEUP_FOR_SOURCE();
            // Despite the name, this works for windows handles as well
            CFRunLoopSourceRef rls = __CFRunLoopModeFindSourceForMachPort(rl, rlm, livePort);
            if (rls) {
#if DEPLOYMENT_TARGET_MACOSX || DEPLOYMENT_TARGET_EMBEDDED || DEPLOYMENT_TARGET_EMBEDDED_MINI
                    mach_msg_header_t *reply = NULL;
                    sourceHandledThisLoop = __CFRunLoopDoSource1(rl, rlm, rls, msg, msg->msgh_size, &reply) || sourceHandledThisLoop;
                    if (NULL != reply) {
                        (void)mach_msg(reply, MACH_SEND_MSG, reply->msgh_size, 0, MACH_PORT_NULL, 0, MACH_PORT_NULL);
                        CFAllocatorDeallocate(kCFAllocatorSystemDefault, reply);
                        }
#endif
                    }
            }
#if DEPLOYMENT_TARGET_MACOSX || DEPLOYMENT_TARGET_EMBEDDED || DEPLOYMENT_TARGET_EMBEDDED_MINI
        if (msg && msg != (mach_msg_header_t *)msg_buffer) free(msg);
#endif
        
	__CFRunLoopDoBlocks(rl, rlm);
        

        if (sourceHandledThisLoop && stopAfterHandle) {
                retVal = kCFRunLoopRunHandledSource;
            }
        else if (timeout_context->termTSR < mach_absolute_time()) {
                    retVal = kCFRunLoopRunTimedOut;
        } else if (__CFRunLoopIsStopped(rl)) {
                __CFRunLoopUnsetStopped(rl);
                retVal = kCFRunLoopRunStopped;
        } else if (rlm->_stopped) {
                rlm->_stopped = false;
                retVal = kCFRunLoopRunStopped;
        } else if (__CFRunLoopModeIsEmpty(rl, rlm, previousMode)) {
                retVal = kCFRunLoopRunFinished;
        }
    } while (0 == retVal);
//释放
    if (timeout_timer) {
        dispatch_source_cancel(timeout_timer);
        dispatch_release(timeout_timer);
    } else {
        free(timeout_context);
    }

    return retVal;
}

SInt32 CFRunLoopRunSpecific(CFRunLoopRef rl, CFStringRef modeName, CFTimeInterval seconds, Boolean returnAfterSourceHandled) {     /* DOES CALLOUT */
    CHECK_FOR_FORK();
    //如果销毁 则返回运行结束
    if (__CFRunLoopIsDeallocating(rl)) return kCFRunLoopRunFinished;
    __CFRunLoopLock(rl);//加锁
    //获取当前mode
    CFRunLoopModeRef currentMode = __CFRunLoopFindMode(rl, modeName, false);
    //当前mode 为空 或者 mode 中是空的
    if (NULL == currentMode || __CFRunLoopModeIsEmpty(rl, currentMode, rl->_currentMode)) {
        Boolean did = false;
        //解锁mode rl
        if (currentMode) __CFRunLoopModeUnlock(currentMode);
        __CFRunLoopUnlock(rl);
        return did ? kCFRunLoopRunHandledSource : kCFRunLoopRunFinished;
    }
    volatile _per_run_data *previousPerRun = __CFRunLoopPushPerRunData(rl);
    CFRunLoopModeRef previousMode = rl->_currentMode;
    rl->_currentMode = currentMode;
    int32_t result = kCFRunLoopRunFinished;

	if (currentMode->_observerMask & kCFRunLoopEntry ) __CFRunLoopDoObservers(rl, currentMode, kCFRunLoopEntry);
	result = __CFRunLoopRun(rl, currentMode, seconds, returnAfterSourceHandled, previousMode);
	if (currentMode->_observerMask & kCFRunLoopExit ) __CFRunLoopDoObservers(rl, currentMode, kCFRunLoopExit);

        __CFRunLoopModeUnlock(currentMode);
        __CFRunLoopPopPerRunData(rl, previousPerRun);
	rl->_currentMode = previousMode;
    __CFRunLoopUnlock(rl);
    return result;
}
```
上图中`runloop_1`中，第5不条件是`如果有source1则跳转9`,其实是不对的，看下源码：
```
//处理source0
        Boolean sourceHandledThisLoop = __CFRunLoopDoSources0(rl, rlm, stopAfterHandle);
        if (sourceHandledThisLoop) {
            __CFRunLoopDoBlocks(rl, rlm);
        }

        Boolean poll = sourceHandledThisLoop || (0ULL == timeout_context->termTSR);
//第2次执行  第一次didDispatchPortLastTime = true
//        主线程中 dispatchPort = _dispatch_get_main_queue_port_4CF()
        if (MACH_PORT_NULL != dispatchPort && !didDispatchPortLastTime) {
#if DEPLOYMENT_TARGET_MACOSX || DEPLOYMENT_TARGET_EMBEDDED || DEPLOYMENT_TARGET_EMBEDDED_MINI
            msg = (mach_msg_header_t *)msg_buffer;
            if (__CFRunLoopServiceMachPort(dispatchPort, &msg, sizeof(msg_buffer), &livePort, 0)) {
                goto handle_msg;
            }
#endif
        }
```
`dispatchPort`其实是主线程监测的端口，`didDispatchPortLastTime == fale`是也就是执行第二次的循环才产生的，默认是`true`，所以条件是主线程，第2次循环，则跳转`handle_msg`。
则完整的流程是：

```
1. 通知observer run loop被触发
  2. 如果有timers事件的话，通知observer
  3. 如果有source0要处理的话，通知observer
  4. 触发所有的准备完毕的source0
  5. 如果当前是主线程的runloop，并且主线程有事儿，跳到第9步
  6. 通知Observer runloop将进入sleep状态
  7. mach进入sleep和监听状态
  8. 通知observer，runloop被woke up
  9. 如果runloop是被唤醒，CFRUNLOOP_WAKEUP_FOR_WAKEUP
  10. 如果用户定义的timer被触发，处理event并重启RunLoop
  11. 如果dispatchPort，处理主线程
  12. 如果一个source1被触发，__CFRunLoopDoSource1
  13. 继续循环或通知observer runloop将要exited。
```
[demo注释下载](https://github.com/ifgyong/demo/blob/master/OC/test/test/CF-855.17/CFRunLoop.c)

文章有不足之处，希望各位看官大佬指出。


参考文章:
[源码下载](https://opensource.apple.com/tarballs/CF/)
[底层源码解析及其应用](https://www.jianshu.com/p/d6c5c0bf97fd)
[guide](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Multithreading/RunLoopManagement/RunLoopManagement.html)
[YY](https://blog.ibireme.com/2015/05/18/runloop/)
