catch捕捉，也是一个老话题了，我们看一下OC是怎么实现的。

使用NSAssert不符合条件的主动崩溃
```
- (void)check:(NSString *)name age:(NSInteger)age{
    NSAssert(name.length>0, @"name is nil");
    NSAssert(age>0 && age<120, @"age is outof rang 0...120");
}
```
还有另外一种被动报错，可以捕捉到catch中的
```
    @try {
        NSArray *list = @[];
        list[2];
    } @catch (NSException *exception) {
//这里报错才会执行
        NSLog(@"%@",exception.description);
    } @finally {
        //这里一定会执行
    }
```
实际输出是：但是不会崩溃。
```
 *** -[__NSArray0 objectAtIndex:]: index 2 beyond bounds for empty NSArray
```

那么我们在看一下Swift中声明Error类型的用法:
```
enum ErrorTest:Error {
    case nameVisiableError
    case ageError
    case heightError
    case nameLengthError
}


```
首先我们继承Error这个`protocol`，定义一下自己错误类型，后边可以根据自己的错误类型来判断是哪一种错误。
然后实现抛出异常函数`throws`:
```
    func checkPerson(p:Person) throws -> String {
        guard p.age>0 && p.age<120 else {
            throw ErrorTest.ageError
        }
        guard p.name.count<10 && p.name.count>0 else {
            throw ErrorTest.nameVisiableError
        }
       return "success"
    }
```
我们这边是使用`throw ErrorTest.nameVisiableError`来抛出不符合条件的异常情况，
然后捕捉函数：
```
 let p2 = Person(age: 1000, name: "小明")

do {
            try str = checkPerson(p: p2)
//不报错 下边会输出，报错则不执行
print(str)
        } catch  {
//报错则执行相对应的错误类型
            switch error {
            case ErrorTest.ageError:print("ageError"); break
            case ErrorTest.nameLengthError:print("nameLengthError"); break
            case ErrorTest.nameVisiableError:print("nameLengthError"); break
            case ErrorTest.heightError:print("heightError"); break
            default:break
            }
        }
```
定义一个年龄是1000的小明，年龄超出整除范围，实际输出是：
```
ageError
```
当然我们也可以在catch中做好错误记录或者弹框提示超出范围提示用户。

> try try?和try！

这三个的区别：
try：会执行函数之后抛出函数
try?是选择类型的执行，当报错的时候，返回nil，不报错的时候返回正常的值
try!是强制解包，当抛出异常的时候也解包，导致崩溃问题。

再看另外一种Swift的实现方式：
```
    func checkPersonAssert(p:Person) -> Void {
        assert(p.age>0 || p.age<120, "age is our of rang\n")
//报错则崩溃，并输出age is our of rang 。下边的代码不会执行
        assert(p.name.count>0, "name is out of rang\n")
//报错则崩溃，并输出 name is out of rang
    }
```
这种和OC其实一样，不符合条件的直接崩溃并输出信息。
这种经常用于封装函数的必要条件的地方使用。

报错和捕捉先说这么多，后边有发现更好的在补充。
