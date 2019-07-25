遇到的坑之一：
``[access] This app has crashed because it attempted to access privacy-sensitive data without a usage description. The app's Info.plist must contain an NSPhotoLibraryUsageDescription key with a string value explaining to the user how the app uses this data.``

解决办法：设置``info.plist``文件权限设置就行了。
1，在项目中找到``info.plist``文件，右击有个 Open As，以Source Code 的形式打开
2，分别复制 以下 Value 和Key，Key 一定不能错，Value 貌似可以随便填写

``相机权限描述：<key>NSCameraUsageDescription</key><string>app想要访问您设备的相机</string>
通信录：    <key>NSContactsUsageDescription</key>    <string>app想要访问您设备的通讯录</string>
麦克风：    <key>NSMicrophoneUsageDescription</key>    <string>app想要访问您设备的麦克风</string>
相机：    <key>NSPhotoLibraryUsageDescription</key>    <string>app想要访问您设备的照片库</string>``


打包遇见的坑：
添加了iOS10推送的适配，必须用xcode8打包，但是打包上传到appStore上面的包显示是构建无效。
首先把用到的隐私字段全部加进去，就像前面所说的相机，照片，蓝牙，等等。``全部添加``。
然后在看看第三方有没有问题，没有问题的话估计打包要成功了。
第二：
又踩到一个坑是
用到的第三方库更新的时候要用pod更新到1.0.0版本才行。当我codeing完成之后，发现提交到仓库push操作失败，报错：`` fatal: Authentication failed for 'https://github.com/CocoaPods/Specs.git/'``在请问了Google之后发现，``git remote -v`` 发现这个仓库的url是pod的url，所以要将自己的仓库url改成自己的url。具体代码：
``
  git remote -v //查看自己push fetch 的url
git remote set-url origin XXXXX //XXX是你们仓库的地址。
``
后续遇到的还会添加。
