### 打开app设置界面
```
NSURL *url = [NSURL URLWithString:UIApplicationOpenSettingsURLString];
            [[UIApplication sharedApplication]openURL:url];
```
这个是所有有关APP的设置都在这个页面，蓝牙，网络，获取地理位置等等。
