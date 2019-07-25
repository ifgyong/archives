#### 记录我做直播的脚印
### 1.下载[ijkplayer](https://github.com/Bilibili/ijkplayer)到本地
### 2.用命令`cd ijkplayer`打开目录
 *`//查看目录下边的文件`* 
```

$ ls 
COPYING.GPLv2  		compile-android-j4a.sh 	init-android-openssl.sh
COPYING.GPLv3  		config 			init-android-prof.sh
COPYING.LGPLv2.1       	doc    			init-android.sh
COPYING.LGPLv2.1.txt   	extra  			init-config.sh
COPYING.LGPLv3 		ijkmedia       		init-ios-openssl.sh
MODULE_LICENSE_APACHE2 	ijkprof			init-ios.sh
NEWS.md			init-android-exo.sh    	ios
NOTICE 			init-android-j4a.sh    	tools
README.md      		init-android-libsoxr.sh	version.sh
android			init-android-libyuv.sh
```
### 3.下载ffmpeg 
`执行命令 ./init-ios.sh `

```

 
$ ./init-ios.sh   
git version 2.8.4 (Apple Git-73)
== pull gas-preprocessor base ==
Cloning into 'extra/gas-preprocessor'...
remote: Counting objects: 440, done.
remote: Total 440 (delta 0), reused 0 (delta 0), pack-reused 440
Receiving objects: 100% (440/440), 72.66 KiB | 46.00 KiB/s, done.
Resolving deltas: 100% (191/191), done.
Checking connectivity... done.
== pull ffmpeg base ==
Cloning into 'extra/ffmpeg'...

remote: Counting objects: 494448, done.
remote: Total 494448 (delta 0), reused 0 (delta 0), pack-reused 494448
Receiving objects: 100% (494448/494448), 179.12 MiB | 2.87 MiB/s, done.
Resolving deltas: 100% (382172/382172), done.
Checking connectivity... done.
== pull ffmpeg fork armv7 ==
Cloning into 'ios/ffmpeg-armv7'...
Checking connectivity... done.
Counting objects: 494448, done.
Delta compression using up to 8 threads.
Compressing objects: 100% (111157/111157), done.
Writing objects: 100% (494448/494448), done.
Total 494448 (delta 382172), reused 494448 (delta 382172)
Switched to a new branch 'ijkplayer'
/Users/Jerry/Downloads/ijkplayer-master
== pull ffmpeg fork arm64 ==
Cloning into 'ios/ffmpeg-arm64'...
Checking connectivity... done.
Counting objects: 494448, done.
Delta compression using up to 8 threads.
Compressing objects: 100% (111157/111157), done.
Writing objects: 100% (494448/494448), done.
Total 494448 (delta 382172), reused 494448 (delta 382172)
Switched to a new branch 'ijkplayer'
/Users/Jerry/Downloads/ijkplayer-master
== pull ffmpeg fork i386 ==
Cloning into 'ios/ffmpeg-i386'...
Checking connectivity... done.
Counting objects: 494448, done.
Delta compression using up to 8 threads.
Compressing objects: 100% (111157/111157), done.
Writing objects: 100% (494448/494448), done.
Total 494448 (delta 382172), reused 494448 (delta 382172)
Switched to a new branch 'ijkplayer'
/Users/Jerry/Downloads/ijkplayer-master
== pull ffmpeg fork x86_64 ==
Cloning into 'ios/ffmpeg-x86_64'...
Checking connectivity... done.
Counting objects: 494448, done.
Delta compression using up to 8 threads.
Compressing objects: 100% (111157/111157), done.
Writing objects: 100% (494448/494448), done.
Total 494448 (delta 382172), reused 494448 (delta 382172)
Switched to a new branch 'ijkplayer'
/Users/Jerry/Downloads/ijkplayer-master
```

### 4.编译ffmoeg
`cd ios 
./compile-ffmpeg.sh clean
./compile-ffmpeg.sh all`

```
//

$ ./compile-ffmpeg.sh clean
====================
[*] check xcode version
====================
FF_ALL_ARCHS = armv7 arm64 i386 x86_64
==================
clean ffmpeg-armv7
==================
/Users/Jerry/Downloads/ijkplayer-master/ios
clean ffmpeg-arm64
==================
/Users/Jerry/Downloads/ijkplayer-master/ios
clean ffmpeg-i386
==================
/Users/Jerry/Downloads/ijkplayer-master/ios
clean ffmpeg-x86_64
==================
/Users/Jerry/Downloads/ijkplayer-master/ios
clean build cache
=================
clean success

//编译各个版本成静态库
./compile-ffmpeg.sh all

因为信息太多了 我就没复制。

```
添加静态库：
![屏幕快照 2016-12-23 下午1.28.08.png](http://upload-images.jianshu.io/upload_images/783986-96c971a3430913be.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 5.现在ffmpeg已经编译成功
 可以在你的工程导入` 
IJKMediaFramework`可以运行ijkplayer的demo 了。
![屏幕快照 2016-12-23 下午1.31.22.png](http://upload-images.jianshu.io/upload_images/783986-f7cc09b77fc299da.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 6.制作`IJKMediaFramework`静态库。
##### 1.按照图示的路径打开项目

![项目路径](http://upload-images.jianshu.io/upload_images/783986-7c636fae8c4476e7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
##### 2.设置release

![选择target](http://upload-images.jianshu.io/upload_images/783986-f6615c481d83d118.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![设置release](http://upload-images.jianshu.io/upload_images/783986-45fa145aee6a31d5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### 3.对模拟器和真机分别运行
选择文件夹右键->show in finder

![找到prodects右键-》show in finder](http://upload-images.jianshu.io/upload_images/783986-af2ee4d024c3c779.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

将文件夹Release-iphones 和Release-iphonesimulator中的文件合并。
真机的静态库：
![真机的静态库](http://upload-images.jianshu.io/upload_images/783986-b7103f77406b43b3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
模拟器的静态库：

![模拟器 的静态库](http://upload-images.jianshu.io/upload_images/783986-f6037fc63095a00a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

合并的命令：
`lipo -create "真机版本路径" "模拟器版本路径" -output "合并后的文件路径"`
我自己的命令：
```
lipo -create /Users/Jerry/Library/Developer/Xcode/DerivedData/IJKMediaDemo-ejjmowghejcpthhaznyhnoshrphz/Build/Products/Release-iphonesimulator/IJKMediaFramework.framework/IJKMediaFramework  /Users/Jerry/Library/Developer/Xcode/DerivedData/IJKMediaDemo-ejjmowghejcpthhaznyhnoshrphz/Build/Products/Release-iphoneos/IJKMediaFramework.framework/IJKMediaFramework -output /Users/Jerry/Desktop/IJKMediaFramework
```

然后将合并成功的静态库替换刚才Release-iphoneos->IJKMediaFramework-->IJKMediaFramework。
这个framework就可以使用了。
我是已经合并成功的。可以直接下载使用[包含真机和模拟器]
[下载地址](https://pan.baidu.com/s/1bpiliQN)
文件较大，所以暂时用了百度网盘。

### 7.测试是否正常使用。
*测试的时候新建的工程要添加好了那些lib，然后导入刚才合成的framework*
![测试可以正常使用](http://upload-images.jianshu.io/upload_images/783986-74c56ab4e660b442.PNG?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
大家可以下载[demo](git@github.com:ifgyong/ijkplayerDemo.git)运行一下
