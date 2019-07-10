---
title: ScrollView 实现指北
date: 2019-07-08 09:08:47
categories: Android
tags: Android
---

在这之前，好几次想了解 `ScrollView` 实现，粗略翻翻，每次都没抓到要点；又实在没有非常迫切的需求，也就没有花太多心思了。最近工作上有个任务需要类似 `ScrollView` 的实现，只得下功夫研究一翻，于是有了这篇小短文。

首先要澄清的是，我不打算去分析 `ScrollView` 的源码，我们的目的只是为了弄清楚他最根本的实现方式。另外，涉及 `View` 的绘制时，也仅仅当他是一个黑盒，我们将会了解到如何使用 `View` 提供的 API 来实现自己的 scroll view。

对 `ScrollView` 来说，核心的有两部分 —— 滑动和绘制。滑动指竖直或水平的滚动；绘制则是说，子 view 只绘制一遍，每次滑动后，虽然可视内容变化了，子 view 的 `onDraw` 并不需要重新执行。下面我们分两小节来看看他们的实现。

### 滑动

如果你对 UI 比较熟悉，又或者曾经粗略浏览过 `ScrollView` 的源码，应该不难猜到滑动的实现。其实他滑动的就是 `View#scrollTo/scrollBy`。此外，如果需要 fling，`OverScroller` 可以帮上大忙。

现在（虽然还没写几个字），你不妨打开电脑：
1. 创建一个类 `MyScrollView` 并让他继承 `LinearLayout`
2. 设置 `orientation` 为 `vertical`，并添加上足够多的 `TextView`
3. 在竖直滑动的时候，调用 `scrollBy` 滑动内容；检测滑动最简单的方式就是使用 `GestureDetector` 了
4. 滑动的边界可以先忽略（滑动到哪里需要停止）

如果你的实现没问题，应该可以流畅地对 view 的内容进行滑动。唯一的问题是，滑出来的区域是空白的。不要慌，下面我们就来解决它。

### 绘制

老实说，`ScrollView` 的绘制这个问题困扰了我挺久，每次看他的源码，都找不到原因；这次因为一个偶然的机会，才发现了他的奥秘。

当我们把一个 `LinearLayout` 嵌在 `ScrollView` 里面的时候，`onMeasure` 拿到的高度是 unspecified 的，最后 `LinearLayout` 得到的高度会超过 `ScrollView`，并在绘制的时候把所有的内容都一次性绘制出来。具体的验证方法是，我们继承 `LinearLayout`，并把他嵌在 `ScrollView`，然后重写 `onMeasure`：
```Java
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    super.onMeasure(widthMeasureSpec, heightMeasureSpec);
    Log.d("Jekton", TAG + ".onMeasure: mode = " + MeasureSpec.getMode(heightMeasureSpec) +
            ", size = " + MeasureSpec.getSize(heightMeasureSpec));
    Log.d("Jekton", TAG + ".onMeasure: mode = " + MeasureSpec.getMode(heightMeasureSpec) +
            ", size = " + MeasureSpec.getSize(heightMeasureSpec));
}
```
关于 measure 这一步，`ScrollView` 实际上什么也没做，它继承了 `FrameLayout`，这些工作是父类帮它完成的。也就是说，如果我们需要实现自己的 scroll view，最简单的方法就是继承 `FrameLayout`。

现在，我们知道了 `LinearLayout` measure 的高度大于可见区域高度，下面需要解决的问题是如何绘制超出屏幕的内容。老样子，我们继续重写 `LinearLayout` 的方法：
```Java
@Override
protected void dispatchDraw(Canvas canvas) {
    Log.d("Jekton", TAG + ".dispatchDraw: " + canvas + ", " + canvas.getWidth() +
            "-" + canvas.getHeight(), new RuntimeException());
    super.dispatchDraw(canvas);
}
```
下面是我手机的一个打印：
```
D/Jekton: MyLinearLayout.onMeasure: mode = 0, size = 1552
D/Jekton: MyLinearLayout.onMeasure: 3024
D/Jekton: MyLinearLayout.dispatchDraw: android.view.DisplayListCanvas@fe4d0c5, 1080-3024
    java.lang.RuntimeException
        at com.example.ashmemdemo.MyLinearLayout.dispatchDraw(MyLinearLayout.java:37)
        at android.view.View.updateDisplayListIfDirty(View.java:18241)
        at android.view.View.draw(View.java:19042)
        at android.view.ViewGroup.drawChild(ViewGroup.java:4271)
        at android.view.ViewGroup.dispatchDraw(ViewGroup.java:4054)
        at android.view.View.draw(View.java:19317)
        at android.widget.ScrollView.draw(ScrollView.java:1777)
        at android.view.View.updateDisplayListIfDirty(View.java:18250)
        at android.view.View.draw(View.java:19042)
        at android.view.ViewGroup.drawChild(ViewGroup.java:4271)
        at android.view.ViewGroup.dispatchDraw(ViewGroup.java:4054)
        at android.view.View.updateDisplayListIfDirty(View.java:18241)
        at android.view.View.draw(View.java:19042)
        at android.view.ViewGroup.drawChild(ViewGroup.java:4271)
        at android.view.ViewGroup.dispatchDraw(ViewGroup.java:4054)
        at android.view.View.updateDisplayListIfDirty(View.java:18241)
        at android.view.View.draw(View.java:19042)
        at android.view.ViewGroup.drawChild(ViewGroup.java:4271)
        at android.view.ViewGroup.dispatchDraw(ViewGroup.java:4054)
        at android.view.View.updateDisplayListIfDirty(View.java:18241)
        at android.view.View.draw(View.java:19042)
        at android.view.ViewGroup.drawChild(ViewGroup.java:4271)
        at android.view.ViewGroup.dispatchDraw(ViewGroup.java:4054)
        at android.view.View.updateDisplayListIfDirty(View.java:18241)
        at android.view.View.draw(View.java:19042)
        at android.view.ViewGroup.drawChild(ViewGroup.java:4271)
        at android.view.ViewGroup.dispatchDraw(ViewGroup.java:4054)
        at android.view.View.draw(View.java:19317)
        at com.android.internal.policy.DecorView.draw(DecorView.java:915)
        at android.view.View.updateDisplayListIfDirty(View.java:18250)
        at android.view.ThreadedRenderer.updateViewTreeDisplayList(ThreadedRenderer.java:684)
        at android.view.ThreadedRenderer.updateRootDisplayList(ThreadedRenderer.java:690)
        at android.view.ThreadedRenderer.draw(ThreadedRenderer.java:804)
        at android.view.ViewRootImpl.draw(ViewRootImpl.java:3199)
        at android.view.ViewRootImpl.performDraw(ViewRootImpl.java:2997)
        at android.view.ViewRootImpl.performTraversals(ViewRootImpl.java:2526)
        at android.view.ViewRootImpl.doTraversal(ViewRootImpl.java:1515)
        at android.view.ViewRootImpl$TraversalRunnable.run(ViewRootImpl.java:7266)
        at android.view.Choreographer$CallbackRecord.run(Choreographer.java:981)
        at android.view.Choreographer.doCallbacks(Choreographer.java:790)
        at android.view.Choreographer.doFrame(Choreographer.java:721)
        at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:967)
        at android.os.Handler.handleCallback(Handler.java:808)
        at android.os.Handler.dispatchMessage(Handler.java:101)
        at android.os.Looper.loop(Looper.java:166)
        at android.app.ActivityThread.main(ActivityThread.java:7529)
        at java.lang.reflect.Method.invoke(Native Method)
        at com.android.internal.os.Zygote$MethodAndArgsCaller.run(Zygote.java:245)
        at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:921)
```

这段 log 有一个很关键的信息—— **canvas 的高度跟 measure 出来的高度一样**，这说明我们确确实实对超出屏幕的内容进行了绘制。

接下来我们根据调用栈来查找生成 canvas 的代码的位置。最后，在 `ViewGroup` 的 `updateDisplayListIfDirty` 可以找到这样一段代码：
```Java
public RenderNode updateDisplayListIfDirty() {
    // ...

    final DisplayListCanvas canvas = renderNode.start(width, height);
    dispatchDraw(canvas);

    // ...
}
```
这里需要说明一下，log 显示 updateDisplayListIfDirty 是 View 的方法，但我却在 `ViewGroup` 里找到他，是因为我测试手机是 Android 8，但源码用的是 Android 9。另外，之所以使用这种查找问题的方法，是因为我确实没看过 `View` 的源码，也不熟（流下了不学无术的泪水）。正因为如此，文章到这里就准备结束了，`View` 相关的东西，有机会再聊。



