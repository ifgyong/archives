最近这几年混合开发比例慢慢高升，似乎要占据了客户端的主导地位，那么现在新出的flutter究竟是何方神圣？现在我们慢慢实战起来。
#### 安装环境
我的是mac开发环境，安装起来比较轻松，大概10分钟吧，我这里讲一下步骤和注意的问题
- 打开获取FlutterSDK，下载[SDK](https://storage.googleapis.com/flutter_infra/releases/stable/macos/flutter_macos_v1.2.1-stable.zip)
- 解压文件` cd ~/development`
` unzip ~/Downloads/flutter_macos_v1.2.1-stable.zip`
- 添加路径到bash_Profile,
打开bash_profile
```
vi ~/.bash_profile
```
添加一行
```
 export PATH="$PATH:[PATH_TO_FLUTTER_GIT_DIRECTORY]/flutter/bin"

```
`PATH_TO_FLUTTER_GIT_DIRECTORY`是刚才解压的文件目录。
刷新添加的信息
```
source $HOME/.bash_profile
```

验证查看
```
echo $PATH
```
我这边是直接成功了。
![PATH](https://upload-images.jianshu.io/upload_images/783986-108516196012508b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

然后在测试下命令能运行不
```
Yong:flutter Yong$ flutter
Manage your Flutter app development.

Common commands:

  flutter create <output directory>
    Create a new Flutter project in the specified directory.

  flutter run [options]
    Run your Flutter application on an attached device or in an emulator.

Usage: flutter <command> [arguments]

Global options:
-h, --help                  Print this usage information.
-v, --verbose               Noisy logging, including all shell commands executed.
                            If used with --help, shows hidden options.

-d, --device-id             Target device id or name (prefixes allowed).
    --version               Reports the version of this tool.
    --suppress-analytics    Suppress analytics reporting when this command runs.
    --bug-report            Captures a bug report file to submit to the Flutter team.
                            Contains local paths, device identifiers, and log snippets.

    --packages              Path to your ".packages" file.
                            (required, since the current directory does not contain a ".packages" file)

Available commands:
  analyze                  Analyze the project's Dart code.
  attach                   Attach to a running application.
  bash-completion          Output command line shell completion setup scripts.
  build                    Flutter build commands.
  channel                  List or switch flutter channels.
  clean                    Delete the build/ and .dart_tool/ directories.
  config                   Configure Flutter settings.
  create                   Create a new Flutter project.
  devices                  List all connected devices.
  doctor                   Show information about the installed tooling.
  drive                    Runs Flutter Driver tests for the current project.
  emulators                List, launch and create emulators.
  format                   Format one or more dart files.
  help                     Display help information for flutter.
  install                  Install a Flutter app on an attached device.
  logs                     Show log output for running Flutter apps.
  make-host-app-editable   Moves host apps from generated directories to non-generated directories so that they can
                           be edited by developers.
  packages                 Commands for managing Flutter packages.
  precache                 Populates the Flutter tool's cache of binary artifacts.
  run                      Run your Flutter app on an attached device.
  screenshot               Take a screenshot from a connected device.
  stop                     Stop your Flutter app on an attached device.
  test                     Run Flutter unit tests for the current project.
  trace                    Start and stop tracing for a running Flutter app.
  upgrade                  Upgrade your copy of Flutter.
  version                  List or switch flutter versions.
```
这样子所有的命令都出来了，我们先创建一个新的pro试下。

创建一个叫做flutter_01的工程
```
flutter create flutter_01
```

然后run一下

```
flutter run
```
竟然报错了
![error](https://upload-images.jianshu.io/upload_images/783986-021983f40719c494.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
看了一下 ，竟然是模拟器没有跑起来的原因，直接Xcode跑起来一个模拟器，然后再重新`flutter run`。
![flutter run](https://upload-images.jianshu.io/upload_images/783986-72d424b42edd5f46.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


ok已经好了，看一下什么样子。
![helloword](https://upload-images.jianshu.io/upload_images/783986-a160027dfd41a936.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
好了，HelloWord已经成功了。今天先到这里，后边在写一个具体功能的Demo，喜欢的可以Start哦。
[代码在这里](https://github.com/ifgyong/demo)



参考文章：
- [flutter](https://flutter.dev/)
