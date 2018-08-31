---
title: Flutter 开发（3）- 交互、动画、手势和事件处理
date: 2018-08-29 09:06:41
categories: Flutter
tags: Flutter
description: 在这一篇文章中，我们将通过实现一个 echo 客户端的前端页面来学习如何在 Flutter 中进行页面的跳转、手势事件处理。至于动画，我们弄一个小圆点，让他沿着正弦曲线运动，同时改变自身的颜色。
---

> 本文由`玉刚说写作平`台提供写作赞助
> 赞助金额：200元
> 原作者：`水晶虾饺`
> 版权声明：本文版权归微信公众号`玉刚说`所有，未经许可，不得以任何形式转载

在这一篇文章中，我们将通过实现一个 echo 客户端的前端页面来学习如何在 Flutter 中进行页面的跳转、手势事件处理。至于动画，我们弄一个小圆点，让他沿着正弦曲线运动，同时改变自身的颜色。


# 手势处理、获取文本

本节我们来实现一个用户输入的页面。UI 很简单，就是一个文本框和一个按钮。

为了获取文本，我们可以使用 `TextField` 并给它设置一个 `TextEditingController`。通过这个 `controller`，我们就能够拿到输入框中的文本。由于这里需要响应用户事件，必须使用 `StatefulWidget`：
```dart
class MessageForm extends StatefulWidget {
  @override
  State createState() {
    return _MessageFormState();
  }
}

class _MessageFormState extends State<MessageForm> {
  final editController = TextEditingController();

  // 对象被从 widget 树里永久移除的时候调用 dispose 方法（可以理解为对象要销毁了）
  // 这里我们需要主动再调用 editController.dispose() 以释放资源
  @override
  void dispose() {
    super.dispose();
    editController.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Padding(
      padding: EdgeInsets.all(16.0),
      child: Row(
        children: <Widget>[
          // 我们让输入框占满一行里除按钮外的所有空间
          Expanded(
            child: Container(
              margin: EdgeInsets.only(right: 8.0),
              child: TextField(
                decoration: InputDecoration(
                  hintText: 'Input message',
                  contentPadding: EdgeInsets.all(0.0),
                ),
                style: TextStyle(
                  fontSize: 22.0,
                  color: Colors.black54
                ),
                // 获取文本的关键，这里要设置一个 controller
                controller: editController,
                // 自动获取焦点。这样在页面打开时就会自动弹出输入法
                autofocus: true,
              ),
            ),
          ),
          InkWell(
            onTap: () => debugPrint('send: ${editController.text}'),
            onDoubleTap: () => debugPrint('double tapped'),
            onLongPress: () => debugPrint('long pressed'),
            child: Container(
              padding: EdgeInsets.symmetric(vertical: 10.0, horizontal: 16.0),
              decoration: BoxDecoration(
                color: Colors.black12,
                borderRadius: BorderRadius.circular(5.0)
              ),
              child: Text('Send'),
            ),
          )
        ],
      ),
    );
  }
}


class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Flutter UX demo',
      home: AddMessageScreen(),
    );
  }
}

class AddMessageScreen extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text('Add message'),
      ),
      body: MessageForm(),
    );
  }
}
```
这里的代码其实没有包含太多的新知识，我们只需要机械地加入一个 `controller` 就可以拿到输入框中的文本了。就按钮而言，这里本应该使用 `RaisedButton` 或 `FlatButton`。为了演示如何监听手势事件，我们这里故意自己用 `Container` 做了一个按钮，然后通过 `InkWell` 监听手势事件。`InkWell` 除了上面展示的几个事件外，还带有一个水波纹效果。如果不需要这个水波纹效果，读者可以使用支持更多手势事件的 G`estureDetector`。


# 在页面间跳转

我们知道，Flutter 里所有的东西都是 `widget`，所以，一个页面，也是 `widget`。我们的 echo 客户端共有两个页面，一个用于展示所有的消息，另一个页面用户输入消息，后者在上一小节我们已经写好了。下面，我们来实现用于展示消息的页面。

我们的页面包含一个列表和一个按钮，列表用于展示信息，按钮则用来打开上一节我们所实现的 `AddMessageScreen`。这里我们先添加一个按钮并实现页面间的跳转。
```dart
// 这是我们的消息展示页面
class MessageListScreen extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text('Echo client'),
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: () {
          // push 一个新的 route 到 Navigator 管理的栈中，以此来打开一个页面
          Navigator.push(
              context,
              MaterialPageRoute(builder: (_) => AddMessageScreen())
          );
        },
        tooltip: 'Add message',
        child: Icon(Icons.add),
      )
    );
  }
}
```

在消息的输入页面，我们点击 Send 按钮后就返回：
```dart
onTap: () {
  debugPrint('send: ${editController.text}');
  Navigator.pop(context);
}
```

最后，我们加入一些骨架代码，实现一个完整的应用：
```dart
void main() {
  runApp(MyApp());
}

class MyApp extends StatelessWidget {

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Flutter UX demo',
      home: MessageListScreen(),
    );
  }
}
```

Flutter 引入了 route 的概念。为了打开一个新的页面，我们创建一个 `MaterialPageRoute` 并把它 push 到 `Navigator` 管理的栈中。返回前一个页面时，只需要 pop 这个 route 即可。

我们还可以在 `MaterialApp` 里设置好每个 route 对应的页面，然后使用 `Navigator.pushNamed(context, routeName)` 来打开它们：
```dart
MaterialApp(
  // Start the app with the "/" named route. In our case, the app will start
  // on the FirstScreen Widget
  initialRoute: '/',
  routes: {
    // When we navigate to the "/" route, build the FirstScreen Widget
    '/': (context) => HomeScreen(),
    // When we navigate to the "/second" route, build the SecondScreen Widget
    '/about': (context) => AboutScreen(),
  },
);
```

但是，上面代码所提供的功能还不够，我们需要从 `AddMessageScreen` 中返回一个消息。下面我们就来看看如何获取返回值：
```dart
// 首先我们对数据建模
class Message {
  final String msg;
  final int timestamp;

  Message(this.msg, this.timestamp);

  @override
  String toString() {
    return 'Message{msg: $msg, timestamp: $timestamp}';
  }
}


onTap: () {
  debugPrint('send: ${editController.text}');
  final msg = Message(
    editController.text,
    DateTime.now().millisecondsSinceEpoch
  );
  // 为了返回一个值，我们把它传递给 pop
  Navigator.pop(context, msg);
},


floatingActionButton: FloatingActionButton(
  onPressed: () async {
    // push 一个新的 route 到 Navigator 管理的栈中，以此来打开一个页面
    // Navigator.push 会返回一个 Future<T>，如果你对这里使用的 await
    // 不太熟悉，可以参考
    // https://www.dartlang.org/guides/language/language-tour#asynchrony-support
    final result = await Navigator.push(
        context,
        MaterialPageRoute(builder: (_) => AddMessageScreen())
    );
    debugPrint('result = $result');
  },
  // ...
)
```


# 把数据展示到 ListView

```dart
class MessageList extends StatefulWidget {

  // 先忽略这里的参数 key，后面我们就会看到他的作用了
  MessageList({Key key}): super(key: key);

  @override
  State createState() {
    return _MessageListState();
  }
}

class _MessageListState extends State<MessageList> {
  final List<Message> messages = [];

  @override
  Widget build(BuildContext context) {
    return ListView.builder(
      itemCount: messages.length,
      itemBuilder: (context, index) {
        final msg = messages[index];
        final subtitle = DateTime.fromMillisecondsSinceEpoch(msg.timestamp)
            .toLocal().toIso8601String();
        return ListTile(
          title: Text(msg.msg),
          subtitle: Text(subtitle),
        );
      }
    );
  }

  void addMessage(Message msg) {
    setState(() {
      messages.add(msg);
    });
  }
}
```
这段代码里唯一的新知识就是给 `MessageList` 的 `key` 参数，我们下面先看看如何使用他，然后再说明它的作用：
```dart
class MessageListScreen extends StatelessWidget {

  final messageListKey = GlobalKey<_MessageListState>(debugLabel: 'messageListKey');

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text('Echo client'),
      ),
      body: MessageList(key: messageListKey),
      floatingActionButton: FloatingActionButton(
        onPressed: () async {
          // push 一个新的 route 到 Navigator 管理的栈中，以此来打开一个页面
          // Navigator.push 会返回一个 Future<T>，如果你对这里使用的 await
          // 不太熟悉，可以参考
          // https://www.dartlang.org/guides/language/language-tour#asynchrony-support
          final result = await Navigator.push(
              context,
              MaterialPageRoute(builder: (_) => AddMessageScreen())
          );
          debugPrint('result = $result');
          if (result is Message) {
            messageListKey.currentState.addMessage(result);
          }
        },
        tooltip: 'Add message',
        child: Icon(Icons.add),
      )
    );
  }
}
```
引入一个 `GlobalKey` 的原因在于，`MessageListScreen` 需要把从 `AddMessageScreen` 返回的数据放到 `_MessageListState` 中，而我们无法从 `MessageList` 拿到这个 state。

`GlobalKey` 的是应用全局唯一的 key，把这个 key 设置给 `MessageList`，我们就能够通过这个 key 拿到对应的 `statefulWidget` 的 `state`。

现在，整体的效果是这个样子的：
![message-list](message-list.gif)

如果你遇到了麻烦，在 Github 上找到所有的代码：
```shell
git clone https://github.com/Jekton/flutter_demo.git
cd flutter_demo
git checkout ux-basic
```

前面我们说要写一个 echo 程序，到目前为止只是实现了一些 UI，剩余的逻辑我们将在下一篇文章完成。最后，由于没能在我们的例子里找到适合使用动画的地方，下面我们在一个独立的上下文里学习 Flutter 的动画。


# 动画

在这一节我们来画一个小圆点，它往复不断地在正弦曲线上运动。

![](sin-curve.gif)

下面我们先来实现小圆点沿着曲线运动的效果：
```dart
import 'dart:async';
import 'dart:math' as math;

import 'package:flutter/animation.dart';
import 'package:flutter/material.dart';

class AnimationDemoView extends StatefulWidget {
  @override
  State createState() {
    return _AnimationState();
  }
}

class _AnimationState extends State<AnimationDemoView>
    with SingleTickerProviderStateMixin {

  static const padding = 16.0;

  AnimationController controller;
  Animation<double> left;

  @override
  void initState() {
    super.initState();
    // 只有在 initState 执行完，我们才能通过 MediaQuery.of(context) 获取
    // mediaQueryData。这里通过创建一个 Future 从而在 Dart 事件队列里插入
    // 一个事件，以达到延后执行的目的（类似于在 Android 里 post 一个 Runnable）
    // 关于 Dart 的事件队列，读者可以参考 https://webdev.dartlang.org/articles/performance/event-loop
    Future(_initState);
  }

  void _initState() {
    // 1. 创建一个 Animation<T>，最简单的方式就是直接使用 AnimationController。
    // Animation 用户控制动画的进度、状态，但它并不关心屏幕上是什么东西在做动画。
    // AnimationController 输出的值在 0 ~ 1 之间
    controller = AnimationController(
        duration: const Duration(milliseconds: 2000),
        // 注意类定义的 with SingleTickerProviderStateMixin，提供 vsync 最简单的方法
        // 就是继承一个 SingleTickerProviderStateMixin。这里的 vsync 跟 Android 里
        // 的 vsync 类似，用来提供时针滴答，触发动画的更新。
        vsync: this);

    // 我们通过 MediaQuery 获取屏幕宽度
    final mediaQueryData = MediaQuery.of(context);
    final displayWidth = mediaQueryData.size.width;
    debugPrint('width = $displayWidth');
    // 2. 我们用 Tween 把 controller 输出的 0 ~ 1 之间的值映射到 [begin, end]
    // Tween.animate(controller) 返回一个 Animatable<T>，通过这个 Animatable<T> 我们
    // 可以获取映射过的值
    left = Tween(begin: padding, end: displayWidth - padding).animate(controller)
      // 每一帧都会回调这里添加的回调函数
      ..addListener(() {
        // 3. 调用 setState 触发他重新 build 一个 Widget。在 build 方法里，我们根据
        //    Animatable<T> 的当前值来创建 Widget，达到动画的效果（类似 Android 的属
        //    性动画）。
        setState(() {
          // nothing have to do
        });
      })
      // 监听动画状态变化
      ..addStatusListener((status) {
        // 这里我们让动画往复不断执行

        // 一次动画完成
        if (status == AnimationStatus.completed) {
          // 我们让动画反正执行一遍
          controller.reverse();
        // 反着执行的动画结束
        } else if (status == AnimationStatus.dismissed) {
          // 正着重新开始
          controller.forward();
        }
      });
    // 4. 开始动画
    controller.forward();
  }

  @override
  Widget build(BuildContext context) {
    // 假定一个单位是 24
    final unit = 24.0;
    final marginLeft = left == null ? padding : left.value;

    // 把 marginLeft 单位化
    final unitizedLeft = (marginLeft - padding) / unit;
    final unitizedTop = math.sin(unitizedLeft);
    // unitizedTop + 1 是了把 [-1, 1] 之间的值映射到 [0, 2]
    // (unitizedTop+1) * unit 后把单位化的值转回来
    final marginTop = (unitizedTop + 1) * unit + padding;
    return Container(
      // 我们根据动画的进度设置圆点的位置
      margin: EdgeInsets.only(left: marginLeft, top: marginTop),
      // 画一个小红点
      child: Container(
        decoration: BoxDecoration(
            color: Colors.red, borderRadius: BorderRadius.circular(7.5)),
        width: 15.0,
        height: 15.0,
      ),
    );
  }

  @override
  void dispose() {
    super.dispose();
    controller.dispose();
  }
}


void main() {
  runApp(MyApp());
}

class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Flutter animation demo',
      home: Scaffold(
        appBar: AppBar(title: Text('Animation demo')),
        body: AnimationDemoView(),
      ),
    );
  }
}
```

上面的动画中，我们只是对位置做出了改变，下面我们将在位置变化的同时，也让小圆点从红到蓝进行颜色的变化。
```dart
class _AnimationState extends State<AnimationDemoView>
    with SingleTickerProviderStateMixin {

  // ...

  Animation<Color> color;

  void _initState() {
    // ...

    color = ColorTween(begin: Colors.red, end: Colors.blue).animate(controller);
    controller.forward();
  }

  @override
  Widget build(BuildContext context) {
    // ...

    final color = this.color == null ? Colors.red : this.color.value;
    return Container(
      // 我们根据动画的进度设置圆点的位置
      margin: EdgeInsets.only(left: marginLeft, top: marginTop),
      // 画一个小圆点
      child: Container(
        decoration: BoxDecoration(
            color: color, borderRadius: BorderRadius.circular(7.5)),
        width: 15.0,
        height: 15.0,
      ),
    );
  }
}
```

在 GitHub 上，可以找到所有的代码：
```
git clone https://github.com/Jekton/flutter_demo.git
cd flutter_demo
git checkout sin-curve
```

在这个例子中，我们还可以加多一个效果，让小圆点在运动的过程中大小也不断变化，这个就留给读者作为练习。更多的 Flutter 动画知识，可以参考 [https://flutter.io/animations/](https://flutter.io/animations/)。
