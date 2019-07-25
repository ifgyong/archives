> 现在大部分iOS开发者的都已经知道消息转发机制了，但是到底消息转发，底层发生了什么事情呢？今天带大家探索一下底层`_objc_msgSend`的实现过程

我们找到头文件是这样子解释的：
```
/** 
 * Sends a message with a simple return value to an instance of a class.
 * 
 * @param self A pointer to the instance of the class that is to receive the message.
 * @param op The selector of the method that handles the message.
 * @param ... 
 *   A variable argument list containing the arguments to the method.
 * 
 * @return The return value of the method.
 * 
 * @note When it encounters a method call, the compiler generates a call to one of the
 *  functions \c objc_msgSend, \c objc_msgSend_stret, \c objc_msgSendSuper, or \c objc_msgSendSuper_stret.
 *  Messages sent to an object’s superclass (using the \c super keyword) are sent using \c objc_msgSendSuper; 
 *  other messages are sent using \c objc_msgSend. Methods that have data structures as return values
 *  are sent using \c objc_msgSendSuper_stret and \c objc_msgSend_stret.
 */
#if !OBJC_OLD_DISPATCH_PROTOTYPES
OBJC_EXPORT void
objc_msgSend(void /* id self, SEL op, ... */ )
    OBJC_AVAILABLE(10.0, 2.0, 9.0, 1.0, 2.0);
```
[点我一起下载runtime源码](https://opensource.apple.com/tarballs/objc4/)
下载好直接搜索`_objc_msgSend`
![搜索msgSend](https://upload-images.jianshu.io/upload_images/783986-7f0053a8ebd4a963.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
然后
![汇编语言找到](https://upload-images.jianshu.io/upload_images/783986-a449eaf4addd2ea6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
#### 这是汇编实现的obj_msgsend()

```
/********************************************************************
 *
 * id objc_msgSend(id self, SEL _cmd, ...);
 * IMP objc_msgLookup(id self, SEL _cmd, ...);
 * 
 * objc_msgLookup ABI:
 * IMP returned in r12
 * Forwarding returned in Z flag
 * r9 reserved for our use but not used
 *
 ********************************************************************/

	ENTRY _objc_msgSend
	
	cbz	r0, LNilReceiver_f //比较 r0==0 跳转LNilReceiver_f

	ldr	r9, [r0]		// r9 = self->isa
	GetClassFromIsa			// r9 = class 从isa获取class
	CacheLookup NORMAL //执行CacheLookup函数 从缓存中寻找
	// cache hit, IMP in r12, eq already set for nonstret forwarding
//如果找到  返回IMP
	bx	r12			// call imp
//否则继续寻找
	CacheLookup2 NORMAL
	// cache miss
	ldr	r9, [r0]		// r9 = self->isa
	GetClassFromIsa			// r9 = class
	b	__objc_msgSend_uncached//调用_objc_msgSend_uncached函数
```
大致思路是先从`CacheLookup `中寻找，找到则返回，找不到则继续寻找`CacheLookup 2`。下边我们探索一下`CacheLookup `的实现
大致思路是寻找cache，从`isa->class`寻找，循环找不到则跳出，找出来则返回`IMP`。

```
.macro CacheLookup
	// p1 = SEL, p16 = isa
	ldp	p10, p11, [x16, #CACHE]	// p10 = buckets, p11 = occupied|mask
#if !__LP64__
	and	w11, w11, 0xffff	// p11 = mask
#endif
	and	w12, w1, w11		// x12 = _cmd & mask
	add	p12, p10, p12, LSL #(1+PTRSHIFT)
		             // p12 = buckets + ((_cmd & mask) << (1+PTRSHIFT))

	ldp	p17, p9, [x12]		// {imp, sel} = *bucket
1:	cmp	p9, p1			// if (bucket->sel != _cmd)比较sel 和_cmd
	b.ne	2f			//     scan more没有找到则继续寻找跳转2楼
	CacheHit $0			// call or return imp 找到则返回imp
	
2:	// not hit: p12 = not-hit bucket 没有找到则继续寻找
	CheckMiss $0			// miss if bucket->sel == 0
	cmp	p12, p10		// wrap if bucket == buckets 比较bucket 和buckets
	b.eq	3f  //没有找到则继续寻找跳转3楼
	ldp	p17, p9, [x12, #-BUCKET_SIZE]!	// {imp, sel} = *--bucket
	b	1b			// loop

3:	// wrap: p12 = first bucket, w11 = mask
	add	p12, p12, w11, UXTW #(1+PTRSHIFT)
		                        // p12 = buckets + (mask << 1+PTRSHIFT)

	// Clone scanning loop to miss instead of hang when cache is corrupt.
	// The slow path may detect any corruption and halt later.

	ldp	p17, p9, [x12]		// {imp, sel} = *bucket
1:	cmp	p9, p1			// if (bucket->sel != _cmd)比较sel 和_cmd
	b.ne	2f			//     scan more//没有找到则继续寻找跳转2楼
	CacheHit $0			// call or return imp
	
2:	// not hit: p12 = not-hit bucket
	CheckMiss $0			// miss if bucket->sel == 0 如果没有sel
	cmp	p12, p10		// wrap if bucket == buckets如果找到
	b.eq	3f //跳转三楼
	ldp	p17, p9, [x12, #-BUCKET_SIZE]!	// {imp, sel} = *--bucket
	b	1b			// loop 跳转楼上1楼 循环寻找sel

3:	// double wrap
	JumpMiss $0 //结束
	
.endmacro

//寻找miss的时候执行此函数
.macro CheckMiss
	// miss if bucket->sel == 0
.if $0 == GETIMP
	cbz	p9, LGetImpMiss
.elseif $0 == NORMAL
//执行 __objc_msgSend_uncached 然后下边看此函数实现
	cbz	p9, __objc_msgSend_uncached
.elseif $0 == LOOKUP
	cbz	p9, __objc_msgLookup_uncached
.else
.abort oops
.endif
.endmacro
```
方法查找列表函数 跳转到`__class_lookupMethodAndLoadCache3`,然后后边看一下该函数实现
```
.macro MethodTableLookup
	
	// push frame
	SignLR
	stp	fp, lr, [sp, #-16]!
	mov	fp, sp

	// save parameter registers: x0..x8, q0..q7
	sub	sp, sp, #(10*8 + 8*16)
	stp	q0, q1, [sp, #(0*16)]
	stp	q2, q3, [sp, #(2*16)]
	stp	q4, q5, [sp, #(4*16)]
	stp	q6, q7, [sp, #(6*16)]
	stp	x0, x1, [sp, #(8*16+0*8)]
	stp	x2, x3, [sp, #(8*16+2*8)]
	stp	x4, x5, [sp, #(8*16+4*8)]
	stp	x6, x7, [sp, #(8*16+6*8)]
	str	x8,     [sp, #(8*16+8*8)]

	// receiver and selector already in x0 and x1
	mov	x2, x16
//跳转_class_lookupMethodAndLoadCache3 
//汇编跳转c/c++函数名字去掉一个下划线即可
	bl	__class_lookupMethodAndLoadCache3

	// IMP in x0
	mov	x17, x0
	
	// restore registers and return
	ldp	q0, q1, [sp, #(0*16)]
	ldp	q2, q3, [sp, #(2*16)]
	ldp	q4, q5, [sp, #(4*16)]
	ldp	q6, q7, [sp, #(6*16)]
	ldp	x0, x1, [sp, #(8*16+0*8)]
	ldp	x2, x3, [sp, #(8*16+2*8)]
	ldp	x4, x5, [sp, #(8*16+4*8)]
	ldp	x6, x7, [sp, #(8*16+6*8)]
	ldr	x8,     [sp, #(8*16+8*8)]

	mov	sp, fp
	ldp	fp, lr, [sp], #16
	AuthenticateLR

```
现在找到了该函数：
![函数](https://upload-images.jianshu.io/upload_images/783986-0c940d7f3b45a427.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

跳转到了c实现的`objc-runtime-new.mm`中，现在又到了我们熟悉的`runtime`。
现在我们分析一下该函数：
```
IMP _class_lookupMethodAndLoadCache3(id obj, SEL sel, Class cls)
{
//cls 是class obj是消息接受者，sel是函数名字
//为什么一个是YES？第二个是NO？第三个是YES？
//因为第一个是在initialize还没有搜索过，第二个是cache已经搜索过，第三个是消息是否需要消息转发。
    return lookUpImpOrForward(cls, sel, obj, 
                              YES/*initialize*/, NO/*cache*/, YES/*resolver*/);
}
```
我们现在展开该函数：
```
IMP lookUpImpOrForward(Class cls, SEL sel, id inst, 
                       bool initialize, bool cache, bool resolver)
{
    IMP imp = nil;
    bool triedResolver = NO;

    runtimeLock.assertUnlocked();

    // 如果cache yes 去查找缓存
    if (cache) {
        imp = cache_getImp(cls, sel);
        if (imp) return imp;
    }

    runtimeLock.lock();
//检查class是否存在 否的话 直接崩溃
    checkIsKnownClass(cls);
// 没有该属性class_rw_t 则初始化
    if (!cls->isRealized()) {
        realizeClass(cls);
    }
// 该 initialized 是yes，而且没有实现则初始化initialize
    if (initialize  &&  !cls->isInitialized()) {
        runtimeLock.unlock();
        _class_initialize (_class_getNonMetaClass(cls, inst));
        runtimeLock.lock();
    }

    
 retry:    
    runtimeLock.assertLocked();

    // Try this class's cache.
//方式动态添加的方法跳过 再次查找缓存
    imp = cache_getImp(cls, sel);
    if (imp) goto done;

    // Try this class's method lists.
//class的方法列表中查找
    {
        Method meth = getMethodNoSuper_nolock(cls, sel);
        if (meth) {//添加到缓存中方便下次查找
            log_and_fill_cache(cls, meth->imp, sel, inst, cls);
            imp = meth->imp;
            goto done;//直接标签跳转done
        }
    }

    // Try superclass caches and method lists.
    {//查找父类的缓存和方法列表
        unsigned attempts = unreasonableClassCount();
//循环迭代查找父类是否实现该函数
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

    // No implementation found. Try method resolver once.

    if (resolver  &&  !triedResolver) {
        runtimeLock.unlock();
//消息动态解析
        _class_resolveMethod(cls, sel, inst);
        runtimeLock.lock();
        triedResolver = YES;//标记查找过了resolveMethod
        goto retry;
    }

    // No implementation found, and method resolver didn't help. 
    // 动态解析下一步 forwarding.

    imp = (IMP)_objc_msgForward_impcache;
    cache_fill(cls, sel, imp, inst);

 done:
    runtimeLock.unlock();

    return imp;
}
```
动态解析架构图：
![动态解析架构图](https://upload-images.jianshu.io/upload_images/783986-06f6b6b455d9ae93.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

动态解析第一步：
```
void _class_resolveMethod(Class cls, SEL sel, id inst)
{//判断是否是元类
    if (! cls->isMetaClass()) {
        _class_resolveInstanceMethod(cls, sel, inst);
    } 
    else {
        _class_resolveClassMethod(cls, sel, inst);
        if (!lookUpImpOrNil(cls, sel, inst, 
                            NO/*initialize*/, YES/*cache*/, NO/*resolver*/)) 
        {
            _class_resolveInstanceMethod(cls, sel, inst);
        }
    }
}
```

```
@implementation Person
void fy_sayHello(){
    NSLog(@"执行动态添加的方法");
}
//第一次机会
//+ (BOOL)resolveInstanceMethod:(SEL)sel{
//    NSString *sel_str=NSStringFromSelector(sel);
//    if (sel) {
//        NSLog(@"动态添加方法 %@",sel_str);
//        class_addMethod([self class], sel, (IMP)fy_sayHello, "v@:");
//    }
//    return [super resolveClassMethod:sel];
//}
//第二次机会
-(id)forwardingTargetForSelector:(SEL)aSelector{
    NSLog(@"forwatdingtargent:%s",__func__);
    if (aSelector == @selector(sayHello)) {
        return [Dog new];
    }
    return [super forwardingTargetForSelector:aSelector];
}
//-(void)haha{
//    NSLog(@"haha 老弟");
//}
////第三次机会
//- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector{
//        if (aSelector) {
//            Method meth= class_getInstanceMethod([self class], @selector(haha));
//            const char *type = method_getTypeEncoding(meth);
//            return [NSMethodSignature signatureWithObjCTypes:type];
//        }
//    return [NSMethodSignature methodSignatureForSelector:aSelector];
//}
//-(void)forwardInvocation:(NSInvocation *)anInvocation{
//    NSLog(@"%s %@",__func__,NSStringFromSelector(anInvocation.selector));
//    anInvocation.selector = @selector(haha);
//    [anInvocation invoke];
//    
//}
```
现在我们了解了消息转发，那么我么来做一个拦截一个没有实现函数体的消息的功能：
首先新建一个类别？那么这个类别基于谁呢？当然基于基类了，否则怎么拦截所有对象的消息呢？
具体代码.h如下：
```

#import <Foundation/Foundation.h>
#import <objc/runtime.h>
NS_ASSUME_NONNULL_BEGIN

@interface NSObject (Add)
//记录发送的类的sel名字
@property (nonnull,copy) NSString *unmethodName;

@end

NS_ASSUME_NONNULL_END
```
实现如下：
```
#import "NSObject+Add.h"

@implementation NSObject (Add)
-(void)handleNoMethod{
    NSLog(@"没有实现该方法: %@ %@",NSStringFromClass([self class]),self.unmethodName);
}
//第三次机会
//消除警告
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Wobjc-protocol-method-implementation"

// 被夹在这中间的代码针对于此警告都会忽视不显示出来

//常见警告的名称
//1.声明变量未使用  "-Wunused-variable"
//2.方法定义未实现  "-Wincomplete-implementation"
//3.未声明的选择器  "-Wundeclared-selector"
//4.参数格式不匹配  "-Wformat"
//5.废弃掉的方法     "-Wdeprecated-declarations"
//6.不会执行的代码  "-Wunreachable-code"
//7.忽略在arc 环境下performSelector产生的 leaks 的警告 "-Warc-performSelector-leaks"
//8.忽略类别方法覆盖的警告 "-Wobjc-protocol-method-implementation"（修复开源库bug，覆盖开源库方法时会用到）
//给类别新增属性
-(void)setUnmethodName:(NSString *)unmethodName{
    objc_setAssociatedObject(self, "unmethodName", unmethodName, OBJC_ASSOCIATION_COPY);
}
- (NSString *)unmethodName{
    return  objc_getAssociatedObject(self,"unmethodName");
}
//函数签名
- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector{
    if (aSelector) {
        Method meth= class_getInstanceMethod([self class], @selector(handleNoMethod));
        const char *type = method_getTypeEncoding(meth);
        return [NSMethodSignature signatureWithObjCTypes:type];
    }
    return [NSMethodSignature methodSignatureForSelector:aSelector];
}
//回调函数
-(void)forwardInvocation:(NSInvocation *)anInvocation{
    NSLog(@"%s %@",__func__,NSStringFromSelector(anInvocation.selector));
    self.unmethodName = NSStringFromSelector(anInvocation.selector);
    anInvocation.selector = @selector(handleNoMethod);
    [anInvocation invoke];
}
#pragma clang diagnostic pop
@end
```
看一下打印结果：
![拦截结果](https://upload-images.jianshu.io/upload_images/783986-72848d97af303fca.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


当然大家也可以利用这个做点其他事情，比如像 切面编程的Aspect，就是利用该特性添加和交换函数。











