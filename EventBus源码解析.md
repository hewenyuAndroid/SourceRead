### 简介
在使用事件总线的时候，我们通常是通过其单例对象来进行操作的，其单例对象是通过 DCL 实现的；本文将从源码的角度来分析`EventBus` 的实现原理；

### register()
通过单例对象调用 `register()` 方法即可订阅消息，它的入参是一个 `Object` 对象，也就是说任何对象都可以订阅消息，`register()` 方法的代码如下：
```Java
public void register(Object subscriber) {
    // 获取 subscriber(订阅者) 的 Class 对象
    Class<?> subscriberClass = subscriber.getClass();
    // SubscriberMethod 通过 Class 对象获取 subscriber 中订阅的方法
    List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);
    synchronized (this) {
        for (SubscriberMethod subscriberMethod : subscriberMethods) {
            // 订阅操作
            subscribe(subscriber, subscriberMethod);
        }
    }
}
```
`register()` 方法中主要做了两件事: 
1. 解析订阅对象中的订阅方法；
2. 执行订阅操作；

##### 1. 解析订阅对象中的订阅方法
```Java
// SubscriberMethodFinder 类
List<SubscriberMethod> findSubscriberMethods(Class<?> subscriberClass) {
    // 这里传入的参数即为上面提到的 订阅者 的 Class 对象
    // 这里做了一个缓存，提升性能（一般我们都是通过反射去解析方法的）
    List<SubscriberMethod> subscriberMethods = METHOD_CACHE.get(subscriberClass);
    if (subscriberMethods != null) {
        return subscriberMethods;
    }
    // ignoreGeneratedIndex 值来自 Eventbus 创建时候的 DEFAULT_BUILDER 对象，
    // DEFAULT_BUILDER 对象中的 ignoreGeneratedIndex 默认是 false;
    if (ignoreGeneratedIndex) {
        // 如果是使用注解生成器生成代码这回执行到这里（类似ButterKnife）
        subscriberMethods = findUsingReflection(subscriberClass);
    } else {
        // 默认情况下会执行到这里，通过反射来解析订阅的方法
        subscriberMethods = findUsingInfo(subscriberClass);
    }
    // 如果订阅对象没有 订阅方法 即没有 @Subscribe() 注解标记的方法就会抛出异常
    if (subscriberMethods.isEmpty()) {
        throw new EventBusException("Subscriber " + subscriberClass
                + " and its super classes have no public methods with the @Subscribe annotation");
    } else {
        // 加入缓存中，方便下次调用
        METHOD_CACHE.put(subscriberClass, subscriberMethods);
        return subscriberMethods;
    }
}
```
`findSubscriberMethods()` 方法首先会去缓存中判断之前是否有解析过该类型的 `subscriber`(订阅者) 对象，用于提升性能；

如果是第一次解析该类型，则会继续往下执行，这里会有个`ignoreGeneratedIndex` 判断，该值来自 `Eventbus` 单例对象创建时的 `DEFAULT_BUILDER` 对象中默认是 `false` 表示需要通过反射解析 `subscriber` 中订阅的方法，如果为 `true` 则是通过 `apt` 在编译获取（类似 `ButterKnife`），这里我们只分析我们用的比较多的 反射解析；
```Java
// SubscriberMethodFinder 类
private List<SubscriberMethod> findUsingInfo(Class<?> subscriberClass) {
    // 从 FIND_STATE_POOL 池子里面获取 FindState 对象
    FindState findState = prepareFindState();
    // 初始化 findState 对象
    findState.initForSubscriber(subscriberClass);
    while (findState.clazz != null) {   // findState.clazz 来自上一行代码的初始化
        findState.subscriberInfo = getSubscriberInfo(findState);
        if (findState.subscriberInfo != null) { // 这里返回默认为 null 执行 else
           ...
        } else {
            // 这里即为反射解析 subscriber 对象的实际位置
            findUsingReflectionInSingleClass(findState);
        }
        findState.moveToSuperclass();
    }
    // 这里将 findUsingReflectionInSingleClass() 中解析的 SubscriberMethod 列表重新进行封装
    // 并对 findState 对象进行回收；
    return getMethodsAndRelease(findState);
}

private void findUsingReflectionInSingleClass(FindState findState) {
    Method[] methods;
    try {
        // 反射获取方法数组
        methods = findState.clazz.getDeclaredMethods();
    } catch (Throwable th) {
    }
    for (Method method : methods) {
        // 获取方法的修饰符
        int modifiers = method.getModifiers();
        // 判断方法的修饰符
        if ((modifiers & Modifier.PUBLIC) != 0 && (modifiers & MODIFIERS_IGNORE) == 0) {
            // 获取订阅方法的入参列表
            Class<?>[] parameterTypes = method.getParameterTypes();
            if (parameterTypes.length == 1) {
                // 获取 Subscribe 注解
                Subscribe subscribeAnnotation = method.getAnnotation(Subscribe.class);
                if (subscribeAnnotation != null) {
                    // 获取事件类型（方法参数对象的类型）
                    Class<?> eventType = parameterTypes[0];
                    if (findState.checkAdd(method, eventType)) {
                        // 获取注解中执行的 ThreadMode
                        ThreadMode threadMode = subscribeAnnotation.threadMode();
                        // 订阅方法中解析的参数封装到 SubscriberMethod 对象中并将其放入到 FindState 对象中
                        findState.subscriberMethods.add(new SubscriberMethod(method, eventType, threadMode,
                                subscribeAnnotation.priority(), subscribeAnnotation.sticky()));
                    }
                }
            } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
               // 如果 Subscribe 注解标记的方法的参数数量 != 1 ，就会抛出异常
            }
        } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
            // 这里如果是非 public 修饰的方法使用了 Subscribe 注解，就会抛出异常
        }
    }
}
```
上述代码主要使用了反射来解析 `subscriber` 对象中订阅的方法，然后将其封装到 `SubscriberMethod` 对象并暂存到 `findState` 对象中，最后会将 `findState` 对象中缓存的 `SubscriberMethod` 列表重新封装返回同时回收 `findState` 对象；

##### 执行订阅操作
上面我们分析了 `subscriber` 对象是如何解析并返回 `SubscriberMethod` 列表的，接下来我们看下 `Eventbus` 是如何完成订阅的:
```Java
// Eventbus 类

// 集合1：根据事件类型缓存的集合，方便发送事件
private final Map<Class<?>, CopyOnWriteArrayList<Subscription>> subscriptionsByEventType;
// 集合2：根据订阅对象缓存的集合，方便订阅对象解绑订阅
private final Map<Object, List<Class<?>>> typesBySubscriber;
// 集合3：缓存粘性事件
private final Map<Class<?>, Object> stickyEvents;

private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {
    // 获取事件对象的 Class ，这里的 eventType 即为上面分析的 订阅方法的 参数
    Class<?> eventType = subscriberMethod.eventType;
    // 将 subscriber 和 我们解析的 subscriberMethod 对象再一次封装
    Subscription newSubscription = new Subscription(subscriber, subscriberMethod);
    // 获取订阅了该事件的方法对象的集合，这样有利于向所有的订阅方法发送事件
    CopyOnWriteArrayList<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
    if (subscriptions == null) {
        subscriptions = new CopyOnWriteArrayList<>();
        subscriptionsByEventType.put(eventType, subscriptions);
    } else {
    }
    // 缓存订阅的方法，这里会根据 Subscriber 注解中指定的优先级进行排序，将优先级大的方法会放在前面
    int size = subscriptions.size();
    for (int i = 0; i <= size; i++) {
        if (i == size || subscriberMethod.priority > subscriptions.get(i).subscriberMethod.priority) {
            subscriptions.add(i, newSubscription);
            break;
        }
    }

    // 根据订阅对象获取缓存的事件列表，该列表缓存的作用主要是用于解绑订阅
    List<Class<?>> subscribedEvents = typesBySubscriber.get(subscriber);
    if (subscribedEvents == null) {
        subscribedEvents = new ArrayList<>();
        typesBySubscriber.put(subscriber, subscribedEvents);
    }
    subscribedEvents.add(eventType);
    
    // 判断是否是粘性事件，这里再 Subscriber 注解中指定，后面会进行分析
    if (subscriberMethod.sticky) {
        if (eventInheritance) {
            Set<Map.Entry<Class<?>, Object>> entries = stickyEvents.entrySet();
            for (Map.Entry<Class<?>, Object> entry : entries) {
                Class<?> candidateEventType = entry.getKey();
                if (eventType.isAssignableFrom(candidateEventType)) {
                    Object stickyEvent = entry.getValue();
                    // 粘性事件会在注册完成的时候向订阅的对象发送通知
                    checkPostStickyEventToSubscription(newSubscription, stickyEvent);
                }
            }
        } else {
            Object stickyEvent = stickyEvents.get(eventType);
            checkPostStickyEventToSubscription(newSubscription, stickyEvent);
        }
    }
}
```
通过上面的代码我们可以知道，整个订阅的过程中，有三个 Map 集合作为缓存非常关键：
**1. subscriptionsByEventType**
key：事件类型的 `Class` 对象，这里的事件类型即上述分析的订阅方法的参数类型；
value：`CopyOnWriteArrayList<Subscription>` 集合对象，`Subscription` 为 `Subscriber`  和 `SubscriberMethod` 两个对象的封装；
`subscriptionsByEventType` 的作用在于发送事件时，可以通过事件类型拿到所有订阅该事件的方法然后发送事件；

**2. typesBySubscriber**
key：`subscriber` 对象，即订阅对象；
value：`List<Class<?>>` 集合对象，缓存的是该订阅对象中订阅的所有的事件对象的类型集合；
`typesBySubscriber` 的作用在于，解绑订阅时，可以通过对象快速的找到需要解绑的事件类型，通过事件类型快速的定位到 `subscriptionsByEventType` 集合中需要解绑的位置；

**3. stickyEvents**
key：事件类型的`Class`对象，这里的事件类型即上述分析的订阅方法的参数类型；
value：粘性事件的事件对象；
`stickyEvents` 粘性事件的原理其实就是将粘性事件缓存到该集合中，如果后续订阅的方法支持粘性事件，同时`stickyEvents` 集合中存在该事件类型的对象，此时就会将该事件发送给订阅的方法；

### unregister()
分析完了注册订阅，我们来看下 `Eventbus` 是如何实现解绑的，关于解绑上面我们也提到过，先来看一下它的源码：
```Java
// Eventbus 类

public synchronized void unregister(Object subscriber) {
    // 获取 subscriber 对象中所有的订阅的事件类型
    // 这里 typesBySubscriber 集合我们在上面有分析过其作用；
    List<Class<?>> subscribedTypes = typesBySubscriber.get(subscriber);
    if (subscribedTypes != null) {
        for (Class<?> eventType : subscribedTypes) {
            // 根据订阅的事件类型解绑
            unsubscribeByEventType(subscriber, eventType);
        }
        typesBySubscriber.remove(subscriber);
    } else {
        ...
    }
}
```
解绑的代码还是比较简单的，主要就是通过 `typesBySubscriber` 集合中获取当前对象所有的事件类型，然后再到 `subscriptionsByEventType` 集合中将该订阅对象移除；

### Subscribe
`Subscribe` 是一个注解对象用来标记订阅的方法，该注解主要有三个作用：
1. 指定线程的模型，关于线程模型我们再下一个模块进行分析；
2. 设置是否接收粘性事件，粘性事件是什么就不用多BB了吧；
3. 设置接收事件的优先级，优先级大的订阅方法先收到事件；
```Java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.METHOD})
public @interface Subscribe {
    // 线程模型，默认使用当前发送的线程
    ThreadMode threadMode() default ThreadMode.POSTING;
    // 是否接收粘性事件
    boolean sticky() default false;
    // 事件的优先级，优先级大的先发送
    int priority() default 0;
}
```

### post()/postSticky()
前面分析了一大堆，就是为了我们发送事件做铺垫，前面我们有分析过一个 Map 集合 `subscriptionsByEventType` ，它的作用就是来帮助我们完成事件的发送，接下来我们就来分析下事件发送是如何完成的：
```Java
// Eventbus 类

// 发送粘性事件
public void postSticky(Object event) {
    synchronized (stickyEvents) 
        // 将事件缓存到 stickyEvents 集合中
        // 这个集合我们在将 register 的时候分析过
        stickyEvents.put(event.getClass(), event);
    }
    // 最终还是调用了 post() 方法
    post(event);
}

// 发送事件
public void post(Object event) {
    // 获取事件队列和线程状态信息
    PostingThreadState postingState = currentPostingThreadState.get();
    List<Object> eventQueue = postingState.eventQueue;
    eventQueue.add(event);

    if (!postingState.isPosting) {
        // 判断是否是主线程
        postingState.isMainThread = isMainThread();
        postingState.isPosting = true;
        try {
            // 这里开启循环发送队列中的所有消息
            while (!eventQueue.isEmpty()) {
                // 发送事件
                postSingleEvent(eventQueue.remove(0), postingState);
            }
        } finally {
        }
    }
}

private void postSingleEvent(Object event, PostingThreadState postingState) throws Error {
    // 获取事件的类型
    Class<?> eventClass = event.getClass();
    boolean subscriptionFound = false;
    // 事件是否允许继承
    // 该变量的值来自于 Builder 默认 = true
    // 可以在创建 EventBus 对象的 时候通过其 Builder 对象来配置
    if (eventInheritance) {
        // 递归遍历事件类型所实现的所有接口类型，继承的父类行及其实现的接口类型
        // 只要发送了该事件，只要是订阅了上面遍历出来的类型的事件的方法，都会接收到该事件
        List<Class<?>> eventTypes = lookupAllEventTypes(eventClass);
        int countTypes = eventTypes.size();
        for (int h = 0; h < countTypes; h++) {
            Class<?> clazz = eventTypes.get(h);
            // 收到事件的顺序
            // 当前事件-->当前事件的接口类型事件-->当前事件的父类行事件-->...
            subscriptionFound |= postSingleEventForEventType(event, postingState, clazz);
        }
    } else {
        subscriptionFound = postSingleEventForEventType(event, postingState, eventClass);
    }
    // 没有找到订阅该事件的对象
    if (!subscriptionFound) {
        ...
        if (sendNoSubscriberEvent && eventClass != NoSubscriberEvent.class &&
                eventClass != SubscriberExceptionEvent.class) {
            post(new NoSubscriberEvent(this, event));
        }
    }
}


private boolean postSingleEventForEventType(Object event, PostingThreadState postingState, Class<?> eventClass) {
    CopyOnWriteArrayList<Subscription> subscriptions;
    synchronized (this) {
        // 从我们上面分析的第一个 Map 集合中，根据事件类型获取订阅了该事件类型的订阅方法列表
        subscriptions = subscriptionsByEventType.get(eventClass);
    }
    if (subscriptions != null && !subscriptions.isEmpty()) {
        // 遍历订阅的方法列表，该列表是根据优先级排序了的
        for (Subscription subscription : subscriptions) {
            postingState.event = event;
            postingState.subscription = subscription;
            boolean aborted = false;
            try {
                // 执行订阅的方法，该方法内部会根据指定的线程模型来判断在哪个线程执行方法
                postToSubscription(subscription, event, postingState.isMainThread);
                // 在每个订阅方法执行完之后判断是否拦截此事件
                // 调用  EventBus.getDefault().cancelEventDelivery(event) ; 即可拦截事件
                aborted = postingState.canceled;
            } finally {
                postingState.event = null;
                postingState.subscription = null;
                postingState.canceled = false;
            }
            // 如果事件已经拦截，后面的订阅者将不会接收到消息
            if (aborted) {
                break;
            }
        }
        return true;
    }
    return false;
}

// 该方法为真正执行订阅方法回调的地方，通过反射来执行订阅的方法
private void postToSubscription(Subscription subscription, Object event, boolean isMainThread) {
    switch (subscription.subscriberMethod.threadMode) {
        case POSTING:  // 事件发送和订阅方法执行在同一个线程
            invokeSubscriber(subscription, event);
            break;
        case MAIN:  // 订阅的方法执行在主线程
            if (isMainThread) {
                invokeSubscriber(subscription, event);
            } else {
                // 这里如果是子线程发送的事件，需要在主线程中执行
                // 会通过 Handler 转到主线程中执行
                // mainThreadPoster 对象其实是 Handler 的子类
                mainThreadPoster.enqueue(subscription, event);
            }
            break;
        case MAIN_ORDERED:  // 订阅的方法执行在主线程，这里的事件会先进入队列
            if (mainThreadPoster != null) {
                mainThreadPoster.enqueue(subscription, event);
            } else {
                // temporary: technically not correct as poster not decoupled from subscriber
                invokeSubscriber(subscription, event);
            }
            break;
        case BACKGROUND:    // 子线程中执行订阅方法
            if (isMainThread) {
                // 这里会通过线程池来执行
                backgroundPoster.enqueue(subscription, event);
            } else {
                invokeSubscriber(subscription, event);
            }
            break;
        case ASYNC: 
             // 子线程中执行订阅方法
             // 这里和 BACKGROUND 不同的是，即使发送事件是在子线程，
             // 也会开辟一个新的线程来执行订阅方法
            asyncPoster.enqueue(subscription, event);
            break;
        default:
            throw new IllegalStateException("Unknown thread mode: " + subscription.subscriberMethod.threadMode);
    }
}
```
整个事件发送的流程细化下来也就是这几个方法，下面我们来对其进行归纳：
1. 发送粘性事件实际上就是比发送事件多了一步将事件存入粘性事件集合（这也验证了我们上面对粘性事件集合的分析）；
2. 默认的情况下发送的事件是支持继承的，即：订阅了事件父类型、事件实现的接口类型都会收到当前事件；
3. 拦截事件的传递可以调用 `EventBus.getDefault().cancelEventDelivery(Event)`  方法，该方法的调用有一定的局限：
a). 传入的事件只能取消正在执行发送的事件，即订阅方法中只能调用其方法参数的事件，且不能传入null;
b). 只能拦截 `ThreadMode.POSTING` 模型的中的事件；
c). 只能拦截一类事件，而且如果是上述第二点中的父类行的事件，调用此方法拦截也无效；
d). 个人建议调用拦截事件的方法可以使用 try/catch 捕获异常，防止应用崩溃；
4. `EventBus` 会根据订阅方法中指定的线程模型来指定订阅方法在哪个线程中执行，具体的线程模型详见代码中的注释；