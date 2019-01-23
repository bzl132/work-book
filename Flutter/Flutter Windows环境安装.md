Flutter Windows环境安装

### 配置环境变量

```
export PUB_HOSTED_URL=https://pub.flutter-io.cn
export FLUTTER_STORAGE_BASE_URL=https://storage.flutter-io.cn
```

下载Flutter SDK

https://flutter.io/docs/development/tools/sdk/archive#windows

解压后,配置环境变量

```
解压到  D:\Program Files\Flutter\flutter_windows_v1.0.0-stable
环境变量 D:\Program Files\Flutter\flutter_windows_v1.0.0-stable\bin
```

在控制台输入flutter doctor

```
Doctor summary (to see all details, run flutter doctor -v):
[âˆš] Flutter (Channel stable, v1.0.0, on Microsoft Windows [Version 6.3.9600],
    locale zh-CN)
[!] Android toolchain - develop for Android devices (Android SDK 28.0.3)
    ! Some Android licenses not accepted.  To resolve this, run: flutter doctor
      --android-licenses
[âˆš] Android Studio (version 3.2)
    X Flutter plugin not installed; this adds Flutter specific functionality.
    X Dart plugin not installed; this adds Dart specific functionality.
[!] IntelliJ IDEA Community Edition (version 2016.3)
    X Flutter plugin not installed; this adds Flutter specific functionality.
    X Dart plugin not installed; this adds Dart specific functionality.
    X This install is older than the minimum recommended version of 2017.1.0.
[!] IntelliJ IDEA Ultimate Edition (version 2017.2)
    X Flutter plugin not installed; this adds Flutter specific functionality.
    X Dart plugin not installed; this adds Dart specific functionality.
[!] VS Code, 64-bit edition (version 1.22.2)
[!] Connected device
    ! No devices available

```

解决问题

```
[!] Android toolchain - develop for Android devices (Android SDK 28.0.3)
    ! Some Android licenses not accepted.  To resolve this, run: flutter doctor
      --android-licenses
```

```shell
C:\Windows\System32>flutter doctor --android-licenses
A newer version of the Android SDK is required. To update, run:
D:\android_sdk\tools\bin\sdkmanager --update

D:\android_sdk\tools\bin\sdkmanager --update
Warning: An error occurred during installation: Failed to move away or delete ex
isting target file: D:\android_sdk\tools
Move it away manually and try again..

重命名tools文件夹名称,再运行update命令
D:\android_sdk\tools-1\bin\sdkmanager --update

done

flutter doctor --android-licenses
一路y
```

安装模拟器

1.下载模拟器

http://www.genymotion.net/

按照教程安装  http://www.runoob.com/w3cnote/android-tutorial-genymotion-install.html