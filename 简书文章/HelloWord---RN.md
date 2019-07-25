##### 1.运行rn到iOS模拟器上
```
react-native run-ios --simulator
```

##### 2.错误：
```
ls: /Users/Jerry/Library/Caches/com.facebook.ReactNativeBuild/glog-0.3.5.tar.gz: No such file or directory
shasum: /Users/Jerry/Library/Caches/com.facebook.ReactNativeBuild/glog-0.3.5.tar.gz:
```
答案：是下载第三方库失败，修改
```
/Users/Jerry/Desktop/demoTests/HelloWorld/node_modules/react-native/scripts/ios-install-third-party.sh
```
文件中下载地方的代理，
```rm -f "$cachedir/$file"
        (cd "$cachedir"; curl -J -L -O "$url")
        fetched=yes
```
修改之后：
```
rm -f "$cachedir/$file"
        (cd "$cachedir"; curl -J -L -O -x 127.0.0.1:1087 "$url")
        fetched=yes
```
方可解决问题

##### 3. [“config.h” file not found react native iOS](https://stackoverflow.com/questions/48540944/config-h-file-not-found-react-native-ios)
这是下载的第三方没有编译需要执行命令
```
cd node_modules/react-native/third-party/glog-0.3.4/
./configure
```

##### 4.
```Connection to localhost port 8081 [tcp/sunproxyadmin] succeeded!
Port 8081 already in use, packager is either not running or not running correctly
Command PhaseScriptExecution failed with a nonzero exit code
```
设置 `build System  LegacyBuild System`
#### 5.8081占用

```
error:Connection to localhost port 8081 [tcp/sunproxyadmin] succeeded!
Port 8081 already in use, packager is either not running or not running correctly
Command /bin/sh failed with exit code 2
```

方案：
```
killall -9 node
yarn start &react-native run-ios
```

##### 6.识别不了CFBundleIdenID
```
react-native app doesn't work ":CFBundleIdentifier", Does Not Exist 
```
方案：
```
删除node_modules 和相关的build文件夹 
执行npm install
```

#### 7. 占用8081
```
 ERROR  Metro Bundler can't listen on port 8081
```
方案：
```
a.找到占用端口的应用

sudo lsof -i :8081//占用8081的进程
kill -9 40247 //杀死进程
react-native start //重新启动服务
```

##### 8 .root-state is uid 777 and doesn't match your euid 501
方案：
```
rm -rf /usr/local/var/run/watchman/root-state
```

![HelloWord](https://upload-images.jianshu.io/upload_images/783986-287c3ce7bd1c8363.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

至此 Helloword已经运行完成，不过文字没有显示出来，那我们把标签的代码贴出来。
```
import React, { Component } from 'react';
import {
  Platform,
  StyleSheet,
  Text,
  View
} from 'react-native';

const instructions = Platform.select({
  ios: '第一个程序\n',
  android: '第一个程序\n',
});

export default class App extends Component<{}> {
  render() {
    return (
      <View style={styles.container}>
      
        <Text style={styles.welcome}>
          Hello Word
        </Text>
       
        <Text style={styles.instructions}>
          {instructions}
        </Text>
      </View>
    );
  }
}
```




