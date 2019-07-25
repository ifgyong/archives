上篇文章讲过了`runloop`的含义，那么我们就实战演练一下。
首先看一下系统提供的函数：
```
//获取当前和主runloop
CF_EXPORT CFRunLoopRef CFRunLoopGetCurrent(void);
CF_EXPORT CFRunLoopRef CFRunLoopGetMain(void);

typedef CFStringRef CFRunLoopMode CF_EXTENSIBLE_STRING_ENUM;

typedef struct CF_BRIDGED_MUTABLE_TYPE(id) __CFRunLoop * CFRunLoopRef;

typedef struct CF_BRIDGED_MUTABLE_TYPE(id) __CFRunLoopSource * CFRunLoopSourceRef;

typedef struct CF_BRIDGED_MUTABLE_TYPE(id) __CFRunLoopObserver * CFRunLoopObserverRef;

typedef struct CF_BRIDGED_MUTABLE_TYPE(NSTimer) __CFRunLoopTimer * CFRunLoopTimerRef;
```
用到的对象基本是这几种。
创建对象函数：
```
//是否包含source
CF_EXPORT Boolean CFRunLoopContainsSource(CFRunLoopRef rl, CFRunLoopSourceRef source, CFRunLoopMode mode);
//add source
CF_EXPORT void CFRunLoopAddSource(CFRunLoopRef rl, CFRunLoopSourceRef source, CFRunLoopMode mode);
//删除source
CF_EXPORT void CFRunLoopRemoveSource(CFRunLoopRef rl, CFRunLoopSourceRef source, CFRunLoopMode mode);
//是否包含 Observer
CF_EXPORT Boolean CFRunLoopContainsObserver(CFRunLoopRef rl, CFRunLoopObserverRef observer, CFRunLoopMode mode);
//添加 Observer
CF_EXPORT void CFRunLoopAddObserver(CFRunLoopRef rl, CFRunLoopObserverRef observer, CFRunLoopMode mode);
//删除 Observer
CF_EXPORT void CFRunLoopRemoveObserver(CFRunLoopRef rl, CFRunLoopObserverRef observer, CFRunLoopMode mode);
//是否包含 Timer
CF_EXPORT Boolean CFRunLoopContainsTimer(CFRunLoopRef rl, CFRunLoopTimerRef timer, CFRunLoopMode mode);
//添加 Timer
CF_EXPORT void CFRunLoopAddTimer(CFRunLoopRef rl, CFRunLoopTimerRef timer, CFRunLoopMode mode);
//删除 Timer
CF_EXPORT void CFRunLoopRemoveTimer(CFRunLoopRef rl, CFRunLoopTimerRef timer, CFRunLoopMode mode);
```

创建好之后，需要跑起来：

```
//拷贝当前mode
CF_EXPORT CFRunLoopMode CFRunLoopCopyCurrentMode(CFRunLoopRef rl);
//拷贝所有mode
CF_EXPORT CFArrayRef CFRunLoopCopyAllModes(CFRunLoopRef rl);
//添加mode 到runloop
CF_EXPORT void CFRunLoopAddCommonMode(CFRunLoopRef rl, CFRunLoopMode mode);
//获取下次fire的时间
CF_EXPORT CFAbsoluteTime CFRunLoopGetNextTimerFireDate(CFRunLoopRef rl, CFRunLoopMode mode);
//运行loop
CF_EXPORT void CFRunLoopRun(void);
//在mode中运行runloop
CF_EXPORT CFRunLoopRunResult CFRunLoopRunInMode(CFRunLoopMode mode, CFTimeInterval seconds, Boolean returnAfterSourceHandled);
//查看runloop是否在等待
CF_EXPORT Boolean CFRunLoopIsWaiting(CFRunLoopRef rl);
//手动唤醒loop
CF_EXPORT void CFRunLoopWakeUp(CFRunLoopRef rl);
//手动结束loop
CF_EXPORT void CFRunLoopStop(CFRunLoopRef rl);
```
数据结构：
```
typedef CF_ENUM(SInt32, CFRunLoopRunResult) {
    kCFRunLoopRunFinished = 1,//完成
    kCFRunLoopRunStopped = 2,//停止
    kCFRunLoopRunTimedOut = 3,//超时
    kCFRunLoopRunHandledSource = 4//处理source
};

/* Run Loop Observer Activities */
typedef CF_OPTIONS(CFOptionFlags, CFRunLoopActivity) {
    kCFRunLoopEntry = (1UL << 0),//进入loop
    kCFRunLoopBeforeTimers = (1UL << 1),//开始处理timers
    kCFRunLoopBeforeSources = (1UL << 2),//开始处理sources
    kCFRunLoopBeforeWaiting = (1UL << 5),//开始等待
    kCFRunLoopAfterWaiting = (1UL << 6),//唤醒
    kCFRunLoopExit = (1UL << 7),//退出
    kCFRunLoopAllActivities = 0x0FFFFFFFU//所有活动
};

```
source和sources1的数据结构：
```
typedef struct {
    CFIndex	version;
    void *	info;
    const void *(*retain)(const void *info);
    void	(*release)(const void *info);
    CFStringRef	(*copyDescription)(const void *info);
    Boolean	(*equal)(const void *info1, const void *info2);
    CFHashCode	(*hash)(const void *info);
    void	(*schedule)(void *info, CFRunLoopRef rl, CFRunLoopMode mode);
    void	(*cancel)(void *info, CFRunLoopRef rl, CFRunLoopMode mode);
    void	(*perform)(void *info);
} CFRunLoopSourceContext;

typedef struct {
    CFIndex	version;
    void *	info;
    const void *(*retain)(const void *info);
    void	(*release)(const void *info);
    CFStringRef	(*copyDescription)(const void *info);
    Boolean	(*equal)(const void *info1, const void *info2);
    CFHashCode	(*hash)(const void *info);
#if (TARGET_OS_MAC && !(TARGET_OS_EMBEDDED || TARGET_OS_IPHONE)) || (TARGET_OS_EMBEDDED || TARGET_OS_IPHONE)
    mach_port_t	(*getPort)(void *info);
    void *	(*perform)(void *msg, CFIndex size, CFAllocatorRef allocator, void *info);
#else
    void *	(*getPort)(void *info);
    void	(*perform)(void *info);
#endif
} CFRunLoopSourceContext1;


typedef struct {
    CFIndex	version;
    void *	info;
    const void *(*retain)(const void *info);
    void	(*release)(const void *info);
    CFStringRef	(*copyDescription)(const void *info);
} CFRunLoopObserverContext;

typedef struct {
    CFIndex	version;
    void *	info;
    const void *(*retain)(const void *info);
    void	(*release)(const void *info);
    CFStringRef	(*copyDescription)(const void *info);
} CFRunLoopTimerContext;
```
`source0`：event事件，只含有回调，需要标记待处理（signal），然后手动将runloop唤醒（wakeup）；
`source1` ：包含一个 mach_port 和一个回调，被用于通过内核和其他线程发送的消息，能主动唤醒runloop
系统暴露接口基本就这些，那么我们该如何使用呢？


开发我们经常用的`timer`，在有`scrollview`页面中经常出现，滑动的时候,`timer`好像卡了，就是一个`runloop`同时只能处理一个事件，那么怎么解决这个问题呢？
首先先创建一个循环观察器：
```
static void myRunLoopObserver(CFRunLoopObserverRef observer,  CFRunLoopActivity activity, void *info){
    switch (activity) {
        case kCFRunLoopEntry:
            NSLog(@"下一个loop 开始");
            break;
        case kCFRunLoopExit:
            NSLog(@"loop 退出了");
            break;
        case kCFRunLoopAfterWaiting:
            NSLog(@"kCFRunLoop  醒来了");
            break;
        case kCFRunLoopBeforeWaiting:
        {
            NSLog(@"kCFRunLoop 干完活了");
        }
            break;
        case kCFRunLoopBeforeTimers:
            NSLog(@"kCFRunLoopBeforeTimers");
            break;
        case kCFRunLoopBeforeSources:
            NSLog(@"kCFRunLoopBeforeSources");
            break;
        default:
            break;
    }
}
- (void)threadMain
{

    // The application uses garbage collection, so no autorelease pool is needed.
    NSRunLoop* myRunLoop = [NSRunLoop currentRunLoop];
    
    // Create a run loop observer and attach it to the run loop.
    CFRunLoopObserverContext  context = {0, (__bridge void*)self, NULL, NULL, NULL};
    CFRunLoopObserverRef    observer = CFRunLoopObserverCreate(kCFAllocatorDefault,
                                                               kCFRunLoopAllActivities, YES, 0, &myRunLoopObserver, &context);
       
    
    if (observer)
    {
        CFRunLoopRef    cfLoop = [myRunLoop getCFRunLoop];
        CFRunLoopAddObserver(cfLoop, observer, kCFRunLoopDefaultMode);
    }
    
    // Create and schedule the timer.
    [NSTimer scheduledTimerWithTimeInterval:1
                                     target:self
                                   selector:@selector(myDoFireTimer1:)
                                   userInfo:nil
                                    repeats:YES];
    
    NSInteger    loopCount = 3;
    do
    {
        // Run the run loop 10 times to let the timer fire.
        [myRunLoop runUntilDate:[NSDate dateWithTimeIntervalSinceNow:1]];
        loopCount--;
    }
    while (loopCount);
}
- (void)myDoFireTimer1:(NSObject *)obj{
    NSLog(@"%s",__func__);
}
```
输出：
```
下一个loop 开始
 kCFRunLoopBeforeTimers
 kCFRunLoopBeforeSources
 loop 退出了
 下一个loop 开始
 kCFRunLoopBeforeTimers
 kCFRunLoopBeforeSources
 kCFRunLoop 干完活了
 kCFRunLoop  醒来了
 loop 退出了
 下一个loop 开始
 kCFRunLoopBeforeTimers
 kCFRunLoopBeforeSources
 kCFRunLoop 干完活了
 kCFRunLoop  醒来了
 loop 退出了
```


#### 使用Core Foundation创建和计划计时器
```
- (void)testRunloopTimer{
    CFRunLoopRef loop = CFRunLoopGetCurrent();
    CFRunLoopTimerContext context = {0,NULL,NULL,NULL,NULL};
    CFRunLoopTimerRef timer = CFRunLoopTimerCreate(kCFAllocatorDefault, 0.1, 0.5, 0, 0, &timercallback, &context);
    if (CFRunLoopContainsTimer(loop, timer, kCFRunLoopCommonModes) == false) {
        CFRunLoopAddTimer(loop, timer, kCFRunLoopCommonModes);
    }
    const char  * str= [NSDate dateWithTimeIntervalSinceNow:0].description.UTF8String;

    printf("%s", str);
}
void timercallback(){
    static int count = 0;
    const char  * str= [NSDate dateWithTimeIntervalSinceNow:0].description.UTF8String;
    printf(" %s %d\n",str,count++);
}
```
