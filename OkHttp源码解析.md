### OkHttp
OkHttp 是目前Android开发中最主流的网络框架，关于 OkHttp 的介绍可以查看[项目地址](https://github.com/square/okhttp)；
本文将通过源码来分析 OkHttp 框架，毕竟只有了解了源码才能玩的6；

### 一个简单的网络请求
在分析源码之前我们先来看下 OkHttp 的使用方式，
```Java
 // 获取 OkHttpClient 对象，可以理解为一个浏览器
OkHttpClient client = new OkHttpClient.Builder()
        .build();
// 创建请求对象，可以理解为目标网址
Request request = new Request.Builder()
        .url("https://www.baidu.com")
        .build();
// 创建执行网络请求的 Call 对象
final Call call = client.newCall(request);
// 执行异步请求
call.enqueue(new Callback() {
    @Override
    public void onFailure(Call call, IOException e) {
    }

    @Override
    public void onResponse(Call call, Response response) throws IOException {
        Log.e(TAG, response.body().string());
    }
});
```
当然在执行上面这段代码的时候我们需要增加网络请求权限:
`<uses-permission android:name="android.permission.INTERNET" />`
返回的结果是百度的Html文本:
![返回结果](https://upload-images.jianshu.io/upload_images/7082912-ebfd600a27c971a3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

上面就是一个完整的OkHttp网络请求的使用，使用起来还是非常的方便的（本文主要是分析OkHttp源码，这里就不对Http相关的配置进行扩展了）；

### Call
通过上面的代码我们可以知道我们通过Builder模式创建了一个 `OkHttpClient`  和 一个`Request` ，这两个对象可以理解为一个浏览器客户端和一个请求地址的封装，而真正解析和执行网络请求过程的是 `Call` 对象；
接下来我们就从 `Call` 对象的创建开始分析:
```Java
// Call 是一个接口，这里通过 OkHttpClient 对象创建的对象是其实现类 RealCall 对象
Call call = client.newCall(request);

// 上面的方法最终会执行到 RealCall 类中的 newRealCall 方法，在这里创建需要返回的 RealCall 对象
static RealCall newRealCall(OkHttpClient client, Request originalRequest, boolean forWebSocket) {
    RealCall call = new RealCall(client, originalRequest, forWebSocket);
    call.eventListener = client.eventListenerFactory().create(call);
    return call;
}
```
分析上面的源码可以知道 `client.newCall(request);` 返回的实际上是一个 `RealCall` 对象，因此接下来的 `Call` 对象的同步请求和异步请求都是通过 `RealCall` 对象来实现的，这里我们直接对 `RealCall` 中的同步和异步请求进行分析:
```Java
// 同步请求
@Override 
public Response execute() throws IOException {
    synchronized (this) {
      // 每个 RealCall 对象只能执行一次网络请求
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    captureCallStackTrace();
    eventListener.callStart(this);
    try {
       // 通过 OkHttpClient 里面的 Dispatcher 对象对请求进行管理
      client.dispatcher().executed(this);
      // 通过拦截器进行网络请求拿到响应的 Response 对象
      Response result = getResponseWithInterceptorChain();
      if (result == null) throw new IOException("Canceled");
      return result;
    } catch (IOException e) {
      eventListener.callFailed(this, e);
      throw e;
    } finally {
      // 通过 Dispatcher 对象对结束网络请求的 RealCall 对象进行管理
      // 这里的 this 就是 RealCall 对象
      client.dispatcher().finished(this);
    }
}

// 异步请求
@Override 
public void enqueue(Callback responseCallback) {
    synchronized (this) {
      // 每个 RealCall 对象只能执行一次网络请求
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    captureCallStackTrace();
    eventListener.callStart(this);
    // 通过 OkHttpClient 里面的 Dispatcher 对象对请求进行管理
    client.dispatcher().enqueue(new AsyncCall(responseCallback));
}
```
上述代码是 `RealCall` 对象中的同步和异步的网络请求方法，分析代码可以知道不管是同步请求还是异步请求，每个 `RealCall` 对象只能执行一次，而且它们都调用了 `OkHttpClient` 中的 `dispatcher` 对象来管理`RealCall` ，因此接下来我们就来分析下 `Dispatcher` 这个对象;

### Dispatcher 分发器
上述的同步请求和异步请求都是调用了 `client.dispatcher();` 这个方法来获取 `Dispatcher` 对象，显然这个对象是来自我们的 `OkHttpClient` 类中，通过分析源码可以知道`OkHttpClient` 对象中的 `Dispatcher` 对象是来自其静态内部类 `Builder` 的构造方法中创建的；
```Java
// OkHttpClient.Builder 类
public Builder() {
    dispatcher = new Dispatcher();
    ...
}

// OkHttpClient 类
OkHttpClient(Builder builder) { 
    this.dispatcher = builder.dispatcher;
}
```
找到了 `Dispatcher` 对象的来源，我们分析下 `Dispatcher` 中对同步请求和异步请求的管理；

##### Dispatcher 同步请求管理
我们回到上面 `RealCall` 对象中的同步网络请求，可以发现其调用了 `Dispatcher` 类中对应的 `execute()` 方法，同时将自己作为参数传入到 `Dispatcher` 中，跟进源码可以发现 `Dispatcher` 类对同步网络请求的管理非常简单，只是将请求的 `RealCall` 对象放入到一个 `ArrayDeque` 队列中，`ArrayDeque` 是一个双端队列，内部使用数组进行元素存储，不允许存储 null 值，作为栈使用时，性能比 `Stack` 好，作为队列使用时性能比 `LinkedList` 好，关于 `ArrayDeque` 想深入了解的可以自己找资料，这里就介绍到这里；
```Java
// RealCall 类中的 execute() 方法
client.dispatcher().executed(this);

// Dispatcher 类
synchronized void executed(RealCall call) {
    // 将请求的 RealCall 对象放入到同步请求队列
    runningSyncCalls.add(call);
}
```
上面的代码执行完毕后将会回到 `RealCall` 中的同步方法中调用 `Response result = getResponseWithInterceptorChain();` 调用拦截器执行网络请求拿到响应的 `Response` 对象，关于拦截器我们将在后面进行分析，这里我们只要记住它是调用了这个方法执行了网咯请求且拿到响应的数据即可；
在同步网络请求的最后我们可以看到有一段 `finally` 的代码，根据上面的分析我们可以猜测其作用肯定是将同步请求的 `RealCall` 对象中同步请求队列中移除；
```Java
 // RealCall 类中的 execute() 方法
 client.dispatcher().finished(this);
 
 // Dispatcher 类
 void finished(RealCall call) {
    // 同步请求默认传入
    // param1: 传入同步请求的请求队列
    // param2: 请求的 RealCall 对象
    // param3: 是否调度请求队列这里同步请求默认传入 false
    finished(runningSyncCalls, call, false);
}

 // Dispatcher 类
private <T> void finished(Deque<T> calls, T call, boolean promoteCalls) {
    int runningCallsCount;
    Runnable idleCallback;
    synchronized (this) {
        // 将请求从对应的队列中移除
        if (!calls.remove(call)) 
            throw new AssertionError("Call wasn't in-flight!");
        // 这里同步请求默认传入 false， 异步请求才会进来调度请求队列
        if (promoteCalls) 
            promoteCalls();
        // 重新计算正在请求的数量（同步请求数量 + 异步请求数量）
        runningCallsCount = runningCallsCount();
        idleCallback = this.idleCallback;
    }

    if (runningCallsCount == 0 && idleCallback != null) {
        idleCallback.run();
    }
}
```
到这里整个同步请求就执行完毕了，上面我们分析了`OkHttp` 对同步请求的管理过程，这里我们简单的对同步请求做一个整理:
1. `OkHttp` 在初始化 `OkHttpClient` 网络请求客户端对象的时候会创建一个 `Dispatcher` 对象；
2. 执行同步请求的时候，`OkHttpClient` 会将请求的 `RealCall` 对象缓存到其内部 `Dispatcher` 对象里面的一个同步请求队列中；
3. 执行网络请求（这个模块后面分析）；
4. 请求执行完毕，将请求的 `RealCall` 对象从 `Dispatcher`  的同步请求队列中移除；
5. 同步网络请求没有开辟新的线程来执行网络请求，因此执行同步网络请求会阻塞当前线程，而Android开发中需要自己开辟一个子线程来执行同步网络请求，否则会抛出异常；

##### Dispatcher 异步请求管理
上面分析了 `OkHttp` 对同步网络请求的管理，同步请求的管理还是相对简单，解析来我们将重点分析异步网络请求，这也是我们用的最多的，关于异步请求管理首先我们能想到：线程池、请求队列，而 `OkHttp` 也是这么做的，下面我们来验证我们的猜测：
```Java
// 调用了 Dispatcher 的 enqueue 方法
// 将回调的 Callback 对象封装成了一个 Runnable 对象传入
client.dispatcher().enqueue(new AsyncCall(responseCallback));
```
上面的这段代码是 `RealCall` 对象中异步网络请求的最后一行代码，它首先将监听异步网络请求的 `Callback`  对象封装成了一个 `AsyncCall` 对象，显然这是一个 `Runnable` 类型的对象，它的父类实现了 `Runnable` 接口；
在封装了 `Callback` 对象后调用了 `Dispatcher`  的 `enqueue()` 方法，关于 `Dispatcher` 在同步请求中已经分析过了，我们跟进源码:
```Java
// Dispatcher 类
synchronized void enqueue(AsyncCall call) {
    // 判断当前异步请求的数量是否超过阈值，默认： maxRequests = 64
    // 判断同一个主机的请求数量是否超过阈值，默认:  maxRequestsPerHost = 5
    if (runningAsyncCalls.size() < maxRequests && runningCallsForHost(call) < maxRequestsPerHost) {
        // 上面的条件都满足，将请求加入到异步请求的请求队列中
        runningAsyncCalls.add(call);
        // 调用线程池，执行 Runnable 对象
        // 这里的线程池核心线程数量=0，最大线程数量为 Integer.MAX_VALUE
        executorService().execute(call);
    } else {
        // 上面的条件有超出，将请求加入到异步请求的就绪队列中
        readyAsyncCalls.add(call);
    }
}
```
从上面的代码我们可以知道，`Dispatcher` 类内部维护了两个 `ArrayDeque` 对象 `runningAsyncCalls` 和 `readyAsyncCalls` 从名字中就可以知道一个是运行中的异步请求队列，一个是就绪的的异步请求队列，上面我们分析同步请求时 `Dispatcher` 内部只有一个同步请求队列，这里主要的原因是，同步请求会按照队列的顺序一个个顺序执行，而异步请求会调用线程池，`Dispatcher` 类中的线程池并没有设定线程数量上线，如果请求数量非常的密集，线程池中创建了太多的线程反而会影响性能，因此如果请求过多可以额外增加一个等待请求的队列将其缓存起来，而缓存到就绪队列的条件在代码的注释中已经写明；
异步请求最终是调用了 `Dispatcher` 类里面的线程池来执行，传入的 `Runnable` 对象就是 `RealCall` 异步请求方法中封装的 `AsyncCall` 对象 ，其 `run()` 由其父类实现:
```Java
// NamedRunnable 类
@Override public final void run() {
    String oldName = Thread.currentThread().getName();
    Thread.currentThread().setName(name);
    try {
        // 调用了 execute() 方法，该方法是抽象的，他的实现在 AsyncCall 中
        execute();
    } finally {
        Thread.currentThread().setName(oldName);
    }
}

// 抽象方法
protected abstract void execute();
```
上面的 `run()` 方法很简单，其主要的逻辑都是都是在 `execute()` 抽象方法中实现的，我们转到 `AsyncCall` 中的 `execute()` 方法：
```Java
@Override protected void execute() {
  boolean signalledCallback = false;
  try {
    // 调用拦截器执行网络请求，这里和同步请求一样也是调用了这个方法，我们在后面分析
    Response response = getResponseWithInterceptorChain();
    if (retryAndFollowUpInterceptor.isCanceled()) {
      // 请求和重定向拦截器判断当前请求是否失败，onFailure 方法
      signalledCallback = true;
      responseCallback.onFailure(RealCall.this, new IOException("Canceled"));
    } else {
      // 请求成功，回调 onResponse 方法
      signalledCallback = true;
      responseCallback.onResponse(RealCall.this, response);
    }
  } catch (IOException e) {
    ...
  } finally {
    // 请求执行完毕，调用 dispatcher 对当前请求的 RealCall 移出请求队列
    client.dispatcher().finished(this);
  }
}
```
上面的代码就是异步网络请求的及回调的过程，其实现和同步网络请求非常的相似，只是增加了请求之后的回调，而这里的 `execute()` 方法是在线程池中的子线程中执行的，因此这里的回调即`OkHttp` 异步网络请求的回调方法执行实在子线程中执行的；
同样，在异步请求的最后也调用了 `Dispatcher` 的 `finished()` 方法对请求完毕的 `RealCall` 对象进行管理，只不过这里调用的是其重载的方法，参数是 `AsyncCall` 对象:
```Java
// Dispatcher 类
void finished(AsyncCall call) {
    // 异步网络请求的请求管理方法，
    // param1: 传入异步请求的请求队列
    // param2: 当前请求的 RealCall 对象
    // param3: 是否调度请求队列这里异步请求默认传入 true
    finished(runningAsyncCalls, call, true);
}

// 这个方法和同步请求队列中的队列管理方法是一样的
private <T> void finished(Deque<T> calls, T call, boolean promoteCalls) {
    int runningCallsCount;
    Runnable idleCallback;
    synchronized (this) {
        // 将请求从对应的队列中移除
        if (!calls.remove(call)) 
            throw new AssertionError("Call wasn't in-flight!");
        // 这里异步请求传入的是 true
        if (promoteCalls) 
            // 调度队列
            promoteCalls();
        // 重新计算正在请求的数量（同步请求数量 + 异步请求数量）
        runningCallsCount = runningCallsCount();
        idleCallback = this.idleCallback;
    }

    if (runningCallsCount == 0 && idleCallback != null) {
        idleCallback.run();
    }
}

// 调度请求队列，执行异步网络请求时才会调用
private void promoteCalls() {
    // 正在执行的异步请求是否超过阈值，默认: maxRequests = 64
    if (runningAsyncCalls.size() >= maxRequests) 
        return;
    // 异步请求的就绪队列是否为空
    if (readyAsyncCalls.isEmpty()) 
        return;
    // 迭代异步请求的就绪队列，获取满足条件的请求然后执行
    for (Iterator<AsyncCall> i = readyAsyncCalls.iterator(); i.hasNext(); ) {
        AsyncCall call = i.next();
        // 判断当前正在执行的异步请求统一主机的请求数量是否超过阈值，默认: maxRequestsPerHost = 5
        if (runningCallsForHost(call) < maxRequestsPerHost) {
            // 满足条件，将请求从就绪队列中移除
            i.remove();
            // 将请求加入到异步请求的请求队列中
            runningAsyncCalls.add(call);
            // 调用线程池执行请求
            executorService().execute(call);
        }
        // 如果请求的最大数量超过了阈值，跳出循环，等待其它请求完成时再来调用该方法
        if (runningAsyncCalls.size() >= maxRequests) 
            return; 
    }
}
```
异步网络请求的队列管理对比同步网络请求的队列管理就复杂的多，虽然异步请求的线程池没有设置线程数量上限，但是其内部设置了一个最大请求数量的阈值（maxRequests = 64），即同时执行的异步网络请求不能超过 64 个，同时同一主机同时请求的数量不能超过 5 个，如果这两个请求有一个条件超出，`OkHttp` 就会将此次请求加入到异步请求的就绪队列中，如果都不超出则将请求加入到异步请求的执行队列中，每个异步请求执行完毕后除了将当前请求的 `RealCall` 对象从异步请求的执行队列中移除外，还会调用 `promoteCalls()`  方法，去异步请求的就绪队列中调度满足执行条件的异步请求；
到这里 `OkHttp` 的异步网络请求对象管理就分析完毕了，也验证了我们上面说的，异步请求中使用到了线程池和请求队列，而且 `OkHttp` 还管理了两个异步请求队列，我们对其做一下整理:
1. `OkHttp` 执行异步网络请求的时候，会将监听回调的 `Callback` 对象封装成一个 `Runnable` 对象；
2. `Dispatcher` 类内部有两个 `ArrayDeque` 队列，分别是正在执行的异步网络请求队列和就绪的异步网络请求队列，需要执行的异步网络请求加入哪个队列的依据是: 最大请求数量是否超过阈值（maxRequests = 64）,同一主机正在执行异步网络请求数量是否超过阈值（maxRequestsPerHost = 5）;
3. `Dispatcher` 内部维护了一个线程池，用来执行异步网络请求，其最大线程数量是 Integer.MAX_VALUE，但是由于上面第二点中的最大请求数量的限制，因此默认情况下，`OkHttp` 同时执行的异步网络请求数量将不会超过 64;
4. 异步网络请求的主要逻辑在 `AsyncCall` 中复写的 `execute()` 方法中，其真正的网络请求和同步网络请求是一样的，这个在后面的模块中分析；
5. 异步网络请求的回调方法是在执行请求的线程中调用的，异步网络请求的回调函数如果有 UI 操作需要切换线程；
6. 异步网络请求执行完毕后，`Dispatcher` 会将其从正在执行的异步请求队列中移除，同时去调用就绪的异步请求队列，是否有满足条件的请求，如果有满足，将其从就绪队列中移除然后加入到执行队列中；

关于 `OkHttp` 对同步同步请求和异步请求的管理到这里就分析完毕了，整个逻辑相对来说还是比较清晰的，其重点在于 `Dispatcher` 类中对同步请求和异步请求的队列管理，`Dispatcher` 内部还维护了一个线程池用于执行异步网络请求，下图是我对 `Dispatcher` 管理请求绘制的流程图:
![Dispatcher请求管理](https://upload-images.jianshu.io/upload_images/7082912-029051c6d0b0ba80.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


### Interceptor 拦截器
上面分析了 `OkHttp` 中对同步请求和异步请求的管理，我们发现不管是同步请求还是异步请求，都执行了`getResponseWithInterceptorChain()`来获取响应 `Response` 对象，接下来我们将重点分析该方法；
```Java
// RealCall 类
Response getResponseWithInterceptorChain() throws IOException {
    // 创建了一个List集合，用来缓存所有的拦截器
    List<Interceptor> interceptors = new ArrayList<>();
    // 首先加入的是自定义拦截器
    interceptors.addAll(client.interceptors());
    
    // 这里开始加入 OkHttp 为我们提供的拦截器
    // 重定向拦截器
    interceptors.add(retryAndFollowUpInterceptor);
    // 桥接拦截器
    interceptors.add(new BridgeInterceptor(client.cookieJar()));
    // 缓存拦截器
    interceptors.add(new CacheInterceptor(client.internalCache()));
    // 网络连接拦截器
    interceptors.add(new ConnectInterceptor(client));
    // 判断是否需要加入网络拦截器
    if (!forWebSocket) {
        interceptors.addAll(client.networkInterceptors());
    }
    // 数据访问拦截器
    interceptors.add(new CallServerInterceptor(forWebSocket));
 

    // 创建拦截器链，RealInterceptroChain 会去获取上述 interceptores 指定下标位置的拦截器
    // 然后在 proceed() 方法中创建下一个拦截器的 
    Interceptor.Chain chain = new RealInterceptorChain(interceptors, null, null, null, 0,
        originalRequest, this, eventListener, client.connectTimeoutMillis(),
        client.readTimeoutMillis(), client.writeTimeoutMillis());
    // 执行拦截器
    return chain.proceed(originalRequest);
}

```
上面的代码主要完成了三个功能：
1. 添加自定义拦截器；
2. 添加`OkHttp` 提供的拦截器；
3. 创建 `RealInterceptorChain` 对象并调用其 `proceed()` 方法；

首先这里的拦截器不管是自定义的还是 `OkHttp` 提供的都需要实现 `Interceptor` 接口，查看其源码可以知道 `Interceptor` 接口内部只有一个方法；
```Java
public interface Interceptor{
    Response interceptor(Chain chain) throws IOException;
    
    // 后续的拦截器链对象需要实现此接口
    interface Chain{
        ...
    }
}
```
了解了 `Interceptor` 接口后，我们来分析下 `RealInterceptorChain` 类，看看它是如何完成所有的拦截器的调用:
```Java
// RealInterceptorChain 类
// 上述代码的 proceed() 方法最终会执行到这里
public Response proceed(Request request, StreamAllocation streamAllocation, HttpCodec httpCodec,
      RealConnection connection) throws IOException {
      // index 即上述拦截器集合中的下标，
      // getResponseWithInterceptorChain() 方法中传入的 index = 0
      // 因此拦截器的执行是从第 0 个位置开始的，也就是自定义的拦截器会优先于系统的执行
    if (index >= interceptors.size()) 
        throw new AssertionError();
    ...
    // 创建下一个拦截器链对象，这里需要注意的是拦截器的下标变成了 index + 1
    RealInterceptorChain next = new RealInterceptorChain(interceptors, streamAllocation, httpCodec,
        connection, index + 1, request, call, eventListener, connectTimeout, readTimeout,
        writeTimeout);
    // 获取当前拦截器链对应位置的拦截器
    Interceptor interceptor = interceptors.get(index);
    // 调用拦截器复写的 intercept() 方法，并将下一个拦截器链传入
    // 这里只需要在拦截器的 intercept() 方法中调用传入的拦截器链的 proceed() 方法
    // 就可以遍历执行完集合中的所有拦截器
    Response response = interceptor.intercept(next);
    ...
    return response;
}
```
整个拦截器的执行流程通过拦截器链来完成调用，下图是根据个人理解绘制的拦截器执行流程:
![OkHttp拦截器执行流程](https://upload-images.jianshu.io/upload_images/7082912-655bcb78d08b3313.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
其整体的执行流程有点类似于 `View` 事件的分发机制，而每个拦截器就可以看作是一个`ViewGroup` 可以对传入的事件（请求）进行处理（向下传递/拦截消费）；
`OkHttp` 拦截器的优势在于每个拦截器都只需要处理自己模块的逻辑即可，把整个网络请求的过程层次化，例如登录账户时，后台会返回一个 `Token` 字符串，常规的逻辑可以是在请求成功之后解析对应的返回值来获取 `Token` ，但是如果你使用了 `OkHttp` 你就可以使用更加优雅的方式 `Interceptor` 来提前完成 `Token` 的解析和缓存，而且你并不需要关心哪些位置需要解析 `Token` ；
以上就是 `OkHttp` 拦截器的执行流程，我们看到系统默认提供了几个拦截器，关于默认的拦截器我将在后续的文章中进行分析；