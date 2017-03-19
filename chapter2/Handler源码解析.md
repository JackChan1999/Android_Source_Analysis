## Handler 之 源码解析

这篇文章是对 Handler 的源码解析。如果你是初学者，对 Handler 才开始学习，推荐你先看上一篇文章，[Handler 之 初识及简单应用](/HandlerIntroduce.md)，那是一篇对 Handler 的基础讲解，从为什么要有 Handler 到 如何使用 Handler 两个方面对 Handler 进行了介绍，并对我们熟知的常识『Android 中不允许在子线程中更新 UI』做了一个简要的分析。

好，下面开始 Handler 源码解析。

## Handler机制

其实讲 Handler 内部机制的博客已经很多了，但是自己还是要在看一遍，源码是最好的资料。

在具体看源码之前，有必要先理解一下 Handler、Looper、MessageQueue 以及 Message 他们的关系。

### 关系

Looper: 是一个消息轮训器，他有一个叫 loop() 的方法，用于启动一个循环，不停的去轮询消息池。

这里插一句，Looper 中的 loop() 方法是如何实现阻塞的呢？可查看 [Android-Discuss issue 245](https://github.com/android-cn/android-discuss/issues/245)

- MessageQueue: 就是上面说到的消息池
- Handler: 用于发送消息，和处理消息
- Message: 一个消息对象

现在要问了，他们是怎么关联起来的？一切都要从 Handler 的构造方法开始。如下所示

```java
final MessageQueue mQueue;
final Looper mLooper;
final Callback mCallback;
final boolean mAsynchronous;

public Handler(Callback callback, boolean async) {
    mLooper = Looper.myLooper();
    if (mLooper == null) {
        throw new RuntimeException(
            "Can't create handler inside thread that has not called Looper.prepare()");
    }
    mQueue = mLooper.mQueue;
    mCallback = callback;
    mAsynchronous = async;
  }
```

可以看到 Handler 本身定义了一个 MessageQueue 对象 mQueue，和一个 Looper 的对象 mLooper。

不过，对 Handler 的这两个成员变量的初始化都是通过 Looper 来赋值的。

```java
mLooper = Looper.myLooper();
mQueue = mLooper.mQueue;
```

现在，我们新建的 Handler 就和 Looper、MessageQueue 关联了起来，而且他们是一对一的关系，一个 Handler 对应一个
Looper，同时对应一个 MessageQueue 对象。这里给 MessageQueue 的赋值比较特殊，

```java
mQueue = mLooper.mQueue;
```
这里直接使用 looper 的 mQueue 对象，将 looper 的 mQueue 赋值给了 Handler 自己，现在 Looper 和 Handler 持有着同一个 MessageQueue 。

这里可以看到 Looper 的重要性，现在 Handler 中的 Looper 实例和 MessageQueue 实例都是通过 Looper 来完成设置的，那么下面我们具体看看 Looper 是怎么实例化的，以及他的 mQueue 是怎么来的。       

### Looper

从上面 Handler 的构造方法中可以看到，Handler 的 mLooper 是这样被赋值的。
```java
mLooper = Looper.myLooper();
```
接着看 myLooper() 的实现
```java
static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();

public static @Nullable Looper myLooper() {
    return sThreadLocal.get();
}
```
`这里出现了一个平时不怎么看到的 ThreadLocal 类，关于这个类，推荐去阅读任玉刚的一篇文章 - Android的消息机制之ThreadLocal的工作原理,讲的很不错。另外自己也写了一篇文章，用于讲解 ThreadLocal 的用法，以及他在 Handler 和 Looper 中的巧妙意义。`

[任玉刚 - Android的消息机制之ThreadLocal的工作原理](http://blog.csdn.net/singwhatiwanna/article/details/48350919)

这里他是通过 ThreadLocal 的 get 方法获得，很奇怪，之前我们没有在任何地方对 sThreadLocal 执行过 set 操作。
这里却直接执行 get 操作，返回的结果必然为空啊！

但是如果现在为空，我们在 new Handler() 时，程序就已经挂掉了啊，因为在 Handler 的构造方法中，如果执行 Looper.myLooper() 的返回结果为空。

```java
mLooper = Looper.myLooper();
if (mLooper == null) {
   throw new RuntimeException(
       "Can't create handler inside thread that has not called Looper.prepare()");
}
```

但是，我们的程序没有挂掉，这意味着，我们在执行 myLooper() 方法时，他返回的结果不为空。

为什么呢？那我们在 Looper 中看看，哪里有对应的 set 方法，如下所示,我们找到了一个全局静态方法 prepare

```java

public static void prepare() {
    prepare(true);
}

private static void prepare(boolean quitAllowed) {
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    sThreadLocal.set(new Looper(quitAllowed));
}
```

我们看到，在最后一行这里，执行了对应的 set 操作，这里把一个 new 出来的 Looper 直接 set 到 sThreadLocal 中。

但是我们不知道，到底什么时候，是谁调用了 prepare() 方法，从而给 sThreadLocal 设置了一个 Looper 对象。

后来在网上经过搜索，找到了答案，我们的 Android 应用在启动时，会执行到 ActivityThread 类的 main 方法，就和我们以前写的 java 控制台程序一样，其实 ActivityThread 的 main 方法就是一个应用启动的入口。

在这个入口里，会做很多初始化的操作。其中就有 Looper 相关的设置，代码如下

```java
public static void main(String[] args) {

    //............. 无关代码...............

    Looper.prepareMainLooper();

    Looper.loop();

    throw new RuntimeException("Main thread loop unexpectedly exited");
}
```

在这里，我们很清楚的看到，程序启动时，首先执行 Looper.prepareMainLooper() 方法，接着执行了 loop() 方法。

先看看 prepareMainLooper 方法。

```java
public static void prepareMainLooper() {
    prepare(false);
    synchronized (Looper.class) {
        if (sMainLooper != null) {
            throw new IllegalStateException("The main Looper has already been prepared.");
        }
        sMainLooper = myLooper();
    }
}

public static @Nullable Looper myLooper() {
    return sThreadLocal.get();
}

private static void prepare(boolean quitAllowed) {
  if (sThreadLocal.get() != null) {
      throw new RuntimeException("Only one Looper may be created per thread");
  }
  sThreadLocal.set(new Looper(quitAllowed));
}
```

这里，首先调用了 prepare() 方法，执行完成后，sThreadLocal 成功绑定了一个 new Looper() 对象，然后执行

```java
sMainLooper = myLooper();
```

可以看看 sMainLooper 的定义，以及 myLooper() 方法；
```java
private static Looper sMainLooper;  // guarded by Looper.class    

public static @Nullable Looper myLooper() {
    return sThreadLocal.get();
}
```

现在的 sMainLooper 就有值了，也就是说，只要我们的 App 启动，sMainLooper 中就已经设置了一个 Looper 对象。以后调用
sMainLooper 的 get 方法将返回在程序启动时设置的 Looper，不会为空的。  

下面在看 main 方法中的 调用的 Looper.loop() 方法。 已经把一些无关代码删了，否则太长了，

```java
public static void loop() {

    //获得一个 Looper 对象
    final Looper me = myLooper();

    // 拿到 looper 对应的 mQueue 对象
    final MessageQueue queue = me.mQueue;

    //死循环监听(如果没有消息变化，他不会工作的) 不断轮训 queue 中的 Message
    for (;;) {
        // 通过 queue 的 next 方法拿到一个 Message
        Message msg = queue.next(); // might block
        //空判断
        if (msg == null)return;
        //消息分发   
        msg.target.dispatchMessage(msg);
        //回收操作  
        msg.recycleUnchecked();
    }
}
```
现在，想一个简单的过程，我们创建了一个 App,什么也不做，就是一个 HelloWorld 的 Android 应用，
此时，你启动程序，即使什么也不干，按照上面的代码，你应该知道的是，现在的程序中已经有一个 Looper 存在了。
并且还启动了消息轮询。 Looper.loop();

但是，目前来看，他们好像没什么用，只是存在而已。

此时你的项目如果使用了 Handler,你在主线程 new 这个 Handler 时，执行构造方法

```java
public Handler(Callback callback, boolean async) {
    mLooper = Looper.myLooper();
    if (mLooper == null) {
        throw new RuntimeException(
            "Can't create handler inside thread that has not called Looper.prepare()");
    }
    mQueue = mLooper.mQueue;
    mCallback = callback;
    mAsynchronous = async;
}
```

此时的 myLooper() 返回的 Looper 就是应用启动时的那个 Looper 对象，我们从 Looper 的构造方法得知，在 new Looper 时，会新建一个对应的消息池对象 MessageQueue

```java
private Looper(boolean quitAllowed) {
       mQueue = new MessageQueue(quitAllowed);
       mThread = Thread.currentThread();
}
```

那么在 Handler 的构造方法中，那个 mQueue 其实也是在应用启动时就已经创建好了。

现在再来回顾一下 Handler 的构造方法，在构造方法中，他为自己的 mQueue 和 mLooper 分别赋值，而这两个值其实在应用启动时，就已经初始化好了。

并且，现在已经启动了一个消息轮训，在监听 mQueue 中是不是有新的 Message !

现在这个轮训器已经好了，我们看发送消息的过程。

### SendMessage

我们使用 Handler 发送消息 mHandler.sendMessage(msg);

Handler 有好多相关的发送消息的方法。但是追踪源码，发现他们最终都来到了 Handler 的这个方法 sendMessageAtTime

```java
public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
    MessageQueue queue = mQueue;
    if (queue == null) {
        RuntimeException e = new RuntimeException(
                this + " sendMessageAtTime() called with no mQueue");
        Log.w("Looper", e.getMessage(), e);
        return false;
    }
    return enqueueMessage(queue, msg, uptimeMillis);
}
```

接着看 enqueueMessage() 方法体

```java
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
      msg.target = this;
      // 使用默认的 handler 构造方法时，mAsynchronous 为 false。
      if (mAsynchronous) {
          msg.setAsynchronous(true);
      }
      return queue.enqueueMessage(msg, uptimeMillis);
    }
```

这里有一句至关重要的代码

```java
msg.target = this;
```
我们看看 msg 的 target 是怎么声明的    
```java
Handler target;
```

意思就是每个 Message 都有一个类型为 Handler 的 target 对象，这里在 handler 发送消息的时候，最终执行到
上面的方法 enqueueMessage() 时,会自动把当前执行 sendMessage() 的 handler对象，赋值给 Message 的 target。

也就是说，Handler 发送了 Message，并且这个 Message 的 target 就是这个 Handler。

想想为什么要这么做？

这里再说一下，Handler 的作用，

* 发送消息
* 处理消息

先不看代码，我们可以想想，handler 发送了 message ,最终这个 message 会被发送到 MessageQueue 这个消息队列。
那么最终，谁会去处理这个消息。

在这里消息发送和处理遵循『谁发送，谁处理』的原则。

现在问题来了，就按照上面说的，谁发送，谁处理，那现在应该是 handler 自己处理了，但是他在哪里处理呢？到现在我们也没看到啊。

慢慢来，接下来继续看消息的传递。

现在，我们只要发送了消息，那么消息池 mQueue 就会增加一个消息，Looper 就会开始工作，之前已经说了，在应用启动的时候，已经启动了 Looper 的 loop() 方法，这个方法会不断的去轮训 mQueue 消息池，只要有消息，它就会取出消息，并处理，那他是怎么处理的呢？看一下 loop() 的代码再说。

```java
public static void loop() {

    //获得一个 Looper 对象
    final Looper me = myLooper();

    // 拿到 looper 对应的 mQueue 对象
    final MessageQueue queue = me.mQueue;

    //死循环监听(如果没有消息变化，他不会工作的) 不断轮训 queue 中的 Message
    for (;;) {
        // 通过 queue 的 next 方法拿到一个 Message
        Message msg = queue.next(); // might block
        //空判断
        if (msg == null)return;
        //消息分发   
        msg.target.dispatchMessage(msg);
        //回收操作  
        msg.recycleUnchecked();
    }
}
```

看 for()循环，他在拿到消息后，发现 msg 不为空，接着就会执行下面这句非常重要的代码
```java
msg.target.dispatchMessage(msg);
```

这里执行了 msg.target 的方法 dispatchMessage，上面已经在 sendMessage 时看到了，我们在发送消息时，会把 msg 的 target设置为 handler 本身，也就是说，handler 发送了消息，最终自己处理了自己刚刚分发的消息。恩恩，答案就在这里，『谁发送，谁处理』的道理在这里终于得到了体现。

那么他是怎么处理消息的？看看 Handler 的 dispatchMessage() 是怎么实现的。

```java
public void dispatchMessage(Message msg) {
    if (msg.callback != null) {
        handleCallback(msg);
    } else {
        if (mCallback != null) {
            if (mCallback.handleMessage(msg)) {
                return;
            }
        }
        handleMessage(msg);
    }
}
```
我们看到，我们没有给 msg 设置 callback 也没有给 handler 的 mCallback 设置过值，所以此时，会执行 handleMessage() 方法；

```java

public void handleMessage(Message msg) {

}
```
发现这是一个空方法，所以自己的新建 Handler 时，只要复写了这个方法，我们就可以接受到从子线程中发送过来的消息了 。

在看一遍自己定义 Handler 时，如何定义的，如何复写 handlerMessage
```java
private Handler mHandler = new Handler(){
     @Override
     public void handleMessage(Message msg) {
         super.handleMessage(msg);
         switch (msg.what){
             case 1:
                 Bitmap bitmap = (Bitmap) msg.obj;
                 imageView.setImageBitmap(bitmap);
                 break;
             case -1:
                 Toast.makeText(MainActivity.this, "msg "+msg.obj.toString(), Toast.LENGTH_SHORT).show();
                 break;
         }
     }
 };
```
在这里，我们处理了自己发送的消息，到此 Handler 的内部机制大体就分析完了。

但是从上面的 dispatchMessage 方法我们也能看出，Handler 在处理消息时的顺序是什么？

```java
public void dispatchMessage(Message msg) {
    if (msg.callback != null) {
        handleCallback(msg);
    } else {
        if (mCallback != null) {
            if (mCallback.handleMessage(msg)) {
                return;
            }
        }
        handleMessage(msg);
    }
}
```

他首先判断 Message 对象的 callback 对象是不是为空，如果不为空，就直接调用 handleCallback 方法，并把 msg 对象传递过去，这样消息就被处理了,我们来看 Message 的 handleCallback 方法：

```java
private static void handleCallback(Message message) {
    message.callback.run();
}
```

没什么好说的了，直接调用 Handler post 的 Runnable 对象的 run() 方法。

如果在发送消息时，我们没有给 Message 设置 callback 对象，那么程序会执行到 else 语句块，此时首先判断 Handler 的 mCallBack 对象是不是空的，如果不为空，直接调用 mCallback 的 handleMessage 方法进行消息处理。

最终，只有当 Handler 的 mCallback 对象为空，才会执行自己的 handleMessage 方法。

这是整个处理消息的流程。

现在就会想了，在处理消息时，什么时候才能执行到第一种情况呢，也就是通过 Message 的 callback 对象处理。

其实简单，查看源码发现

/*package*/ Runnable callback;

callback 是一个 Runnable 接口，那我们这怎么才能设置 Message 的 callback 的参数呢？最后观察发现，Handler 发送消息时，除了使用 sendMessage 方法，还可以使用一个叫 post 的方法，而他的形参正好就是 Runnable,我们赶紧拔出他的源码看看。

```java
public final boolean post(Runnable r)
{
   return  sendMessageDelayed(getPostMessage(r), 0);
}
```

接着看 getPostMessage() 这个方法。

```java
private static Message getPostMessage(Runnable r) {
    Message m = Message.obtain();
    m.callback = r;
    return m;
}
```

代码看到这里，已经很清楚了，getPostMessage() 返回了一个 Message 对象，这个对象中设置了刚才传递过来的 runnable 对象。

到这里，你应该明白了，在处理消息时，除了 Handler 自身的 handlerMessage() 方法设置处理，还可以直接在发消息时指定一个 runnable 对象用于处理消息。

另外上面通过 dispatchMessage() 的代码已经看出来，处理消息有三种情形，第一种直接使用 Message 的 running 对象处理，如果不行使用第二种 用 Handler 的 mCallback 对象处理，最后才考虑使用 handlerMessage 处理，关于第二种情形，这里就不分析了，自己试着看代码应该能找到

## 参考文章

[鸿洋_ - Android 异步消息处理机制 让你深入理解 Looper、Handler、Message三者关系](http://blog.csdn.net/lmj623565791/article/details/38377229)
