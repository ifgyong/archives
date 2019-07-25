最近遇到一个问题，手机APP升级了但是想降级到某个版本，网上搜了半天擦发现可以自己通过iTunes12.7.0以前的版本下载旧版本APP的。自己总结一下发出啦，希望可以帮到一些人。
首选需要的软件：
- [下载iTunes12.6.3]( https://pan.baidu.com/s/1xk3qvCPyY-vuCRxDihjMrg) 提取码：ncds 
-  [下载fiddler](https://www.telerik.com/fiddler)
安装好之后，配置一下fiddler，看图：
![1](https://upload-images.jianshu.io/upload_images/783986-1286559d204ecd11.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![2](https://upload-images.jianshu.io/upload_images/783986-e499cdc6d763af53.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


![3](https://upload-images.jianshu.io/upload_images/783986-627e7b3ecc86dd22.JPG?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![4](https://upload-images.jianshu.io/upload_images/783986-8d153cfb1dd501b3.JPG?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![5](https://upload-images.jianshu.io/upload_images/783986-e1342492d2240355.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![6](https://upload-images.jianshu.io/upload_images/783986-a287899cf4596b0a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
双击安装即可。安装了证书就可以抓到HTTPS的请求了，HTTPS端口是443
那么现在打开iTunes试试能抓到借口吗？
![7](https://upload-images.jianshu.io/upload_images/783986-f1cb80b149b37438.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



![8](https://upload-images.jianshu.io/upload_images/783986-1318bc2aab462b8b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
### [查询历史版本APP版本 id](https://tools.lancely.tech/apple/app-search)


![9](https://upload-images.jianshu.io/upload_images/783986-1ce2b81235a35fd6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![10](https://upload-images.jianshu.io/upload_images/783986-40d5788b41184672.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![11](https://upload-images.jianshu.io/upload_images/783986-ac6f6b18818c16ed.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![12](https://upload-images.jianshu.io/upload_images/783986-b456c7e316267a37.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![13](https://upload-images.jianshu.io/upload_images/783986-8b7ad9a06d32cb86.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)




在目录`C:\Users\登陆用户名字\Music\iTunes\iTunes Media\Mobile Applications`可以查看下载的APP了。
![13](https://upload-images.jianshu.io/upload_images/783986-318f142594f13a25.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

下载iTunes可以使用第三方工具安装到手机上了，例如爱思助手等等。

可以下载旧版本的微信，qq都是可以的。

#### 本教程仅供学习交流，如作他用所承受的法律责任一概与作者无关
