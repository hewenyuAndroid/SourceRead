### д��ǰ��
���Ϲ��� `Handler` �������кܶ���Щд��Ҳ�ǱȽϺõģ�������Ҫ��Ϊ�˼�¼�Լ��Ķ�Դ���������ᣬ�����Լ���Դ�����⣬�ǵú����ʱ���Լ�Ҳ���� `Handler` ��Դ�룬��ʱû�����ʼ���Щ�ط��ǵ�Ҳ���Ǻ����������Ժ��Դ���Ķ���ö������Լ��ܽ��һƪ���»�ȽϺã�

### ���� Handler ��ʹ�ó���
���� ` Handler` �������ǿ϶�����İ�������������߳��õ����ݣ�ͨ�� `Handler` ֪ͨ���߳�֪ͨUI�Ϳ���ʹ�� `Handler` ��������������֪ͨ�Ĺ��ܣ��ٷ��� `Handler` ��˵��  [Handler�ĵ�](https://developer.android.google.cn/reference/android/os/Handler) ����Ȥ�Ŀ���ȥ���£��������ǿ����ҵ��ٷ��� `Handler` �������Ҫ��;:
> (1) to schedule messages and runnables to be executed at some point in the future
> (2) to enqueue an action to be performed on a different thread than your own.

��ŵ���˼���ǵ�һ�������ǵ� Message/Runnable ��������ڽ�����ĳ��ʱ��ִ�У��ڶ����ǽ���ͬ�߳��еĲ���������У�
���� `Handler` ����Ϣ�������ǽ�����������:`Handler` ��`Looper`��`MessageQueue`��`Message`��

### Handler ����Ĵ���
���� `Handler` �����޷���ͨ�� new ������������ûʲô�ý��ģ���������������Ҫ�������Ĺ��캯��������������⣬`Handler` �����ع��캯���кܶ࣬�����ϸ�۲����ǲ��ѷ��������յ���������Ƿ�����ָ����`Looper` ���󣬹��� `Looper` �����Ȳ�չ�����������ֻ����һ��û��ָ��Ĭ��`Looper` ����Ĺ��죻
```Java
// ���๹�����ն����ߵ����������
public Handler(Callback callback, boolean async) {
    // ʡ�Դ���...
    // 1. ��ȡ��ǰ���� Handler �������ڵ��߳�
    mLooper = Looper.myLooper();
    if (mLooper == null) {
        throw new RuntimeException(
            "Can't create handler inside thread that has not called Looper.prepare()");
    }
    // 2. �����õ��� Looper �����ȡ��Ϣ����
    mQueue = mLooper.mQueue;
    mCallback = callback;
    mAsynchronous = async;
}
```
��������������֪����������ͨ�� new ���� `Handler` �����ʱ��ϵͳ������������Щ���飬��Ҫ���ǻ�ȡһ�� `Looper` �����Լ� `Looper` �����е�һ����Ϣ���У����� `Looper` ����� `MessageQueue` ���������ں����������ﲻ��չ��������ֻ��Ҫ֪�������� `Handler` ������ʱ��ϵͳ�����Ǹ�������������Ϳ��ԣ�

### ������Ϣ
�������Ƿ����� `Handler` ����Ĵ��������󴴽����˽�������Ӧ���Ƿ�����Ϣ�ˣ�ͨ�����Ƿ�����Ϣ�ķ�ʽҲ�кܶ࣬����ʹ�÷�ʽ�Ĳ�ͬ�����ǿ��Խ����ǵ���Ϣ���·�Ϊ�����࣬һ���Ǵ��� `Message` �����͵���Ϣ������һ���Ǵ��� `Runnable` �����͵���Ϣ��������Բ鿴��������ͼƬ:

![sendMessage](https://upload-images.jianshu.io/upload_images/7082912-ead373d5c7edecd0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![post](https://upload-images.jianshu.io/upload_images/7082912-776d837012506344.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### sendMessage()
����� sendMessage ������ֻ���� `sendMessage(Message message)` ����������ָ�����һ��ͼ�е���һ�෽�����������Դ����һ�����Ǿ�֪���� `sendEmptyMessage()` ����ʵ��������ϵͳ������**��װ**�� `Message` �������ն����ߵ����ǵ� `sendMessageAtTime()` ���������ǵ� `Message` ����ŵ���Ϣ������:
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
    // ���ｫ��ǰ������Ϣ�� Handler ���ó�Ϊ��Ϣ����� target
    msg.target = this;
    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
    // ������� MessageQueue �������ӷ���
    return queue.enqueueMessage(msg, uptimeMillis);
}
```

##### post()
ͬ������� post ������ֻ���� `post(Runnable r)` ���������ǿ��Կ����⼸�����صķ���:
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
�����ԣ��÷���Ҳ�ǵ��������ߵ� `sendMessage` ���������ҽ� `Runnable` �����װ���� `Message` ���󣬷�װ���߼�Ҳ�Ƿǳ��ļ�࣬���� `Runnable` ����ֵ���� `Message` �ĳ�Ա������
```Java
private static Message getPostMessage(Runnable r) {
    Message m = Message.obtain();
    m.callback = r;
    return m;
}
```

���������Ƿ����� `Handler` ����Ĵ����Լ�������Ϣ��Դ�룬������Ϣ���ն��ǵ����� `sendMessageAtTime()` ���������
�������Ƿ����� `Handler` �Ĵ����ͷ�����Ϣ��Դ�룬֪���� `Handler` �е� `MessageQueue` �����Ǵӹ����л�ȡ�� `Looper`  �������õ��ģ��������Ƿ��͵���Ϣ�ǵ����� `queue.enqueueMessage(msg, uptimeMillis);` ���д��뽫���ǵ���Ϣ���뵽��Ϣ�����У��������ǵ���Ϣ�ڶ������������������ǽ�����ͨ��Դ����������

### MessageQueue ��Ϣ����
���ǽ��������Դ�������������Ƿ��͵� `Message` ��������ν��д洢�ģ�
```Java
boolean enqueueMessage(Message msg, long when) {
    // ʡ�Դ���...
    synchronized (this) {
        // ʡ�Դ���...
        // �����ǵ���Ϣ��������Ϊʹ����
        msg.markInUse();
        // ������Ϣ���͵�ʱ��
        msg.when = when;
        // ������ʱ����Ϣ����
        Message p = mMessages;
        boolean needWake;
        if (p == null || when == 0 || when < p.when) {
            // New head, wake up the event queue if blocked.
            // ������Ҫ������������ж��߼�
            // 1.  p == null ˵�� mMessage = null��Ҳ�������Ǵ������� Message Ϊ���еĵ�һ����Ϣ��
            // 2. when == 0 ,˵�����Ƿ��͵���Ϣ��Ҫ�������ͣ���Ҫ������е�ͷ����
            // 3. when < p.when ��˵�����ǵ���Ϣ�ķ���ʱ��ȶ�����ĵ�һ����Ϣ��ʱ���磬��Ҫ�����ڶ��е�ͷ����
            msg.next = p;
            mMessages = msg;
            needWake = mBlocked;
        } else {
            // Inserted within the middle of the queue.  Usually we don't have to wake
            // up the event queue unless there is a barrier at the head of the queue
            // and the message is the earliest asynchronous message in the queue.
            needWake = mBlocked && p.target == null && msg.isAsynchronous();
            // ������Կ�����һ�����͵�����ṹ
            Message prev;
            for (;;) {
                prev = p;
                p = p.next;
                if (p == null || when < p.when) {
                    // p == null ͨ��ѭ���ж� prev �Ƿ��Ƕ��е����һ����Ϣ;
                    // when < p.when �ж�������Ҫ������е�������Ϣ��Ҫ���͵�ʱ���Ƿ�
                        //  �� prev �� p ����������ķ���ʱ��֮�䣻
                    break;
                }
                if (needWake && p.isAsynchronous()) {
                    needWake = false;
                }
            }
             // �������������ǵ���Ϣ���� prev �� p ����������֮��
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
����Ĵ�������Ҷ��Ѿ�д��ע�ͣ��������Ƕ������Ĵ�����һ���ܽ�;
1. `MessageQueue` ��Ϣ����ʵ����һ������ṹ������ `Java` �е� `LinkedList` ����ͨ��������ʽ�и��ô�����������Ϣ����ɾ�ǳ��Ŀ죻
2. `MessageQueue` �����е���Ϣ˳���Ǹ�����Ϣ��Ҫ���͵�ʱ������õģ�������ͬ�����ǵ��÷�����Ϣ������˳��
![MessageQueue](https://upload-images.jianshu.io/upload_images/7082912-36cb0492e8547423.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### Looper ��Ϣѭ����
��������ķ��������˽⵽��ͨ�� `Handler` �����͵���Ϣ���ջ������Ϣ��Ҫ���͵�ʱ����뵽 `MessageQueue` ���������У��� `Handler` �е��� `MessageQueue` �������������� `Looper` �����У����������ص������ `Looper` �����;
```Java
// ˽�еĹ���
private Looper(boolean quitAllowed) {
    // ������Ϣ����
    mQueue = new MessageQueue(quitAllowed);
    // ��ȡ��ǰ���߳�
    mThread = Thread.currentThread();
}
```

##### prepare() 
������ `Looper` �Ĺ��췽���з�����Ϣ������������ʵ��������ģ�ͬʱ����ȡ�˵�ǰ���̣߳���������Ĺ��캯����˽�еģ���������ж� `Looper` �����ʵ�����������ڲ���ɵģ�ͨ���Ķ�Դ�����ǿ��Է����� `prepare()` �����ڲ�ʵ������һ�� `Looper` ����;
```Java
// Looper ��
public static void prepare() {
    prepare(true);
}

// Looper ��
private static void prepare(boolean quitAllowed) {
    // �������һ���߳��е� Looper ����ֻ�ܵ���һ��  prepare() ������
    // Ҳ����ֻ��ʵ����һ�� Looper ���󣬾���鿴 sThreadLocal.get() �����Ľ���
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    // ��ʵ������ɵ� Looper ���󻺴浽 ThreadLocal ������
    sThreadLocal.set(new Looper(quitAllowed));
}

// ThreadLocal ��
public void set(T value) {
    // ��ȡ��ǰ���߳�
    Thread t = Thread.currentThread();
    // ��ȡ�̵߳� ThreadLocalMap ����
    ThreadLocalMap map = getMap(t);
    // �������´����� Looper ���󻺴浽
    // ��ǰ�̵߳� ThreadLocalMap ������
    // ThreadLocalMap ��Ұ������Ϊһ��HashMap����
    // ���ﲻ��չ���������Ŀ���������������
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}

// ThreadLocal ��
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}

// ThreadLocal ��
public T get() {
    // ��ȡ��ǰ���߳�
    Thread t = Thread.currentThread();
    // ��ȡ�߳��е� ThreadLocalMap ����
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        // �� Looper �е� ThreadLocal ������Ϊ Key����ȡ��Ӧ�� Entry
        // Ȼ����� Entry ��ȡ��Ӧ�� Looper ����
        // ������Ҫע����� Looper �е� ThreadLocal ����Ϊ��̬�ĳ�Ա����
        // Ҳ����ζ�Ÿñ������ڴ���ֻ��һ��Ҳ����ζ����� Key ��Ψһ�ģ�
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            // ���ﷵ�ػ���� Looper ����
            return result;
        }
    }
    // ������߳��ǵ�һ�ε��� prepare() �������ִ�е�����
    // �鿴setInitialValue() �Ĵ����֪�䷵�ص���һ�� null 
    return setInitialValue();
}
```
����Ĵ��벻�࣬Ҳ�ܺ���⣬����ĵط�Ҳ��д��ע�ͣ�����������һ���ܽ᣺
1. `Looper` �Ĺ��캯����˽�еģ�����ֻ�ܵ����� `prepare()` ������ʵ���� `Looper` ���󣬵���ʵ������ɵĶ����ǻ����ڵ�ǰ�̵߳� `ThreadLocalMap` �����еģ�
2. `Looper` �е���Ϣ����������ʵ������ʱ�򴴽��ģ�ͬʱ��ȡ�˴���`Looper` ����ʱ���ڵ��̣߳�
3. �� `prepare()`  ���������ǿ��Է���ÿ���߳�ֻ�ܵ���һ�θ÷���Ҳ����ÿ���߳����ֻ�ܳ���һ�� `Looper` ������һ����ڵ��� `prepare()` ������ʱ������жϣ�
4. �������� `Handler` ����ʱ�ķ��������ǿ���֪�� `Handler` �е� `Looper` ����Ҳ��ͨ����������� `sThreadLocal.get()` ������ȡ�ģ�����ǰ `Thread` �����е� `ThreadLocalMap` �л���� `Looper` ����
���������ڷ���ȥ�� `Handler` �Ĺ��캯�������Ƿ���������ص� `Looper` ����Ϊ�ռ�û���ڵ�ǰ�̵߳��� `Looper.prepare()` ������ȥ����`Handler`����ϵͳ�ǻ��׳��쳣��:
```Java
// handler ��
public Handler(Callback callback, boolean async) {
    // ʡ�Դ���...
    mLooper = Looper.myLooper();
    if (mLooper == null) {
        throw new RuntimeException(
            "Can't create handler inside thread that has not called Looper.prepare()");
    }
    // ʡ�Դ���...
}
```
��Ҳ����Ϊʲô���������߳���ֱ�Ӵ��� `Handler` ����ϵͳ�����쳣��ԭ�򣬼�����Ĵ���:
```Java
// ʾ������
new Thread(new Runnable() {
    @Override
    public void run() {
        // ���߳��в�������仰����Handler�����ǻ����쳣��
        Looper.prepare();
        Handler handler = new Handler(){
            @Override
            public void handleMessage(Message msg) {
                Log.e("TAG", "hahaha");
            }
        };
        handler.sendEmptyMessage(100);
        // ���߳��е����λ�ò�������仰���͵���Ϣ���޷����յ�
        // ���ｫ�����һ����ѭ��
        Looper.loop();
    }
}).start();
```
��ʱͬ����һ����������ˣ�Ϊʲô���������߳���ֱ��ʵ���� `Handler` ���󲻻����쳣������Ҫ���� `Looper.prepare()` ���д��룬������Ϸ������׳����������ᵽ **Only one Looper may be created per thread** �쳣�������Ȼ����ϵͳ�����̰߳����ǵ����� `Looper.prepare()` ���������ǿ�����Ӧ����������� `ActivityThread` �� `main()` ���������ҵ��������д���:
```Java
// ActivityThread ��
public static void main(String[] args) {
    // ʡ�Դ���...
    Looper.prepareMainLooper();
    // ʡ�Դ���...
    Looper.loop();
    throw new RuntimeException("Main thread loop unexpectedly exited");
}
```

##### loop()
���������`Looper` ���������ʵ�����ģ����������Ƿ�����������ν����ǵ���Ϣ�����е���Ϣ���ͳ����ģ������ `ActivityThread` �е� `main()` �������������� `Looper.loop()` �������������������������:
```Java
/**
 * Run the message queue in this thread. Be sure to call
 * {@link #quit()} to end the loop.
 */
 // Looper ��
public static void loop() {
    // ��ȡ��ǰ�̵߳� Looper ����
    final Looper me = myLooper();
    // ��ȡ Looper �е���Ϣ����
    final MessageQueue queue = me.mQueue;
    // ʡ�Դ���...

    // ������һ����ѭ�����ڻ�ȡ��Ե� Message
    for (;;) {
        Message msg = queue.next(); // might block
        if (msg == null) {
            // No message indicates that the message queue is quitting.
            return;
        }
        
        try {
            // �������Ƿ����� Message �� Target ������Ƿ��͸� Message
            // �� Handler ��������ת�� Handler �еĸ÷����з���
            msg.target.dispatchMessage(msg);
            
        } finally {
            if (traceTag != 0) {
                Trace.traceEnd(traceTag);
            }
        }

        // ���� Message �����������Ϊʲô�����ڴ�����Ϣ����ʱ
        // ��ֱ�� new ���ǵ��� Message.obtain() ��������ȡ��ԭ��
        msg.recycleUnchecked();
    }
}

// Handler ��
public void dispatchMessage(Message msg) {
    // �����ж���� Message �����е� Callback ��Ϊ��
    // ��ִ���� callback �����е� run ����
    // ���Ϊ�վ͵��� handler ����� handleMessage() ����
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

// Handler ��
private static void handleCallback(Message message) {
    message.callback.run();
}
```
loop() ��������Ҫ�ǽ����ǵ���Ϣ�����е���Ϣ���͵�ָ���� `Handler` �����У��������Ƕ� `loop()` ������һ���ܽ�:
1. ���� `loop()` ������ϵͳ�᲻�ϵ�ȥ��ȡ��ӵ���Ϣ����
2. ÿ�� `Message` �����������ڷ���ʱ���õ�  target ֵ��Handler�����ö�Ӧ����� `dispatchMessage()` ������
3. `Handler` �����е� `dispatchMessage()`  ��������ݷ��͵���Ϣ�����ͼ������������Ϣ�������ص���ͬ�ķ����������`sendMessage()` ���͵���Ϣ�ͻ���� `handMessage()` ����������� `post()` ���͵���Ϣ�ͻ�ص��� `Runnable` ����� `run()` ������
4. `loop()` ÿ��ѭ������ʱ�Ὣ `Message` ������л��գ���һ�����ǽ���������з�����

### Message ��Ϣ����
���������� `Handler` ����Ϣ���ƴ����ϵ��������Ǿͷ�������ˣ�����һ��������Ϣ����ĵ�����������򵥵ķ����£�ͨ�����Ƿ��͵�`Message` ���󶼲���ͨ�� new �����Ķ���ͨ��`Message.obtain()` ��������ȡ�ģ��о��������һ����֪������϶���Ϊ�˻�����Ϣ�����ֹ��Դ�˷ѣ�����������������Դ�뿴�������ʵ�ֵ�:
```JAVA
// Message ��
public static Message obtain() {
    synchronized (sPoolSync) {
        // ��ȡ����
        if (sPool != null) {
            Message m = sPool;
            // ������ĵڶ��������ᵽ��һ��λ��
            sPool = m.next;
            // �� Message ��������
            m.next = null;
            // �����Ϣ�ѱ�ʹ�õ� flag
            m.flags = 0; // clear in-use flag
            // ����������1
            sPoolSize--;
            return m;
        }
    }
    // ����ǵ�һ�Σ���ֱ��ʵ����Message����
    return new Message();
}
```
����ܼ���Ϊ���ǵ���Ϣ����ʱ����ṹ�����ֻ��洢�����ͷ���ɣ�ÿ��ͨ�� `obtain()` �õ� `Message` ���������ֻ��Ҫ����������� �����������������û�������Ϊ null��������δ����ǻ�ȡ���ǵĻ���� `Message` �����л�ȡ�ͻ��и�ֵ�������� `loop()` ������ for ѭ������ᵽ��������� `msg.recycleUnchecked()` ������ `Message` ������л���:
```JAVA
// Message ��
void recycleUnchecked() {
    // ����Ϣ��Ϊ�Ѿ�ʹ�ù���
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
    
    // �ж��Ƿ񳬹�����󻺴�����
    // û�г����򻺴���򲻻���
    synchronized (sPoolSync) {
        if (sPoolSize < MAX_POOL_SIZE) {
            next = sPool;
            sPool = this;
            sPoolSize++;
        }
    }
}
```
ͨ�� Message ��Ļ��淽���ķ������ǿ���֪�� Message �����Ļ�����ƣ�����Ϣִ����Ϻ�Ҳ���Զ��е���ʽ���л��棬ͬʱ�������������������޵ļ���MAX_POOL_SIZE = 50 ����

### �ܽ�
��ƪ������Ҫ�Ƕ� `Handler` ִ�����̵�Դ���������Ҫͨ�� `Handler` ,`MessageQueue` ,`Looper` ,`Message` ���ĸ�����еķ������м��漰���� `ThreadLocal` ,`ThreadLocalMap` ��֪ʶû�н�����չ������Ȥ�����ѿ��������������ϣ�д���µ���Ҫ��ͼҲ�ǰ����Լ�����һЩ�Ķ�����Դ�룬��ӭ���뷨�����ѽ�����