---
title: Flutter 开发（2）- UI控件和布局
date: 2018-08-26 15:33:41
categories: Flutter
tags: Flutter
description: 本篇文章我们借助 Flutter 官网提供的两个 demo，来介绍 Flutter 的一些基本 UI 控件和布局方法。在第一个例子中，我们将不加修改（或仅做少量修改）地采用原文的代码来讲述问题。第二个例子的数据来自官网，但使用一种大家更为熟悉的方式开发。
---

> 本文由`玉刚说写作平`台提供写作赞助
> 赞助金额：200元
> 原作者：`水晶虾饺`
> 版权声明：本文版权归微信公众号`玉刚说`所有，未经许可，不得以任何形式转载

本篇文章我们借助 Flutter 官网提供的两个 demo，来介绍 Flutter 的一些基本 UI 控件和布局方法。原文地址是 [https://flutter.io/tutorials/layout/](https://flutter.io/tutorials/layout/)。在第一个例子中，我们将不加修改（或仅做少量修改）地采用原文的代码来讲述问题。第二个例子的数据来自官网，但使用一种大家更为熟悉的方式开发。


# 示例一

在这一节里，我们的目标是通过实现下面这个界面，来学习基本的 UI 空间和布局方法。

![lakes-diagram](lakes-diagram.png)


## 展示图片

1. 把图片 [lake](lake.png) 放到项目根目录的 `images` 文件夹下（如果没有，你需要自己创建一个）
2. 修改 `pubspec.yaml`，找到下面这个地方，然后把图片加进来
    ```
    flutter:
    
      # The following line ensures that the Material Icons font is
      # included with your application, so that you can use the icons in
      # the material Icons class.
      uses-material-design: true
    
      # To add assets to your application, add an assets section, like this:
      # assets:
      #  - images/a_dot_burr.jpeg
      #  - images/a_dot_ham.jpeg
    ```
    修改后如下：
    ```
    flutter:
    
      # The following line ensures that the Material Icons font is
      # included with your application, so that you can use the icons in
      # the material Icons class.
      uses-material-design: true
    
      # To add assets to your application, add an assets section, like this:
      assets:
        - images/lake.jpg
    ```

3. 现在，我们可以把这张图片展示出来了：
    ```dart
    void main() {
      runApp(MyApp());
    }
    
    class MyApp extends StatelessWidget {
      @override
      Widget build(BuildContext context) {
        return MaterialApp(
          title: 'Flutter UI basic 1',
          home: Scaffold(
            appBar: AppBar(
              title: Text('Top Lakes'),
            ),
            body: Image.asset(
              'images/lake.jpg',
              width: 600.0,
              height: 240.0,
              // cover 类似于 Android 开发中的 centerCrop，其他一些类型，读者可以查看
              // https://docs.flutter.io/flutter/painting/BoxFit-class.html
              fit: BoxFit.cover,
            )
          ),
        );
      }
    }
    ```

关于 `StatelessWidget` 和 `MaterialApp` 这些，我们在[Flutter 开发框架、流程、编译打包、调试](https://jekton.github.io/2018/08/26/flutter-first-app/)中已经有说明，这里不再赘述。

如果读者是初学 Flutter，建议在遇到不熟悉的 API 时翻一翻文档，并在文档中找到 demo 所使用的 API。我们的例子不可能覆盖所有的 API，通过这种方式熟悉文档后，读者就可以根据文档实现出自己想要的效果。不妨就从 `Image` 开始吧，在 [https://docs.flutter.io/flutter/widgets/Image/Image.asset.html](https://docs.flutter.io/flutter/widgets/Image/Image.asset.html) 找出上面我们使用的 `Image.asset` 构造函数的几个参数的含义，还有 `BoxFit` 的其他几个枚举值。


## 基本的布局

在这一小节，我们来实现图片下方的标题区域。完成这一小节后，读者对 Flutter 的布局将会有一个基本的认识。

![](title-section-diagram.png)

我们直接来看代码：
```dart
class _TitleSection extends StatelessWidget {
  final String title;
  final String subtitle;
  final int starCount;

  _TitleSection(this.title, this.subtitle, this.starCount);

  @override
  Widget build(BuildContext context) {
    // 我们知道，Flutter 里所有的东西都是 Widget。为了给 title section 加上 padding，
    // 这里我们给内容套一个 Container
    return Container(
      // 设置上下左右的 padding 都是 32px。类似的还有 EdgeInsets.only/symmetric 等
      padding: EdgeInsets.all(32.0),
      // 只有一个子元素的 widget，一般使用 child 参数来设置；像下面使用的 Row，包含有多
      // 个元素，对应的则是 children。
      // Row、Column 类似于我们 Android 开发里面使用的 LinearLayout，分别对应 orien-
      // tation 为 horizontal 和 vertical。
      child: Row(
        children: <Widget>[
          // 和 LinearLayout 一样，我们从左到右放入子元素。
          // Expanded 提供了 LinearLayout layout_weight 类似的功能。这里为了让标题占
          // 满屏幕宽度的剩余空间，用 Expanded 把标题包了起来
          Expanded(
            // 再次提醒读者，Expanded 只能包含一个子元素，使用的参数名是 child。接下来，
            // 为了在竖直方向放两个标题，加入一个 Column。
            child: Column(
              // Column 是竖直方向的，cross 为交叉的意思，也就是说，这里设置的是水平方向
              // 的对齐。在水平方向，我们让文本对齐到 start（读者可以修改为 end 看看效果）
              crossAxisAlignment: CrossAxisAlignment.start,
              children: <Widget>[
                // 聪明的你，这个时候肯定知道为什么突然加入一个 Container 了。
                // 跟前面一样，只是为了设置一个 padding
                Container(
                  padding: const EdgeInsets.only(bottom: 8.0),
                  child: Text(
                    title,
                    style: TextStyle(fontWeight: FontWeight.bold),
                  ),
                ),
                Text(
                  subtitle,
                  style: TextStyle(color: Colors.grey[500]),
                )
              ],
            ),
          ),

          // 这里是 Row 的第二个子元素，下面这两个就没用太多值得说的东西了。
          Icon(
            Icons.star,
            color: Colors.red[500],
          ),

          Text(starCount.toString())
        ],
      ),
    );
  }
}
```

这里，我们把这一整个区域的 UI 用一个类 `_TitleSection` 封装起来。下面我们总结一下：
1. layout 类型的控件分两种：只含一个子元素或包含多个子元素。前者使用参数 `child`，后者则是 `children`。
2. 为了让控件具有 padding、margin 和 width/height 等，我们可以使用一个 Container 把它包起来。
3. Row/Column 提供了类似于 `LinearLayout` 的功能。

建议读者粗略浏览一下 [https://flutter.io/widgets/](https://flutter.io/widgets/)，大概 Flutter 提供了哪些 widget 和它们对应的功能。以后在开发过程中，有需要的时候再详细查看相关文档。


## 对齐

接下来我们要做的这一部分在布局上所用到的知识，基本知识在上一小节我们都已经学习了。这里唯一的区别在于，三个按钮是水平分布的。

![](button-section-diagram.png)

```dart
Widget _buildButtonColumn(BuildContext context, IconData icon, String label) {
  final color = Theme.of(context).primaryColor;

  return Column(
    // main axis 跟我们前面提到的 cross axis 相对应，对 Column 来说，指的就是竖直方向。
    // 在放置完子控件后，屏幕上可能还会有一些剩余的空间（free space），min 表示尽量少占用
    // free space；类似于 Android 的 wrap_content。
    // 对应的，还有 MainAxisSize.max
    mainAxisSize: MainAxisSize.min,
    // 沿着 main axis 居中放置
    mainAxisAlignment: MainAxisAlignment.center,

    children: <Widget>[
      Icon(icon, color: color),
      Container(
        margin: const EdgeInsets.only(top: 8.0),
        child: Text(
          label,
          style: TextStyle(
            fontSize: 12.0,
            fontWeight: FontWeight.w400,
            color: color,
          ),
        ),
      )
    ],
  );
}

class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    //...

    Widget buttonSection = Container(
      child: Row(
	// 沿水平方向平均放置
        mainAxisAlignment: MainAxisAlignment.spaceEvenly,
        children: [
          _buildButtonColumn(context, Icons.call, 'CALL'),
          _buildButtonColumn(context, Icons.near_me, 'ROUTE'),
          _buildButtonColumn(context, Icons.share, 'SHARE'),
        ],
      ),
    );
  //...
}
```

关于 cross/main axis，看看下面这两个图就很清楚了：
![](column-diagram.png)
![](row-diagram.png)

关于 `mainAxisAlignment`，更多的信息可以查看 [https://docs.flutter.io/flutter/rendering/MainAxisAlignment-class.html](https://docs.flutter.io/flutter/rendering/MainAxisAlignment-class.html)。


## 全部放到一起

```dart
class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    final titleSection = _TitleSection(
        'Oeschinen Lake Campground', 'Kandersteg, Switzerland', 41);
    final buttonSection = ...;
    final textSection = Container(
        padding: const EdgeInsets.all(32.0),
        child: Text(
          '''
Lake Oeschinen lies at the foot of the Blüemlisalp in the Bernese Alps. Situated 1,578 meters above sea level, it is one of the larger Alpine Lakes. A gondola ride from Kandersteg, followed by a half-hour walk through pastures and pine forest, leads you to the lake, which warms to 20 degrees Celsius in the summer. Activities enjoyed here include rowing, and riding the summer toboggan run.
          ''',
          softWrap: true,
        ),
    );

    return MaterialApp(
      title: 'Flutter UI basic 1',
      home: Scaffold(
          appBar: AppBar(
            title: Text('Top Lakes'),
          ),
	  // 由于我们的内容可能会超出屏幕的长度，这里把内容都放到 ListView 里。
	  // 除了这种用法，ListView 也可以像我们在 Android 原生开发中使用 ListView 那样，
	  // 根据数据动态生成一个个 item。这个我们在下一节再来学习
          body: ListView(
            children: <Widget>[
              Image.asset(
                'images/lake.jpg',
                width: 600.0,
                height: 240.0,
                // cover 类似于 Android 开发中的 centerCrop，其他一些类型，读者可以查看
                // https://docs.flutter.io/flutter/painting/BoxFit-class.html
                fit: BoxFit.cover,
              ),

              titleSection,
              buttonSection,
              textSection
            ],
          ),
      )
    );
  }
}
}
```

现在，如果没有出错的话，运行后应该就可以看到下面这个页面。
![](Screenshot_20180826-172022.png)

如果你遇到了麻烦，可以在这里找到所有的源码：
```shell
git clone https://github.com/Jekton/flutter_demo.git
cd flutter_demo
git checkout ui-basic1
```

更多的布局知识，读者可以参考 [https://flutter.io/tutorials/layout/](https://flutter.io/tutorials/layout/)。


# 示例二

在这一小节我们来实现一个 list view。

![](listview.png)


这里我们采用的还是官网提供的例子，但是换一种方式来实现，让它跟我们平时使用 Java 时更像一些。由于这部分是建立在读者已经学习了上一节知识的基础上，不会很详细地对 API 进行讲解。希望读者在遇到不明白的地方，能够多查查文档。

首先给数据建模：
```dart
enum BuildingType { theater, restaurant }

class Building {
  final BuildingType type;
  final String title;
  final String address;

  Building(this.type, this.title, this.address);
}
```

然后实现每个 item 的 UI：
```dart
class ItemView extends StatelessWidget {
  final int position;
  final Building building;

  ItemView(this.position, this.building);

  @override
  Widget build(BuildContext context) {
    final icon = Icon(
        building.type == BuildingType.restaurant
            ? Icons.restaurant
            : Icons.theaters,
        color: Colors.blue[500]);

    final widget = Row(
      children: <Widget>[
        Container(
          margin: EdgeInsets.all(16.0),
          child: icon,
        ),
        Expanded(
          child: Column(
            crossAxisAlignment: CrossAxisAlignment.start,
            children: <Widget>[
              Text(
                building.title,
                style: TextStyle(
                  fontSize: 20.0,
                  fontWeight: FontWeight.w500,
                )
              ),
              Text(building.address)
            ],
          ),
        )
      ],
    );

    return widget;
  }
}
```

接着是 ListView。由于渲染机制不同，这里没必要弄个 adapter 来管理 widget：
```dart
class BuildingListView extends StatelessWidget {
  final List<Building> buildings;
  final OnItemClickListener listener;

  BuildingListView(this.buildings, this.listener);

  @override
  Widget build(BuildContext context) {
    return ListView.builder(
      itemCount: buildings.length,
      itemBuilder: (context, index) {
        return new ItemView(index, buildings[index], listener);
      }
    );
  }
}
```

现在，我们来给 item 加上点击事件。
```dart
typedef OnItemClickListener = void Function(int position);

class ItemView extends StatelessWidget {

  final int position;
  final Building building;
  final OnItemClickListener listener;

  // 这里的 listener 会从 ListView 那边传过来
  ItemView(this.position, this.building, this.listener);

  @override
  Widget build(BuildContext context) {
    final widget = ...;

    // 一般来说，为了监听手势事件，我们使用 GestureDetector。但这里为了在点击的时候有个
    // 水波纹效果，使用的是 InkWell。
    return InkWell(
      onTap: () => listener(position),
      child: widget
    );
  }
}

class BuildingListView extends StatelessWidget {
  final List<Building> buildings;
  final OnItemClickListener listener;

  // 这是对外接口。外部通过构造函数传入数据和 listener
  BuildingListView(this.buildings, this.listener);

  @override
  Widget build(BuildContext context) {
    return ListView.builder(
      itemCount: buildings.length,
      itemBuilder: (context, index) {
        return new ItemView(index, buildings[index], listener);
      }
    );
  }
}
```

最后加上一些脚手架代码，我们的列表就能够跑起来了：
```dart
void main() {
  runApp(MyApp());
}

class MyApp extends StatelessWidget {

  @override
  Widget build(BuildContext context) {
    final buildings = [
      Building(BuildingType.theater, 'CineArts at the Empire', '85 W Portal Ave'),
      Building(BuildingType.theater, 'The Castro Theater', '429 Castro St'),
      Building(BuildingType.theater, 'Alamo Drafthouse Cinema', '2550 Mission St'),
      Building(BuildingType.theater, 'Roxie Theater', '3117 16th St'),
      Building(BuildingType.theater, 'United Artists Stonestown Twin', '501 Buckingham Way'),
      Building(BuildingType.theater, 'AMC Metreon 16', '135 4th St #3000'),
      Building(BuildingType.restaurant, 'K\'s Kitchen', '1923 Ocean Ave'),
      Building(BuildingType.restaurant, 'Chaiya Thai Restaurant', '72 Claremont Blvd'),
      Building(BuildingType.restaurant, 'La Ciccia', '291 30th St'),

      // double 一下
      Building(BuildingType.theater, 'CineArts at the Empire', '85 W Portal Ave'),
      Building(BuildingType.theater, 'The Castro Theater', '429 Castro St'),
      Building(BuildingType.theater, 'Alamo Drafthouse Cinema', '2550 Mission St'),
      Building(BuildingType.theater, 'Roxie Theater', '3117 16th St'),
      Building(BuildingType.theater, 'United Artists Stonestown Twin', '501 Buckingham Way'),
      Building(BuildingType.theater, 'AMC Metreon 16', '135 4th St #3000'),
      Building(BuildingType.restaurant, 'K\'s Kitchen', '1923 Ocean Ave'),
      Building(BuildingType.restaurant, 'Chaiya Thai Restaurant', '72 Claremont Blvd'),
      Building(BuildingType.restaurant, 'La Ciccia', '291 30th St'),
    ];
    return MaterialApp(
      title: 'ListView demo',
      home: Scaffold(
        appBar: AppBar(
          title: Text('Buildings'),
        ),
        body: BuildingListView(buildings, (index) => debugPrint('item $index clicked'))
      ),
    );
  }
}
```

这个时候你应该可以看到像这样的界面了：
![](screenshot-listview.png)


如果你遇到了什么麻烦，可以查看 tag ui-basic2 的代码：
```shell
git clone https://github.com/Jekton/flutter_demo.git
cd flutter_demo
git checkout ui-basic2
```












