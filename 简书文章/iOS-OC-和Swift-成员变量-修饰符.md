现讲一下属性和成员两变量的区别？好多人只知道`@property (nonatomic,assign) NSInteger age;
`，其实`property = getter + setter +ivar`。一个getter方法，一个是setter方法，另外一个是成员变量，ivar就是成员变量。
我们定义2个`property`和几个`ivar`，然后使用`runtime`取出来确认一下他们的关系；
看代码：
```
- (void)viewDidLoad {
    [super viewDidLoad];
    
    unsigned int  count = 0;
    
    objc_property_t *propertyList = class_copyPropertyList([Person class], &count);
    for (int i  =0; i < count; i ++) {
        objc_property_t p = propertyList[i];
        NSLog(@" %s",property_getName(p));
    }
    free(propertyList);
    
    [self getAllIvarList];
    [self getAllMethodList];
    
}
- (void) getAllIvarList {
    unsigned int methodCount = 0;
    Ivar * ivars = class_copyIvarList([Person class], &methodCount);
    for (unsigned int i = 0; i < methodCount; i ++) {
        Ivar ivar = ivars[i];
        const char * name = ivar_getName(ivar);
        const char*type = ivar_getTypeEncoding(ivar);
        NSLog(@"成员变量的类型为%s，名字为 %s ",type, name);
    }
    free(ivars);
}
- (void) getAllMethodList{
    unsigned int count = 0;
    Method *methods = class_copyMethodList([Person class], &count);
    for (int i = 0; i < count; i ++) {
        Method m = methods[i];
        SEL sel = method_getName(m);
        NSLog(@"method:%@",NSStringFromSelector(sel));
    }
}
```
然后在看一下输出的参数：
```
age
name
成员变量的类型为@"NSString"，名字为 publicName 
成员变量的类型为@"NSString"，名字为 privateName 
成员变量的类型为@"NSString"，名字为 protectedName 
成员变量的类型为@"NSString"，名字为 packageName 
成员变量的类型为q，名字为 _age 
成员变量的类型为@"NSString"，名字为 _name 
method:initWithDic:
method:.cxx_destruct
method:name
method:setName:
method:init
method:setAge:
method:age
```
##### 通过代码验证可以知道的是定义的`property`都是成对出现的setter和getter方法，而成员变量则没生成。


那么成员变量一般写在匿名类里边，然后.m文件内容在外部无法访问的，造成了修饰符没有起到作用。如下图：
先看一下Objec-C：
```
#import "Person.h"

@interface Person ()
{
@public
    NSString *publicName;
@private
    NSString *privateName;
@protected
    NSString * protectedName;
    @package
    NSString *packageName;
}
@end
```
因为本身在.m中，成员变量本身在外部就是访问不到的，用`public`修饰也没用。

正确的是：
```
#import <Foundation/Foundation.h>

NS_ASSUME_NONNULL_BEGIN

@interface Person : NSObject
{
@public
    NSString *publicName;
@private
    NSString *privateName;
@protected
    NSString * protectedName;
    @package
    NSString *packageName;
}
```

然后我们测试一下这4个层级的范围：
![4个层级的范围](https://upload-images.jianshu.io/upload_images/783986-2602f36dc5f41fa3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![4个层级的范围](https://upload-images.jianshu.io/upload_images/783986-614a36873c9ea6b9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看到编译器直接报错，`private`是仅仅当前类和类别中可用，子类都不能用，范围最小。

![protected](https://upload-images.jianshu.io/upload_images/783986-666b7603c7535c33.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

@protected  在外部不可以访问，只能在子类和自己类，类别中访问。

![public](https://upload-images.jianshu.io/upload_images/783986-f791b39a0b413ed6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
`@public  @package`在自己生成的framework外部可以访问，和public类似。
那么我们再用KVC访问各个级别的看看能访问吗？

初始化`Person`
```
- (instancetype)init{
    self = [super init];
    if (self) {
        self->privateName = @"privateName";
        self->protectedName = @"protectedName";
        self->packageName = @"packageName";
        self->publicName = @"publicName";
    }
    return self;
}
```
通过KVC访问成员变量：
![KVC](https://upload-images.jianshu.io/upload_images/783986-34e6ae73da9a462d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



###### 使用KVC访问framework中的成员变量测试：
![framework](https://upload-images.jianshu.io/upload_images/783986-f2a0ba4f77a7294c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
##### 经过测试私有和受保护的使用KVC也可以访问和赋值的，其实系统中有些成员变量我们也可以冲过使用这个方式来访问。
##### 其实根据测试，OC类中和子类KVC都可访问所有成员变量。

测试版本：Xcode 10.2.1

> @private私有的
代表私有，也就是只有自己有，别人谁都不可用，不不可以继承的。

> @protected受保护的
相较上边的private而言，就没有那么自私了，他自己可以用，自己的子类也是可以共享的,是可以继承的.

> @public公共的
相较上边而言，谁都可以用，只要你有这个类的对象，就可以拿到public下的变量，

> @package包
这个主要是用于框架类，使用@private太限制，使用@protected或者@public又太开放，就使用这个package吧。

那么我们在看一下Swift 5.0：
private
>  访问级别所修饰的属性或者方法只能在当前类里访问

fileprivate
>  访问级别所修饰的属性或者方法在当前的 Swift 源文件里可以访问

internal
> 访问级别所修饰的属性或方法在源代码所在的整个模块都可以访问
如果是框架或者库代码，则在整个框架内部都可以访问，框架由外部代码所引用时，则不可以访问

public
> 可以被任何人访问。但其他 module 中不可以被 override 和继承，而在 module 内可以被 override 和继承

open
> 可以被任何人使用，包括 override 和继承

##### 从高到低排序如下：
open > public > interal > fileprivate > private


Swift测试代码：
```
import Foundation
class Car: NSObject {
    var name = "car_name"
    private var privateName = "privateName"
    public var publicName = "publicName"
    fileprivate var fileprivateName = "fileprivateName"
    open var openName = "openName"
}
class Car2: Car {
    override init() {
        super.init()
        self.name = ""
        self.publicName = ""
        self.fileprivateName = ""
        self.openName = ""
===============本行报错 ===============
        self.privateName = ""
    }
    func loadCar() -> Void {
        let car = Car()
        car.fileprivateName = ""
        car.publicName = ""
        car.name = ""
        car.openName = ""
    }
}
```
![image.png](https://upload-images.jianshu.io/upload_images/783986-aa5d74e1cb742361.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
然后使用swift制作framework之后测试如下：
测试打包前源代码：
```
open class Car: NSObject {
    var name = "car_name"
    private var privateName = "privateName"
    public var publicName = "publicName"
    fileprivate var fileprivateName = "fileprivateName"
    open var openName = "openName"
    
    open func work() -> Void{
        print("car work")
    }
    public func publicWork() -> Void{
        print("car publicwork")
    }
}
public class Car2: Car {
    override init() {
        super.init()
        self.name = ""
        self.publicName = ""
        self.fileprivateName = ""
        self.openName = ""
//        self.privateName = ""
    }
    override public func work() {
        
    }
    override public func publicWork() {
        
    }
    func loadCar() -> Void {
        let car = Car()
        car.fileprivateName = ""
        car.publicName = ""
        car.name = ""
        car.openName = ""
    }
}
```
![open and public](https://upload-images.jianshu.io/upload_images/783986-4b373b15d9c5c7db.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

编译之前：
```
    var name = "car_name"
    private var privateName = "privateName"
    public var publicName = "publicName"
    fileprivate var fileprivateName = "fileprivateName"
    open var openName = "openName"
```
编译之后：
```
open class Car : NSObject {

    public var publicName: String

    open var openName: String

    open func work()

    public func publicWork()
}
```
打包成framework，则public和open 修饰的属性可见，其他的则不在model中。


参考文档：
[官方文档](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjectiveC/Chapters/ocDefiningClasses.html#//apple_ref/doc/uid/TP30001163-CH12-SW2)
