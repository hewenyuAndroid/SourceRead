### 写在前面
网上关于 `Handler` 的文章有很多有些写的也是比较好的，本文主要是为了记录自己阅读源码的理解和体会，加深自己对源码的理解，记得很早的时候自己也看过 `Handler` 的源码，当时没有做笔记有些地方记得也不是很清楚，因此以后的源码阅读最好都可以自己总结出一篇文章会比较好；

### 关于 Handler 的使用场景
关于 ` Handler` 对象，我们肯定不会陌生，我们在子线程拿到数据，通过 `Handler` 通知主线程通知UI就可以使用 `Handler` 来帮我们完成这个通知的功能；官方对 `Handler` 的说明  [Handler文档](https://developer.android.google.cn/reference/android/os/Handler) 感兴趣的可以去看下，这里我们可以找到官方对 `Handler` 对象的主要用途:
> (1) to schedule messages and runnables to be executed at some point in the future
> (2) to enqueue an action to be performed on a different thread than your own.

大概的意思就是第一点让我们的 Message/Runnable 对象可以在将来的某个时刻执行，第二点是将不同线程中的操作加入队列；
整套 `Handler` 的消息机制我们将分析的类有:`Handler` 、`Looper`、`MessageQueue`、`Message`；

### Handler 对象的创建
创建 `Handler` 对象无非是通过 new 来创建，这里没什么好讲的，不过下面我们需要分析它的构造函数来帮助我们理解，`Handler` 的重载构造函数有很多，如果仔细观察我们不难发现其最终的区别就是是否传入了指定的`Looper` 对象，关于 `Looper` 这里先不展开，因此我们只看第一类没有指定默认`Looper` 对象的构造；
```Java
// 此类构造最终都是走到了这个方法
public Handler(Callback callback, boolean async) {
    // 省略代码...
    // 1. 获取当前创建 Handler 对象所在的线程
    mLooper = Looper.myLooper();
    if (mLooper == null) {
        throw new RuntimeException(
            "Can't create handler inside thread that has not called Looper.prepare()");
    }
    // 2. 根据拿到的 Looper 对象获取消息队列
    mQueue = mLooper.mQueue;
    mCallback = callback;
    mAsynchronous = async;
}
```
上述代码中我们知道了我们在通过 new 创建 `Handler` 对象的时候，系统帮我们做了哪些事情，主要就是获取一个 `Looper` 对象以及 `Looper` 对象中的一个消息队列，关于 `Looper` 对象和 `MessageQueue` 对象，我们在后面会分析这里不做展开，我们只需要知道我们在 `Handler` 创建的时候系统帮我们干了这两件事情就可以；

### 发送消息
上面我们分析了 `Handler` 对象的创建，对象创建好了接下来就应该是发送消息了，通常我们发送消息的方式也有很多，根据使用方式的不同，我们可以将我们的消息大致分为两大类，一个是传入 `Message` 对象发送的消息，另外一个是传入 `Runnable` 对象发送的消息，具体可以查看下面两张图片:

![sendMessage](https://upload-images.jianshu.io/upload_images/7082912-ead373d5c7edecd0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![post](https://upload-images.jianshu.io/upload_images/7082912-776d837012506344.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### sendMessage()
这里的 sendMessage 并不是只代表 `sendMessage(Message message)` 方法，而是指上面第一张图中的这一类方法，这里跟着源码走一遍我们就知道了 `sendEmptyMessage()` 方法实际上是由系统帮我们**封装**了 `Message` 对象，最终都会走到我们的 `sendMessageAtTime()` 方法将我们的 `Message` 对象放到消息队列中:
```Java
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

private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
    // 这里将当前发送消息的 Handler 设置成为消息对象的 target
    msg.target = this;
    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
    // 这里调用 MessageQueue 对象的入队方法
    return queue.enqueueMessage(msg, uptimeMillis);
}
```

##### post()
同样这里的 post 并不是只代表 `post(Runnable r)` 方法，我们可以看下这几个重载的方法:
```Java
public final boolean post(Runnable r)
{
   return  sendMessageDelayed(getPostMessage(r), 0);
}

public final boolean postAtTime(Runnable r, long uptimeMillis)
{
    return sendMessageAtTime(getPostMessage(r), uptimeMillis);
}

    public final boolean postDelayed(Runnable r, long delayMillis)
{
    return sendMessageDelayed(getPostMessage(r), delayMillis);
}
```
很明显，该方法也是调用了上诉的 `sendMessage` 方法，并且将 `Runnable` 对象封装成了 `Message` 对象，封装的逻辑也是非常的简洁，就是 `Runnable` 对象赋值给了 `Message` 的成员变量；
```Java
private static Message getPostMessage(Runnable r) {
    Message m = Message.obtain();
    m.callback = r;
    return m;
}
```

到这里我们分析了 `Handler` 对象的创建以及发送消息的源码，发送消息最终都是调用了 `sendMessageAtTime()` 这个方法；
至此我们分析了 `Handler` 的创建和发送消息的源码，知道了 `Handler` 中的 `MessageQueue` 对象是从构造中获取的 `Looper`  对象中拿到的，并且我们发送的消息是调用了 `queue.enqueueMessage(msg, uptimeMillis);` 这行代码将我们的消息放入到消息队列中，至于我们的消息在队列中是如何排序的我们接下来通过源码来分析；

### MessageQueue 消息队列
我们接着上面的源码来分析下我们发送的 `Message` 对象是如何进行存储的：
```Java
boolean enqueueMessage(Message msg, long when) {
    // 省略代码...
    synchronized (this) {
        // 省略代码...
        // 将我们的消息对象设置为使用中
        msg.markInUse();
        // 设置消息发送的时间
        msg.when = when;
        // 创建临时的消息对象
        Message p = mMessages;
        boolean needWake;
        if (p == null || when == 0 || when < p.when) {
            // New head, wake up the event queue if blocked.
            // 这里需要分析下这里的判断逻辑
            // 1.  p == null 说明 mMessage = null，也就是我们传进来的 Message 为队列的第一条消息；
            // 2. when == 0 ,说明我们发送的消息需要立即发送，需要放入队列的头部；
            // 3. when < p.when ，说明我们的消息的发送时间比队列里的第一条消息的时间早，需要放置在队列的头部；
            msg.next = p;
            mMessages = msg;
            needWake = mBlocked;
        } else {
            // Inserted within the middle of the queue.  Usually we don't have to wake
            // up the event queue unless there is a barrier at the head of the queue
            // and the message is the earliest asynchronous message in the queue.
            needWake = mBlocked && p.target == null && msg.isAsynchronous();
            // 这里可以看出是一个典型的链表结构
            Message prev;
            for (;;) {
                prev = p;
                p = p.next;
                if (p == null || when < p.when) {
                    // p == null 通过循环判断 prev 是否是队列的最后一条消息;
                    // when < p.when 判断我们需要加入队列的这条消息需要发送的时间是否
                        //  在 prev 和 p 这两个对象的发送时间之间；
                    break;
                }
                if (needWake && p.isAsynchronous()) {
                    needWake = false;
                }
            }
             // 剪断链表，将我们的消息插入 prev 和 p 这两个对象之间
            msg.next = p; // invariant: p == prev.next
            prev.next = msg;
        }

        // We can assume mPtr != 0 because mQuitting is false.
        if (needWake) {
            nativeWake(mPtr);
        }
    }
    return true;
}
```
具体的代码分析我都已经写了注释，这里我们对上述的代码做一个总结;
1. `MessageQueue` 消息队列实际上一个链表结构，类似 `Java` 中的 `LinkedList` 对象，通过链表形式有个好处就是我们消息的增删非常的快；
2. `MessageQueue` 队列中的消息顺序是根据消息需要发送的时间排序好的，并不等同于我们调用发送消息方法的顺序；
![MessageQueue](https://upload-images.jianshu.io/upload_images/7082912-36cb0492e8547423.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### Looper 消息循环器
根据上面的分析我们了解到了通过 `Handler` 对象发送的消息最终会根据消息需要发送的时间放入到 `MessageQueue` 对象中排列，而 `Handler` 中单的 `MessageQueue` 对象则是来自于 `Looper` 对象中，下面我们重点分析下 `Looper` 这个类;
```Java
// 私有的构造
private Looper(boolean quitAllowed) {
    // 创建消息队列
    mQueue = new MessageQueue(quitAllowed);
    // 获取当前的线程
    mThread = Thread.currentThread();
}
```

##### prepare() 
我们在 `Looper` 的构造方法中发现消息队列是在这里实例化对象的，同时还获取了当前的线程，但是这里的构造函数是私有的，因此我们判断 `Looper` 对象的实例化是在其内部完成的，通过阅读源码我们可以发现其 `prepare()` 方法内部实例化了一个 `Looper` 对象;
```Java
// Looper 类
public static void prepare() {
    prepare(true);
}

// Looper 类
private static void prepare(boolean quitAllowed) {
    // 这里表明一个线程中的 Looper 对象只能调用一次  prepare() 方法，
    // 也就是只能实例化一次 Looper 对象，具体查看 sThreadLocal.get() 方法的解析
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    // 将实例化完成的 Looper 对象缓存到 ThreadLocal 对象中
    sThreadLocal.set(new Looper(quitAllowed));
}

// ThreadLocal 类
public void set(T value) {
    // 获取当前的线程
    Thread t = Thread.currentThread();
    // 获取线程的 ThreadLocalMap 对象
    ThreadLocalMap map = getMap(t);
    // 将我们新创建的 Looper 对象缓存到
    // 当前线程的 ThreadLocalMap 对象中
    // ThreadLocalMap 大家把他理解为一个HashMap即可
    // 这里不做展开，不懂的可以网上找找资料
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}

// ThreadLocal 类
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}

// ThreadLocal 类
public T get() {
    // 获取当前的线程
    Thread t = Thread.currentThread();
    // 获取线程中的 ThreadLocalMap 对象
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        // 将 Looper 中的 ThreadLocal 对象作为 Key，获取对应的 Entry
        // 然后根据 Entry 获取对应的 Looper 对象，
        // 这里需要注意的是 Looper 中的 ThreadLocal 对象为静态的成员变量
        // 也就意味着该变量在内存中只有一份也就意味着这个 Key 是唯一的；
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            // 这里返回缓存的 Looper 对象
            return result;
        }
    }
    // 如果该线程是第一次调用 prepare() 方法则会执行到这里
    // 查看setInitialValue() 的代码可知其返回的是一个 null 
    return setInitialValue();
}
```
上面的代码不多，也很好理解，具体的地方也都写有注释，这里我们做一下总结：
1. `Looper` 的构造函数是私有的，我们只能调用其 `prepare()` 方法来实例化 `Looper` 对象，但是实例化完成的对象是缓存在当前线程的 `ThreadLocalMap` 对象中的；
2. `Looper` 中的消息队列是在其实例化的时候创建的，同时获取了创建`Looper` 对象时所在的线程；
3. 在 `prepare()`  方法中我们可以发现每个线程只能调用一次该方法也就是每个线程最多只能持有一个 `Looper` 对象，这一点会在调用 `prepare()` 方法的时候进行判断；
4. 根据上面 `Handler` 创建时的分析，我们可以知道 `Handler` 中的 `Looper` 对象也是通过调用上面的 `sThreadLocal.get()` 方法获取的，即当前 `Thread` 对象中的 `ThreadLocalMap` 中缓存的 `Looper` 对象；
这里我们在返回去看 `Handler` 的构造函数，我们发现如果返回的 `Looper` 对象为空即没有在当前线程调用 `Looper.prepare()` 方法就去创建`Handler`对象系统是会抛出异常的:
```Java
// handler 类
public Handler(Callback callback, boolean async) {
    // 省略代码...
    mLooper = Looper.myLooper();
    if (mLooper == null) {
        throw new RuntimeException(
            "Can't create handler inside thread that has not called Looper.prepare()");
    }
    // 省略代码...
}
```
这也就是为什么我们在子线程中直接创建 `Handler` 对象系统会抛异常的原因，即下面的代码:
```Java
// 示例代码
new Thread(new Runnable() {
    @Override
    public void run() {
        // 子线程中不加上这句话创建Handler对象是会抛异常的
        Looper.prepare();
        Handler handler = new Handler(){
            @Override
            public void handleMessage(Message msg) {
                Log.e("TAG", "hahaha");
            }
        };
        handler.sendEmptyMessage(100);
        // 子线程中的这个位置不加上这句话发送的消息是无法接收的
        // 这里将会进入一个死循环
        Looper.loop();
    }
}).start();
```
这时同样有一个问题出现了，为什么我们在主线程中直接实例化 `Handler` 对象不会抛异常，不需要加上 `Looper.prepare()` 这行代码，如果加上反而会抛出我们上面提到 **Only one Looper may be created per thread** 异常，这个显然就是系统在主线程帮我们调用了 `Looper.prepare()` 方法，我们可以在应用启动的入口 `ActivityThread` 的 `main()` 方法里面找到这样几行代码:
```Java
// ActivityThread 类
public static void main(String[] args) {
    // 省略代码...
    Looper.prepareMainLooper();
    // 省略代码...
    Looper.loop();
    throw new RuntimeException("Main thread loop unexpectedly exited");
}
```

##### loop()
上面分析了`Looper` 对象是如何实例化的，接下来我们分析下它是如何将我们的消息队列中的消息发送出来的，上面的 `ActivityThread` 中的 `main()` 方法在最后调用了 `Looper.loop()` 方法，我们来分析下这个方法:
```Java
/**
 * Run the message queue in this thread. Be sure to call
 * {@link #quit()} to end the loop.
 */
 // Looper 类
public static void loop() {
    // 获取当前线程的 Looper 对象
    final Looper me = myLooper();
    // 获取 Looper 中的消息队列
    final MessageQueue queue = me.mQueue;
    // 省略代码...

    // 这里是一个死循环用于获取入对的 Message
    for (;;) {
        Message msg = queue.next(); // might block
        if (msg == null) {
            // No message indicates that the message queue is quitting.
            return;
        }
        
        try {
            // 上面我们分析过 Message 的 Target 对象就是发送该 Message
            // 的 Handler 对象，这里转到 Handler 中的该方法中分析
            msg.target.dispatchMessage(msg);
            
        } finally {
            if (traceTag != 0) {
                Trace.traceEnd(traceTag);
            }
        }

        // 回收 Message 对象，这里就是为什么我们在创建消息对象时
        // 不直接 new 而是调用 Message.obtain() 方法来获取的原因；
        msg.recycleUnchecked();
    }
}

// Handler 类
public void dispatchMessage(Message msg) {
    // 这里判断如果 Message 对象中的 Callback 不为空
    // 就执行其 callback 对象中的 run 方法
    // 如果为空就调用 handler 对象的 handleMessage() 方法
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

// Handler 类
private static void handleCallback(Message message) {
    message.callback.run();
}
```
loop() 方法中主要是将我们的消息队列中的消息发送到指定的 `Handler` 对象中，这里我们对 `loop()` 方法做一个总结:
1. 调用 `loop()` 方法后系统会不断的去获取入队的消息对象；
2. 每个 `Message` 对象会根据其在发送时设置的  target 值（Handler）调用对应对象的 `dispatchMessage()` 方法；
3. `Handler` 对象中的 `dispatchMessage()`  方法会根据发送的消息的类型即我们上面的消息分类来回调不同的方法，如果是`sendMessage()` 类型的消息就会调用 `handMessage()` 方法，如果是 `post()` 类型的消息就会回调其 `Runnable` 对象的 `run()` 方法；
4. `loop()` 每次循环结束时会将 `Message` 对象进行回收，这一点我们将在下面进行分析；

### Message 消息对象
到这里整个 `Handler` 的消息机制大致上的流程我们就分析完毕了，还有一个关于消息对象的点我们在这里简单的分析下，通常我们发送的`Message` 对象都不是通过 new 创建的而是通过`Message.obtain()` 方法来获取的，有经验的朋友一看就知道这个肯定是为了缓存消息对象防止资源浪费，下面我们来分析下源码看它是如何实现的:
```JAVA
// Message 类
public static Message obtain() {
    synchronized (sPoolSync) {
        // 获取队列
        if (sPool != null) {
            Message m = sPool;
            // 将链表的第二个数据提到第一个位置
            sPool = m.next;
            // 将 Message 对象重置
            m.next = null;
            // 清除消息已被使用的 flag
            m.flags = 0; // clear in-use flag
            // 链表数量减1
            sPoolSize--;
            return m;
        }
    }
    // 如果是第一次，则直接实例化Message对象
    return new Message();
}
```
代码很简单因为我们的消息队列时链表结构，因此只需存储链表的头即可，每次通过 `obtain()` 拿到 `Message` 对象后我们只需要将该链表后面 对象提上来即可如果没有则会置为 null，上面这段代码是获取我们的缓存的 `Message` 对象，有获取就会有赋值，我们在 `loop()` 方法的 for 循环最后提到过它会调用 `msg.recycleUnchecked()` 方法将 `Message` 对象进行缓存:
```JAVA
// Message 类
void recycleUnchecked() {
    // 将消息置为已经使用过了
    flags = FLAG_IN_USE;
    what = 0;
    arg1 = 0;
    arg2 = 0;
    obj = null;
    replyTo = null;
    sendingUid = -1;
    when = 0;
    target = null;
    callback = null;
    data = null;
    
    // 判断是否超过了最大缓存数量
    // 没有超过则缓存否则不缓存
    synchronized (sPoolSync) {
        if (sPoolSize < MAX_POOL_SIZE) {
            next = sPool;
            sPool = this;
            sPoolSize++;
        }
    }
}
```
通过 Message 类的缓存方法的分析我们可以知道 Message 方法的缓存机制，即消息执行完毕后也会以队列的形式进行缓存，同时这个缓存的数量是有上限的即：MAX_POOL_SIZE = 50 个；

### 总结
本篇文章主要是对 `Handler` 执行流程的源码分析，主要通过 `Handler` ,`MessageQueue` ,`Looper` ,`Message` 这四个类进行的分析，中间涉及到的 `ThreadLocal` ,`ThreadLocalMap` 等知识没有进行扩展，感兴趣的朋友可以网上找找资料，写文章的主要意图也是帮助自己记忆一些阅读过的源码，欢迎有想法的朋友交流；