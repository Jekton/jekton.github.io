---
title: Flutter 开发（5）- 插件的使用、开发和发布
date: 2018-09-16 09:34:02
categories: Flutter
tags: Flutter
description: 本篇文章我们先一起学习 Flutter 插件的使用，然后通过开发一个 toast 插件来学习它的开发，最后发布到 Pub 上。
---

> 本文由[玉刚说写作平台](http://renyugang.io/post/75)提供写作赞助
> 赞助金额：200元
> 原作者：`水晶虾饺`
> 版权声明：本文版权归微信公众号`玉刚说`所有，未经许可，不得以任何形式转载

本篇文章我们先一起学习 Flutter 插件的使用，然后通过开发一个 toast 插件来学习它的开发，最后发布到 Pub 上。

# 插件的使用

Flutter 的库是以 package 的方式来管理。Package 分为两种，Dart package（也叫 library package） 和 plugin package。当我们说 Fluter 包的时候，指的其实也是 Dart 包，它只能使用 Dart 和 Flutter 提供的 API；而当我们说 Flutter 插件时指的是后者，也就是 plugin package。Flutter 插件通常会包含平台特定的代码。对包的使用者来说，两者没有区别。

## 添加依赖

为了使用一个库，我们首先在 pubspec.yaml 里声明一个依赖：
```yaml
dependencies:
  shared_preferences: ^0.4.2
```
`^0.4.2` 表示与 `0.4.2` 兼容的版本。我们也可以指定依赖库的为特定的版本：
- any：任意版本
- 1.2.3：特定的版本
- <1.2.3：小于 1.2.3 的版本，此外还有 <=、>、>= 可以使用
- '>=1.2.3 <2.0.0'：指定一个范围

接下来，在项目的根目录执行 `flutter packages get`。如果你使用 Android Studio 进行开发，也可以直接在 pubspec.yaml 的编辑页面上面点击 Packages get 按钮。

上面例子的是发布在 `https://pub.dartlang.org/` 上的库，除此之外，我们也可以使用其他的源：
```yaml
dependencies:
  transmogrify:
    hosted:
      name: transmogrify
      url: http://your-package-server.com
    version: ^1.4.0

  kittens:
    git:
      url: git://github.com/munificent/cats.git
      ref: some-branch  # 可选的

  kittens:
    git:
      url: git://github.com/munificent/cats.git
      path: path/to/kittens  # 指定路径

    # 甚至可以指定一个本地路径
    transmogrify:
      path: /Users/me/transmogrify
```

如果你看过 Flutter 的 pubspec，应该会注意到 flutter 是这样声明的：
```yaml
dependencies:
  flutter:
    sdk: flutter
```
sdk 用于导入随 Flutter 一起发布的包，目前只有 flutter。


## 使用

导入相关的包后，我们就可以使用它的 API 了：

```dart
import 'package:shared_preferences/shared_preferences.dart';

void foo() async {
  var prefs = await SharedPreferences.getInstance();
  var used = prefs.getBool('used');
  if (!used) {
    prefs.setBool('used', true);
  }
}
```

这种导入方式的问题在于，他把库里所有的符号到导入到了全局的命名空间里面（比方说，在上面的例子里，我们可以直接使用 `SharedPreferences`）。有时为了防止命名空间的污染，我们可以使用 `as` 给导入的库一个名字（当然，对 SharedPreferences 其实没有必要使用限定名就是了）：
```dart
void foo() async {
  var prefs = await sp.SharedPreferences.getInstance();
  var used = prefs.getBool('used');
  if (!used) {
    prefs.setBool('used', true);
  }
}
```

了解了 Flutter 包的使用后，下面我们自己来开发一个 flutter 插件。

# 开发一个插件

学习 Flutter 的过程中，不知道是你是否注意到 Flutter 并没有提供一个 Toast API。为了弥补这个遗憾，在这一节里我们就来开发一个插件，让它支持 Toast。

在开始开发前，我们先来了解一下 Flutter 如何跟平台相关的代码进行通信。

## MethodChannel

Flutter 跟平台相关代码可以通过 `MethodChannel` 进行通信。客户端通过 `MethodChannel` 将方法调用和参数发生给服务端，服务端也通过 `MethodChannel` 接收相关的数据。

![PlatformChannels](PlatformChannels.png)

需要注意的是，上图中的箭头是双向的。也就是说，我们不仅可以从 Flutter 调用 Android/iOS 的代码，也可以从 Android/iOS 调用 Flutter。调用时相关的参数对应如下：

| Dart | Android | iOS |
| ---- | ------- | --- |
| null | null | nil (NSNull when nested) |
| bool | java.lang.Boolean | NSNumber numberWithBool: |
| int | java.lang.Integer | NSNumber numberWithInt: |
| int, if 32 bits not enough | java.lang.Long | NSNumber numberWithLong: |
| double | java.lang.Double | NSNumber numberWithDouble: |
| String | java.lang.String | NSString |
| Uint8List | byte[] | FlutterStandardTypedData typedDataWithBytes: |
| Int32List | int[] | FlutterStandardTypedData typedDataWithInt32: |
| Int64List | long[] | FlutterStandardTypedData typedDataWithInt64: |
| Float64List | double[] | FlutterStandardTypedData typedDataWithFloat64: |
| List | java.util.ArrayList | NSArray |
| Map | java.util.HashMap | NSDictionary |


## 创建项目

这里假设读者使用 Android Studio 开发。
1. 在菜单上选择 File -> New -> New Flutter Project
2. 在弹出的面板里选择 Flutter Plugin，点击 next
3. Project name 我们填入 flutter_toast2018，其他信息读者根据自身需要填写

之所以叫 flutter_toast2018 是因为 Pub 上已经有一个 flutter_toast，所以加上 2018 防止名字冲突。

生成的项目有 4 个主要的目录：
- android：插件本地代码的 Android 端实现
- ios：iOS 端的实现
- lib：Dart 代码。插件的客户将会使用这里实现的接口
- example：插件的使用示例


## 插件开发

### Android 端代码实现

其实在上一步我们生成项目的时候，项目里就已经包含了一个实现了 `platformVersion` 的 Flutter 插件 demo，有兴趣的读者可以看看学习一下。下面，我们来开发自己的 Toast 插件（注意，我们的实现只支持 Android）。

首先我们来了解一下接口 `MethodCallHandler`：
```java
public interface MethodCallHandler {
  void onMethodCall(MethodCall call, Result result);
}
```

这个接口用于处理 Flutter 的本地方法调用请求。也就是说，我们需要实现这个接口，当 Flutter 调用我们的时候，弹出一个 toast。

实现这个接口的是 `FlutterToast2018Plugin`（位于 android 目录下）：
```java
public class FlutterToast2018Plugin implements MethodCallHandler {
  public static void registerWith(Registrar registrar) {
    // "example.com/flutter_toast2018" 是我们 method channel 的名字，Dart 代码里还需要用到它。
    // 为了防止命名冲突，可以在它的前面加上域名
    final channel = new MethodChannel(registrar.messenger(), "example.com/flutter_toast2018");
    channel.setMethodCallHandler(new FlutterToast2018Plugin());
  }

  @Override
  public void onMethodCall(MethodCall call, Result result) {
    // TODO
  }
}
```

为了弹出 Toast，我们给 `FlutterToast2018Plugin` 的构造函数添加一个 `Context` 参数：
```java
public class FlutterToast2018Plugin implements MethodCallHandler {
  private final Context mContext;

  public FlutterToast2018Plugin(Context context) {
    mContext = context;
  }

  // 注册 MethodCallHandler
  public static void registerWith(Registrar registrar) {
    final channel = new MethodChannel(registrar.messenger(), "example.com/flutter_toast2018");
    // context 可以从 Registrar 拿到
    channel.setMethodCallHandler(new FlutterToast2018Plugin(registrar.context());
  }

  // ...
}
```

现在，实现 `onMethodCall` 方法：
```java
public class FlutterToast2018Plugin implements MethodCallHandler {
  // ...

  @Override
  public void onMethodCall(MethodCall call, Result result) {
    // call.method 是方法名，这里我们就叫它 toast
    if (call.method.equals("toast")) {
      // 调用本地代码的时候，只能传递一个参数。为了传递多个，可以把参数放在一个 map 里面。
      // call.arguemnt() 方法支持 Map 和 JSONObject
      String content = call.argument("content");
      String duration = call.argument("duration");
      Toast.makeText(mContext, content,
                     "short".equals(duration) ? Toast.LENGTH_SHORT : Toast.LENGTH_LONG)
              .show();
      // 执行成功
      result.success(true);
    } else {
      result.notImplemented();
    }
  }
}
```

### Flutter 端

Flutter 端需要做的，就是生成一个 `MethodChannel`，然后通过这个 `MethodChannel` 调用 `toast` 方法：
```dart
import 'dart:async';
import 'package:flutter/services.dart';

enum ToastDuration {
  short, long
}

class FlutterToast {
  // 这里的名字要跟 Java 端的对应
  static const MethodChannel _channel =
      const MethodChannel('example.com/flutter_toast2018');

  static Future<bool> toast(String msg, ToastDuration duration) async {
    var argument = {
      'content': msg,
      'duration': duration.toString()
    };
    // 本地方法是一个异步调用。'toast' 对应我们在前面 Java 代码的 onMethodCall
    // 方法里面处理的方法名
    var success = await _channel.invokeMethod('toast', argument);
    return success;
  }
}
```

### 使用插件

在这一节我们修改工程里 example 目录下的示例，用它来演示插件的使用：
```dart
import 'package:flutter/material.dart';

// 首先导入我们的包
import 'package:flutter_toast2018/flutter_toast2018.dart';

void main() => runApp(new MyApp());

class MyApp extends StatelessWidget {

  @override
  Widget build(BuildContext context) {
    return new MaterialApp(
      home: new Scaffold(
        appBar: new AppBar(
          title: const Text('Plugin example app'),
        ),
        body: new Center(
          child: RaisedButton(
            child: Text('toast'),
            // 插件的使用跟其他库没有什么区别，直接调用即可
            onPressed: () => FlutterToast2018.toast(
              'Toast from Flutter', ToastDuration.short
            ),
          ),
        ),
      ),
    );
  }
}
```


# 发布插件

前面我们说过，pubspec 支持通过本地路径和 Git 导入依赖，但为了更好的管理版本依赖，还是推荐发布插件到 [https://pub.dartlang.org/](https://pub.dartlang.org/)。在这一节，我们就把前面开发的 toast 插件发布到 Pub 上。

需要注意的是，由于某些众所周知的原因，pub.dartlang.org 需要一把梯子才能上去。虽然我们也可以通过 flutter-io.cn 来发布，但上传的时候需要登录 Google 账号，梯子还是少不了的。


## 检查配置

首先是 pubspec.yaml。对 Flutter 插件来说，pubspec 里除了插件的依赖，还包含一些元信息，读者可以根据需要，把这些补上：
```yaml
name: flutter_toast2018
description: A new Flutter plugin for Android Toast.
version: 0.0.1
author: Jekton <ljtong64@gmail.com>
homepage: https://jekton.github.io/
```

另外，发布到 Pub 上的包需要包含一个 LICENSE，关于 LICENSE 文件，最简单的方法就是在 GitHub 创建仓库的时候选中一个。

## 检查插件

现在，我们在工程的根目录执行以下命令，检测一下插件有没有什么问题：
```shell
flutter packages pub publish --dry-run
```

如果一切正常，将会输出：
```
...

Package has 0 warnings.
```

## 发布插件

发布插件和上一步一样，只是少了 --dry-run 参数：
```shell
flutter packages pub publish
```

如果是第一次发布，会提示验证 Google 账号。授权后便可以继续上传，如果成功的话，会提示“Successful uploaded package”：
```
Looks great! Are you ready to upload your package (y/n)? y
Pub needs your authorization to upload packages on your behalf.
In a web browser, go to https://accounts.google.com/o/oauth2/auth?access_type=offline&approval_prompt=force&response_type=code&client_id=xxxxxxxxxxxxxx-8grd2eg9tj9f38os6f1urbcvsq399u8n.apps.googleusercontent.com&redirect_uri=http%3A%2F%2Flocalhost%3A52589&scope=https%3A%2F%2Fwww.googleapis.com%2Fauth%2Fuserinfo.email
Then click "Allow access".

Waiting for your authorization...
Successfully authorized.
Uploading...
Successful uploaded package.
```

前面我们发布的包可以在 [https://pub.dartlang.org/packages/flutter_toast2018](https://pub.dartlang.org/packages/flutter_toast2018) 或 [https://pub.flutter-io.cn/packages/flutter_toast2018](https://pub.flutter-io.cn/packages/flutter_toast2018) 找到。

相关的代码则是放到了 GitHub 上：
```shell
git clone https://github.com/Jekton/flutter_toast2018.git
```

最后再提一提我们没有讲到的 Flutter package。为了开发一个 Flutter 包，我们在创建项目的时候可以选择 Flutter package。它和 Flutter 插件唯一的区别是Flutter package 不能包含平台特定的代码（只能使用 Dart 和 Flutter API）。除此之外，开发、发布和使用都跟 Flutter 插件没有什么区别。
