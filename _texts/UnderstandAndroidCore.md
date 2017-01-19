---
collection: texts
layout: post
title:  "了解Android核心：Looper、Handler、HandlerThread"
date:   2017-01-19 
category: Android
---

翻译原文:[Understanding Android Core: Looper, Handler, and HandlerThread](https://blog.mindorks.com/android-core-looper-handler-and-handlerthread-bd54d69fe91a#.zi94mawq8)  

这篇文章包括了Android Looper,Handler以及HandlerThread。这些都是Android操作系统的基础模块。

>我之前只在有限的范围内使用它们。使用的地方涉及发送任务到主线程，主要是在其他线程中更新主线的个UI。多线程操作的其他方法就是线程池、IntentService以及AsyncTask。

多线程和任务运行是老话题了。Java本身有`java.util.concurrent`包和`Fork/join`框架来辅助开发。有些库可以用来简化异步操作。现在，在响应式编程以及异步应用中，Rxjava是最流行的库。

*既然如此，为何还要讨论这个话题？*

Looper,Hander,HanderThread是异步操作的Android版解决方案。他们不是老话题,他们是构建Android框架的简洁结构。

对于初级开发者，强烈建议去了解他们背后的原理。老司机也应该去重温这些课题进行查漏补缺。

**提要：**

1. Android的主线程中已经包含了Looper 和 Handlers 。 对此可以理解为其主要的目的是为了构建非阻塞UI。

2. 开发者可能会纠结于第三方库的尺寸。如果自己的写的话又可能无法做到较高的效率和优化。最好的解决办法就是使用现有的资源。

3. 同样的问题可能会让公司或者独立开发者抛弃SDK。客户会有多种实现方式，但是他们却共用相同的Android框架接口。

4. 搞懂这些会全面增强对Android SDK以及通用包类的理解。

#### 让我们通过问答开始探索

>假定读者有一定的Java线程基础。如果你没有，请自行去学习先。

**Java的线程有什么缺陷?**

Java 的线程是一次性的,执行完run方法后就会消失。

**如何解决这个问题？**

线程是把双刃剑，我们可以通过把任务分配到多个线程处理来提高运行速度，但是过多的线程也会降低处理速度。线程创建本身就会消耗资源。所以最好的选择就是拥有合理数量的线程，重用他们来进行任务运行。

**线程重用的模型：**

1. 线程一直存活，在run方法中进入循环。

2. 线程内的任务放在队列中，然后串行执行。

3. 结束时线程必须终止。

**Android是如何实现的?** 

Android通过Looper, Handler，HandlerThread实现了上面的模型。

1. `message`是将要执行的任务,`Messagequeue`是存放`message`的队列。

2. `Handler`使用`Looper`将任务排列到`Messagequeue`。并且在任务移出队列的时候进行任务运行。 

3. `Looper`保证线程的存活，循环`Messagequeue`然后发送message到`Handler`使其运行任务。

4. 最后，线程会在调用Looper 的` quit()`方法时结束。

> 一个线程只能拥有一个Looper ,但是可以有多个Handler。

**为线程创建 Looper 和 Messagequeue**

线程在开始运行之后，通过调用`Looper.prepare()`来获取 `Looper` 和 `MessageQueue` 。 `Looper.prepare()` 会辨认调用它的线程，然后创建`Looper`和`MessageQueue`并将其关联到线程的`ThreadLocal`中。启动`Looper`需要调用`Looer.loop()`方法。类似的，停止`Looper`需要调用`Looper.quit()`。

```java
class LooperThread extends Thread {
      public Handler mHandler; 
 
      public void run() { 
          Looper.prepare();
 
          mHandler = new Handler() { 
              public void handleMessage(Message msg) { 
                 // 这里处理传递进来的message
                 // 会运行在非UI线程
              } 
          }; 
 
          Looper.loop();
      } 
  }
```

**为线程创建Handler**

通过线程的`Looper`实例化`Handler`会隐式的关联到该线程。我们也可以通过将线程的`Looper`作为`handler`构成函数的形参，来现式的将他们关联在一起。

```java
handler = new Handler() {
    @Override
    public void handleMessage(Message msg) {
        // process incoming messages here
        // this will run in the thread, which instantiates it
    }
};
```

通过`Handler`向`MessageQueue`发送消息有两种模式：
1. `Message`: 定义了多种处理消息数据的方法。通过设置`obj`变量来发送对象。

```java
Message msg = new Message();
msg.obj = "Ali send message";
handler.sendMessage(msg);
```
更多细节去查看[官方文档](https://developer.android.com/reference/android/os/Message.html)

2. `runnable`: `runnable`可以被投递到`MessageQueue`。例如：将任务投递到主线程运行。

```java
new Handler(Looper.getMainLooper()).post(new Runnable() {
    @Override
    public void run() {
        // this will run in the main thread
    }
});
```
上面的例子中，我们创建了`Handler`并关联到主线程的`Looper`。当我们投递`Runnable`时，会将其排列到主线程的`MessageQueue`中，并在主线程中运行。

*Handler有多种方式来操作Message，具体请去查看 [官方文档](https://developer.android.com/reference/android/os/Handler.html)*

创建自己的线程然后提供`Looper`,`MessageQueue`并不是优雅的解决办法。Android 提供了`HandlerThread`(Thread的子类)来简化操作。其内部以健壮的方式实现我们之前做的事情。所以，一定要用`HandlerThread`。

**创建HandlerThread的方法中，使用的最多的就是继承**

```java
private class MyHandlerThread extends HandlerThread {

    Handler handler;

    public MyHandlerThread(String name) {
        super(name);
    }

    @Override
    protected void onLooperPrepared() {
        handler = new Handler(getLooper()) {
            @Override
            public void handleMessage(Message msg) {
                // process incoming messages here
                // this will run in non-ui/background thread
            }
        };
    }
}
```
Note: 在`onLooperPrepared()`回调的时候进行`Handler`的实例化。此时`Handler`将会关联到对应的`Looper`。

1. `Looper`只有在HandlerThread的`start()`方法被调用后才进行准备工作。

2. `Handler`只有在`Looper`准备完成后才能与其进行关联。


**创建HandlerThread的其他方法**

```java
HandlerThread handlerThread = new HandlerThread("MyHandlerThread");
handlerThread.start();
Handler handler = new Handler(handlerThread.getLooper());
```
Note: HandlerThread需要调用`myHandlerThread.quit()`来释放资源终止线程的运行。

我建议熟悉上面的代码来了解其中的细节。

一个示例程序：[Post Office](https://github.com/MindorksOpenSource/post-office-simulator-looper-example)
