> 这个问题是基础也是容易忘记的地方，现在从新探讨一下该问题

##### 一. Load和Initialize的往死了问是一种怎样的体验？
Load 和 Initialize 先加载哪个？
父类和子类以及 Category 的关系？
如果是多个 Category 呢
首先我们查看一下官方文档：
![+load](https://upload-images.jianshu.io/upload_images/783986-5d615a31255eb912.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![+initialize](https://upload-images.jianshu.io/upload_images/783986-268bb862d371a562.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
看完文档我们校验一下：
code：
```
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
@property (nonatomic,assign) NSInteger age;
@property (nonatomic,copy) NSString *name;
- (instancetype)initWithDic:(NSDictionary *)dic;

@end
@interface Person2 : Person
@end



#import "Person.h"

@interface Person ()

@end

@implementation Person
+ (void)load{
    NSLog(@"Person %s",__FUNCTION__);
}
+ (void) initialize{
    NSLog(@"Person %s",__FUNCTION__);
}
@end

@implementation Person2
+ (void)load{
    NSLog(@"Person2 %s",__FUNCTION__);
}
+ (void) initialize{
    
    NSLog(@"Person2 %s",__FUNCTION__);
}
@interface Person3 : Person2

@end
#import "Person3.h"

@implementation Person3
+ (void)load{
    NSLog(@"Person3 %s",__FUNCTION__);
}
+ (void) initialize{
    NSLog(@"Person3 %s",__FUNCTION__);
}
@end
```

然后查看一下输入：
```
Person +[Person load]
Person2 +[Person2 load]
 Car +[Car load]
Person3 +[Person3 load]
Person_add +[Person(add) load]
Person_add +[Person(add) initialize]
Person2 +[Person2 initialize]
Person3 +[Person3 initialize]

```

![build phases](https://upload-images.jianshu.io/upload_images/783986-fcf433295b7b1603.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

然后调换顺序：
![build phases](https://upload-images.jianshu.io/upload_images/783986-1d984042483a6901.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```
Car +[Car load]
Person +[Person load]
Person2 +[Person2 load]
Person3 +[Person3 load]
Person_add +[Person(add) load]
Person_add +[Person(add) initialize]
Person2 +[Person2 initialize]
Person3 +[Person3 initialize]
```


可以看出来`load`加载顺序是由父类到子类，由子类到类别，没有直接关系的是按照`Compile Sources`顺序加载的。
`initialize`是类别-》子类-》父类。

注释掉`Person3`中的`initialize`。再看一下效果：
```
#import "Person3.h"

@implementation Person3
//+ (void)load{
//    NSLog(@"Person3 %s",__FUNCTION__);
//}
//+ (void) initialize{
//    NSLog(@"Person3 %s",__FUNCTION__);
//}
@end
```
![person2](https://upload-images.jianshu.io/upload_images/783986-4820af62dbc4e05b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
结果是`Person2`重复调用了`initialize`,但是`+load`没有重复调用，则在`initialize`中添加下边代码:
```
+ (void) initialize{
    if (self == [Person2 self]) {
        NSLog(@"Person2 %s",__FUNCTION__);
    }
}
```
结果是：
```
Car +[Car load]
Person +[Person load]
Person2 +[Person2 load]
Person_add +[Person(add) load]
Person_add +[Person(add) initialize]
Person2 +[Person2 initialize]
```
多个类别的时候是这样子的：
![类别](https://upload-images.jianshu.io/upload_images/783986-23abbc4cd5fda361.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
```
Car +[Car load]
Person +[Person load]
Person2 +[Person2 load]
Person_add +[Person(add) load]
Person_add2 +[Person(add2) load]
Person_add2 +[Person(add2) initialize]
Person2 +[Person2 initialize]
```
我们在调换一下类别的顺序：
![类别](https://upload-images.jianshu.io/upload_images/783986-01773c86e0c6053d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```
Car +[Car load]
Person +[Person load]
Person2 +[Person2 load]
Person_add2 +[Person(add2) load]
Person_add +[Person(add) load]
Person_add +[Person(add) initialize]
Person2 +[Person2 initialize]
```

##### `Person2`的确是输出了单次，但是优于这是线程安全，多子类的情况下，性能差，某些业务需要单次执行的话，还是放在+load中比较好。多类别的时候，最下层的类别执行`initialize`，覆盖上边的`initialize`。
##### 由于`+load`是冷启动的时候调用的，`initialize`是在收到消息的时候调用的，大家还是根据业务需要作出选择吧。
答案：
##### 1.Load 和 Initialize 先加载哪个？
`load`先调用，`initialize`后调用
##### 2.父类和子类以及 Category 的关系？
`load`是调用顺序是：父类->子类->类别
`initialize`是：类别->子类->父类
`initialize`是子类没实现则调用父类的`initialize`
##### 3.如果是多个 Category 呢？

`load`在`Category`是根据`Compile Sources`的上下顺序调用。
`initialize`在`Category`是根据`Compile Sources`的最后调用，前边的覆盖。

当然大家在做性能优化的时候最好结合自己APP的性能瓶颈在哪里，然后再做细微调整。

思考：
##### 1.为什么Method Swizzling都放在load和initialize的优劣？
##### 2.`Category`能覆盖类的方法吗？


参考文章:
[官方文档](https://developer.apple.com/documentation/foundation/nsbundle/1415927-load?language=occ)


