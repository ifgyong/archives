命令：
`clang -x objective-c -rewrite-objc -isysroot /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator12.1.sdk xxxx.m`
执行这个命令需要修改编译设置：
![image.png](https://upload-images.jianshu.io/upload_images/783986-5627569e5d3b5d57.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
xcode10.1 默认 `c++ language Dialect`  是`c++14`,修改为``c++11``
c++ Standard Library 是`libc++(LLVM C++ standard library with c++ 11 suport)`,修改为`libstdc++(gun c++ standard library)`。

在执行上边的命令，就会出现同名的cpp 文件了。


