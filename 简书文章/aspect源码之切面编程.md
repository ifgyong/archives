Aspect Oriented Programming(AOP)，面向切面[编程](https://baike.baidu.com/item/%E7%BC%96%E7%A8%8B)，是一个比较热门的话题。AOP主要实现的目的是针对业务处理过程中的切面进行提取，它所面对的是处理过程中的某个步骤或阶段，以获得逻辑过程中各部分之间低耦合性的隔离效果。比如我们最常见的就是日志记录了，举个例子，我们现在提供一个查询学生信息的服务，但是我们希望记录有谁进行了这个查询。如果按照传统的OOP的实现的话，那我们实现了一个查询学生信息的服务接口(StudentInfoService)和其实现 类 （StudentInfoServiceImpl），同时为了要进行记录的话，那我们在实现类(StudentInfoServiceImpl)中要添加其实现记录的过程。这样的话，假如我们要实现的服务有多个呢？那就要在每个实现的类都添加这些记录过程。这样做的话就会有点繁琐，而且每个实现类都与记录服务日志的行为紧耦合，违反了面向对象的规则。那么怎样才能把记录服务的行为与业务处理过程中分离出来呢？看起来好像就是查询学生的服务自己在进行，但却是背后日志记录对这些行为进行记录，并且查询学生的服务不知道存在这些记录过程，这就是我们要讨论AOP的目的所在。AOP的编程，好像就是把我们在某个方面的功能提出来与一批对象进行隔离，这样与一批对象之间降低了耦合性，可以就某个功能进行编程。


实现切面编程，拦截类的函数和实例的函数可以通过框架[Aspects](https://github.com/steipete/Aspects)来实现，下面来介绍一下结构和关键实现函数。
框架的架构基本如下：
![架构图](https://upload-images.jianshu.io/upload_images/783986-b363d80adce7eef6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

入口函数是`+ (id<AspectToken>)aspect_hookSelector:(SEL)selector
                      withOptions:(AspectOptions)options
                       usingBlock:(id)block
                            error:(NSError **)error;
`然后调用了一个`c`函数来实现`hook`的过程。
那么我们看一下hook的大概实现
```
static id aspect_add(id self, SEL selector, AspectOptions options, id block, NSError **error) {
//判断三个参数是否为空
    NSCParameterAssert(self);
    NSCParameterAssert(selector);
    NSCParameterAssert(block);

    __block AspectIdentifier *identifier = nil;
//加锁
    aspect_performLocked(^{
        if (aspect_isSelectorAllowedAndTrack(self, selector, options, error)) {
//获取和创建AspectsContainer，其实AspectsContainer就是存储在实例中
//的信息，包括hook的三个数组函数列表
            AspectsContainer *aspectContainer = aspect_getContainerForObject(self, selector);
            identifier = [AspectIdentifier identifierWithSelector:selector object:self options:options block:block error:error];
            if (identifier) {
//addAspect:identifier添加到aspectContainer
                [aspectContainer addAspect:identifier withOptions:options];
                // hook的实现函数
                aspect_prepareClassAndHookSelector(self, selector, error);
            }
        }
    });
    return identifier;
}
```
上边是准备实现hook的准备工作
利用黑魔法`class_addMethod`和`class_replaceMethod`函数实现是拦截了`forwardInvocation:`，根据`self`的`aspectContainer`存的函数数组来分别执行`before` 和`instand` 和`after`三个不同切面的函数。
拦截函数是进入到下边函数：
```
static void __ASPECTS_ARE_BEING_CALLED__(__unsafe_unretained NSObject *self, SEL selector, NSInvocation *invocation) {
    NSCParameterAssert(self);
    NSCParameterAssert(invocation);
    SEL originalSelector = invocation.selector;
	SEL aliasSelector = aspect_aliasForSelector(invocation.selector);
    invocation.selector = aliasSelector;
//获取objectContainer和classContainer
    AspectsContainer *objectContainer = objc_getAssociatedObject(self, aliasSelector);
    AspectsContainer *classContainer = aspect_getContainerForClass(object_getClass(self), aliasSelector);
    AspectInfo *info = [[AspectInfo alloc] initWithInstance:self invocation:invocation];
    NSArray *aspectsToRemove = nil;
    // hooks之前的函数先执行class 
    aspectsToRemove = aspect_invoke(classContainer.beforeAspects, info,aspectsToRemove);
// hooks之前的函数先执行实例
    aspectsToRemove = aspect_invoke(objectContainer.beforeAspects, info,aspectsToRemove);
    // hooks的函数执行
    BOOL respondsToAlias = YES;
    if (objectContainer.insteadAspects.count || classContainer.insteadAspects.count) {
        aspectsToRemove =aspect_invoke(classContainer.insteadAspects, info,aspectsToRemove);
        aspectsToRemove=aspect_invoke(objectContainer.insteadAspects, info,aspectsToRemove);
    }else {
        Class klass = object_getClass(invocation.target);
        do {
            if ((respondsToAlias = [klass instancesRespondToSelector:aliasSelector])) {
                [invocation invoke];
                break;
            }
        }while (!respondsToAlias && (klass = class_getSuperclass(klass)));
    }
    // hooks之后的函数执行class
    aspectsToRemove=aspect_invoke(classContainer.afterAspects, info,aspectsToRemove);
    //  hooks之后的函数执行实例
    aspectsToRemove=aspect_invoke(objectContainer.afterAspects, info,aspectsToRemove);
//如果没有钩子，则调用原始函数 一般引发异常，因为有钩子才会走到这里。
    if (!respondsToAlias) {
        invocation.selector = originalSelector;
        SEL originalForwardInvocationSEL = NSSelectorFromString(AspectsForwardInvocationSelectorName);
        if ([self respondsToSelector:originalForwardInvocationSEL]) {
            ((void( *)(id, SEL, NSInvocation *))objc_msgSend)(self, originalForwardInvocationSEL, invocation);
        }else {
            [self doesNotRecognizeSelector:invocation.selector];
        }
    }
    // 排队等待销毁执行 remove函数
    [aspectsToRemove makeObjectsPerformSelector:@selector(remove)];
}
```
到此已经函数执行完毕。我们可以利用`Aspects`实现一些东西比如：
- 无痕埋点
- 热更新
- 热门数据统计

