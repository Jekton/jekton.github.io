---
title: Flutter 开发（4）- 文件、存储和网络
date: 2018-09-01 07:29:39
categories: Flutter
tags: Flutter
description: 我们将在 flutter-ux-basic 一文的基础上，继续开发一个 echo 客户端。由于日常开发中 HTTP 比 socket 更常见，我们的 echo 客户端将会使用 HTTP 协议跟服务端通信。Echo 服务器也会使用 Dart 来实现。
---

> 本文由`玉刚说写作平`台提供写作赞助
> 赞助金额：200元
> 原作者：`水晶虾饺`
> 版权声明：本文版权归微信公众号`玉刚说`所有，未经许可，不得以任何形式转载

我们将在[Flutter 开发（3）- 交互、动画、手势和事件处理](2018/08/29/flutter-ux-basic)的基础上，继续开发一个 echo 客户端。由于日常开发中 HTTP 比 socket 更常见，我们的 echo 客户端将会使用 HTTP 协议跟服务端通信。Echo 服务器也会使用 Dart 来实现。


# HTTP 服务端

在开始之前，你可以在 GitHub 上找到上篇文章的代码，我们将在它的基础上进行开发。
```shell
git clone https://github.com/Jekton/flutter_demo.git
cd flutter_demo
git checkout ux-basic
```


## 服务端架构

首先我们来看看服务端的架构（说是架构，但其实非常的简单，或者说很简陋）：
```dart
import 'dart:async';
import 'dart:io';

class HttpEchoServer {

  final int port;
  HttpServer httpServer;
  // 在 Dart 里面，函数也是 first class object，所以我们可以直接把
  // 函数放到 Map 里面
  Map<String, void Function(HttpRequest)> routes;

  HttpEchoServer(this.port) {
    _initRoutes();
  }

  void _initRoutes() {
    routes = {
      // 我们只支持 path 为 '/history' 和 '/echo' 的请求。
      // history 用于获取历史记录；
      // echo 则提供 echo 服务。
      '/history': _history,
      '/echo': _echo,
    };
  }

  // 返回一个 Future，这样客户端就能够在 start 完成后做一些事
  Future start() async {
    // 1. 创建一个 HttpServer
    httpServer = await HttpServer.bind(InternetAddress.loopbackIPv4, port);
    // 2. 开始监听客户请求
    return httpServer.listen((request) {
      final path = request.uri.path;
      final handler = routes[path];
      if (handler != null) {
        handler(request);
      } else {
        // 给客户返回一个 404
        request.response.statusCode = HttpStatus.notFound;
        request.response.close();
      }
    });
  }

  void _history(HttpRequest request) {
    // ...
  }

  void _echo(HttpRequest request) async {
    // ...
  }

  void close() async {
    var server = httpServer;
    httpServer = null;
    await server?.close();
  }
}
```

在服务端框架里，我们把支持的所有路径都加到 routes 里面，当收到客户请求的时候，只需要直接从 routes 里取出对应的处理函数，把请求分发给他就可以了。


## 将对象序列化为 JSON

为了把 Message 对象序列化为 JSON，这里我们对 Message 做一些小修改：
```dart
class Message {
  final String msg;
  final int timestamp;


  Message(this.msg, this.timestamp);

  Message.create(String msg)
      : msg = msg, timestamp = DateTime.now().millisecondsSinceEpoch;

  Map<String, dynamic> toJson() => {
    "msg": "$msg",
    "timestamp": timestamp
  };

  @override
  String toString() {
    return 'Message{msg: $msg, timestamp: $timestamp}';
  }
}
```
这里我们加入一个 toJson 方法。下面是服务端的 _echo 方法：
```dart
class HttpEchoServer {
  static const GET = 'GET';
  static const POST = 'POST';

  const List<Message> messages = [];

  // ...

  _unsupportedMethod(HttpRequest request) {
    request.response.statusCode = HttpStatus.methodNotAllowed;
    request.response.close();
  }

  void _echo(HttpRequest request) async {
    if (request.method != POST) {
      _unsupportedMethod(request);
      return;
    }

    // 获取从客户端 POST 请求的 body，更多的知识，参考
    // https://www.dartlang.org/tutorials/dart-vm/httpserver
    String body = await request.transform(utf8.decoder).join();
    if (body != null) {
      var message = Message.create(body);
      messages.add(message);
      request.response.statusCode = HttpStatus.ok;
      // json 是 convert 包里的对象，encode 方法还有第二个参数 toEncodable。当遇到对象不是
      // Dart 的内置对象时，如果提供这个参数，就会调用它对对象进行序列化；这里我们没有提供，
      // 所以 encode 方法会调用对象的 toJson 方法，这个方法在前面我们已经定义了
      var data = json.encode(message);
      // 把响应写回给客户端
      request.response.write(data);
    } else {
      request.response.statusCode = HttpStatus.badRequest;
    }
    request.response.close();
  }
}
```
如果读者对服务端编程没有太大兴趣或不太了解，这里重点关注如果把对象序列化为 JSON 就可以了。下面是客户端部分。


# HTTP 客户端

我们的 echo 服务器使用了 dart:io 包里面 HttpServer 来开发。对应的，我们也可以使用这个包里的 HttpRequest 来执行 HTTP 请求，但这里我们并不打算这么做。第三方库 http 提供了更简单易用的接口。

首先，别忘了把依赖添加到 pubspec 里：
```yaml
# pubspec.yaml
dependencies:
  # ...

  http: ^0.11.3+17
```

客户端实现如下：
```dart
import 'package:http/http.dart' as http;

class HttpEchoClient {
  final int port;
  final String host;

  HttpEchoClient(this.port): host = 'http://localhost:$port';

  Future<Message> send(String msg) async {
    // http.post 用来执行一个 HTTP POST 请求。
    // 它的 body 参数是一个 dynamic，可以支持不同类型的 body，这里我们
    // 只是直接把客户输入的消息发给服务端就可以了。由于 msg 是一个 String，
    // post 方法会自动设置 HTTP 的 Content-Type 为 text/plain
    final response = await http.post(host + '/echo', body: msg);
    if (response.statusCode == 200) {
      Map<String, dynamic> msgJson = json.decode(response.body);
      // Dart 并不知道我们的 Message 长什么样，我们需要自己通过
      // Map<String, dynamic> 来构造对象
      var message = Message.fromJson(msgJson);
      return message;
    } else {
      return null;
    }
  }
}

class Message {
  final String msg;
  final int timestamp;

  Message.fromJson(Map<String, dynamic> json)
    : msg = json['msg'], timestamp = json['timestamp'];

  // ...
}
```

现在，让我们把他们和上一节的 UI 结合到一起。首先启动服务器，然后创建客户端：
```dart
HttpEchoServer _server;
HttpEchoClient _client;

class _MessageListState extends State<MessageList> {
  final List<Message> messages = [];

  @override
  void initState() {
    super.initState();

    const port = 6060;
    _server = HttpEchoServer(port);
    // initState 不是一个 async 函数，这里我们不能直接 await _server.start(),
    // future.then(...) 跟 await 是等价的
    _server.start().then((_) {
      // 等服务器启动后才创建客户端
      _client = HttpEchoClient(port);
    });
  }

  // ...
}
```

```dart
class MessageListScreen extends StatelessWidget {

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      // ...

      floatingActionButton: FloatingActionButton(
        onPressed: () async {
          final result = await Navigator.push(
              context,
              MaterialPageRoute(builder: (_) => AddMessageScreen())
          );
          // 以下是修改了的地方
          if (_client == null) return;
          // 现在，我们不是直接构造一个 Message，而是通过 _client 把消息
          // 发送给服务器
          var msg = await _client.send(result);
          if (msg != null) {
            messageListKey.currentState.addMessage(msg);
          } else {
            debugPrint('fail to send $result');
          }
        },
        // ...
      )
    );
  }
}
```

大功告成，在做了这么多工作以后，我们的应用现在是真正的 echo 客户端了，虽然看起来跟之前没什么两样。接下来，我们就做一些跟之前不一样的——把历史记录保存下来。


# 历史记录存储、恢复

## 获取应用的存储路径

为了获得应用的文件存储路径，我们引入多一个库：
```yaml
# pubspec.yaml
dependencies:
  # ...

  path_provider: ^0.4.1
```
通过它我们可以拿到应用的 file、cache 和 external storage 的路径：
```dart
import 'package:path_provider/path_provider.dart' as path_provider;

class HttpEchoServer {
  String historyFilepath;

  Future start() async {
    historyFilepath = await _historyPath();

    // ...
  }

  Future<String> _historyPath() async {
    // 获取应用私有的文件目录
    final directory = await path_provider.getApplicationDocumentsDirectory();
    return directory.path + '/messages.json';
  }
}
```

## 保存历史记录

```dart
class HttpEchoServer {

  void _echo(HttpRequest request) async {
    // ...

    // 原谅我，为了简单，我们就多存几次吧
    _storeMessages();
  }

  Future<bool> _storeMessages() async {
    try {
      // json.encode 支持 List、Map
      final data = json.encode(messages);
      // File 是 dart:io 里的类
      final file = File(historyFilepath);
      final exists = await file.exists();
      if (!exists) {
        await file.create();
      }
      file.writeAsString(data);
      return true;
    // 虽然文件操作方法都是异步的，我们仍然可以通过这种方式 catch 到
    // 他们抛出的异常
    } catch (e) {
      print('_storeMessages: $e');
      return false;
    }
  }
}
```


## 加载历史记录

```dart
class HttpEchoServer {

  // ...

  Future start() async {
    historyFilepath = await _historyPath();
    // 在启动服务器前先加载历史记录
    await _loadMessages();
    httpServer = await HttpServer.bind(InternetAddress.loopbackIPv4, port);
    // ...
  }

  Future _loadMessages() async {
    try {
      var file = File(historyFilepath);
      var exists = await file.exists();
      if (!exists) return;

      var content = await file.readAsString();
      var list = json.decode(content);
      for (var msg in list) {
        var message = Message.fromJson(msg);
        messages.add(message);
      }
    } catch (e) {
      print('_loadMessages: $e');
    }
  }
}
```

现在，我们来实现 _history 函数：
```dart
class HttpEchoServer {
  // ...

  void _history(HttpRequest request) {
    if (request.method != GET) {
      _unsupportedMethod(request);
      return;
    }

    String historyData = json.encode(messages);
    request.response.write(historyData);
    request.response.close();
  }
}
```
_history 的实现很直接，我们只是把 messages 全都返回给客户端。

接下来是客户端部分：
```dart
class HttpEchoClient {

  // ...

  Future<List<Message>> getHistory() async {
    try {
      // http 包的 get 方法用来执行 HTTP GET 请求
      final response = await http.get(host + '/history');
      if (response.statusCode == 200) {
        return _decodeHistory(response.body);
      }
    } catch (e) {
      print('getHistory: $e');
    }
    return null;
  }

  List<Message> _decodeHistory(String response) {
    // JSON 数组 decode 出来是一个 <Map<String, dynamic>>[]
    var messages = json.decode(response);
    var list = <Message>[];
    for (var msgJson in messages) {
      list.add(Message.fromJson(msgJson));
    }
    return list;
  }
}


class _MessageListState extends State<MessageList> {
  final List<Message> messages = [];

  @override
  void initState() {
    super.initState();

    const port = 6060;
    _server = HttpEchoServer(port);
    _server.start().then((_) {
      // 我们等服务器启动后才创建客户端
      _client = HttpEchoClient(port);
      // 创建客户端后马上拉取历史记录
      _client.getHistory().then((list) {
        setState(() {
          messages.addAll(list);
        });
      });
    });
  }

  // ...
}
```


# 生命周期

最后需要做的是，在 APP 退出后关闭服务器。这就要求我们能够收到应用生命周期变化的通知。为了达到这个目的，Flutter 为我们提供了 WidgetsBinding 类（虽然没有 Android 的 Lifecycle 那么好用就是啦）。
```dart
// 为了使用 WidgetsBinding，我们继承 WidgetsBindingObserver 然后覆盖相应的方法
class _MessageListState extends State<MessageList> with WidgetsBindingObserver {

  // ...

  @override
  void initState() {
    // ...
    _server.start().then((_) {
      // ...

      // 注册生命周期回调
      WidgetsBinding.instance.addObserver(this);
    });
  }

  @override
  void didChangeAppLifecycleState(AppLifecycleState state) {
    if (state == AppLifecycleState.paused) {
      var server = _server;
      _server = null;
      server?.close();
    }
  }
}

```

现在，我们的应用是这个样子的：
![flutter-echo-demo](flutter-echo-demo.gif)

所有的代码可以在 GitHub 上找到：
```shell
git clone https://github.com/Jekton/flutter_demo.git
cd flutter_demo
git checkout io-basic
```