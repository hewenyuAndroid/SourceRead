### Activity简介
Activity作为承载Android和用户交互的组件，在开发过程中是必不可少的，作为一个Android开发人员，我们在接触Android开发的第一天肯定就看过这张Google给出的Activity生命周期图:
![Activity生命周期](https://upload-images.jianshu.io/upload_images/7082912-27e31f9f6c5dbc6c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
图中可以很清晰的看到Activity的各个生命周期的调用顺序（on开头的表明在某个节点调用）,这里关于每个方法是干嘛用的大家可以网上搜一下，我们关注一下Activity启动过程中是如何按照这个顺序调用的这些方法，我们直接从ActivityThread开始分析；

### ActivityThread
main()函数是Java应用程序的入口，在程序运行的时候第一个执行的方法就是main()，这个是在接触Java的时候看到的，但是在写Android应用的时候好像根本没有接触过main()函数，默认的就是 `MainActivity` 的 `onCreate()` 方法开始执行了,这似乎和我们刚开始学的有点不一样；
作为程序入口的main()函数我们可以在ActivityThread类里面找到，而我们的Activity创建、销毁等一系列的生命周期的方法都可以在这里面找到；
我们先来看下作为程序入口的main()函数:
```Java
// ActivityThread类
public static void main(String[] args) {
    // ...
    // 创建了ActivityThread对象，并调用attach方法
    ActivityThread thread = new ActivityThread();
    thread.attach(false);
    // ...
}

// ActivityThread类
private void attach(boolean system) {
    // ...
    // 这里传进来的为false
    if (!system) {
        final IActivityManager mgr = ActivityManager.getService();
        try {
            // 这里得到的对象为单例的ActivityManagerService对象
            // 因此我们直接转到ActivityManagerService中查看
            mgr.attachApplication(mAppThread);
        } catch (RemoteException ex) {
            throw ex.rethrowFromSystemServer();
        }
    } 
    // ...
}

```
这里我们需要关注一下 `mgr.attachApplication(mAppThread);` 这行代码中传入的参数为 `ApplicationThread` 对象，该类为 `ActivityThread` 的内部类，创建Activity的队列消息将由这个内部类发出；
```Java
// ActivityManagerService类
@Override
public final void attachApplication(IApplicationThread thread) {
    //  ...
    // 这里调用了该方法
    attachApplicationLocked(thread, callingPid);
    // ...
}

// ActivityManagerService类
 private final boolean attachApplicationLocked(IApplicationThread thread,
            int pid) {

    // 这个对象将会缓存我们的 thread 参数，即 ApplicationThread 对象
    ProcessRecord app;
    // ...
    // 这里会缓存上一行注释中说的 thread 参数
    app.makeActive(thread, mProcessStats);
   
    try {
        // ...
        // TAG1
        // 这里的thread对象我们前面分析过是 ApplicationThread 对象，
        // 因此我们直接查看 ApplicationThread  中的 bindApplication() 方法；
        if (app.instr != null) {
            thread.bindApplication(processName, appInfo, providers,
                    app.instr.mClass,
                    profilerInfo, app.instr.mArguments,
                    app.instr.mWatcher,
                    app.instr.mUiAutomationConnection, testMode,
                    mBinderTransactionTrackingEnabled, enableTrackAllocation,
                    isRestrictedBackupMode || !normalMode, app.persistent,
                    new Configuration(getGlobalConfiguration()), app.compat,
                    getCommonServicesLocked(app.isolated),
                    mCoreSettingsObserver.getCoreSettingsLocked(),
                    buildSerial);
        } else {
            thread.bindApplication(processName, appInfo, providers, null, profilerInfo,
                    null, null, null, testMode,
                    mBinderTransactionTrackingEnabled, enableTrackAllocation,
                    isRestrictedBackupMode || !normalMode, app.persistent,
                    new Configuration(getGlobalConfiguration()), app.compat,
                    getCommonServicesLocked(app.isolated),
                    mCoreSettingsObserver.getCoreSettingsLocked(),
                    buildSerial);
        }

    } catch (Exception e) {
        // ...
        return false;
    }
  
    if (normalMode) {
        try {
            // TAG2
            // 这个方法将会执行 ApplicationThread 中的 scheduleLaunchActivity() 方法
            if (mStackSupervisor.attachApplicationLocked(app)) {
                didSomething = true;
            }
        } catch (Exception e) {
          
        }
    }
    // ...
    return true;
}

```
上述的源码中我们指定了两个比较关键的点 TAG1 和 TAG2, 接下来我们分别来分析下这两个点:

> **TAG1**

我们平常写项目中都会给项目指定一个Application类，该类也会拥有 `onCreate()` 等生命周期方法，而且Application的 `onCreate` 方法会比第一个Activity的`onCreate()` 方法优先执行，接下来我们从源码的角度来分析下为什么回是这样执行的:
```Java
// ActivityThread --> ApplicationThread类
public final void bindApplication(String processName, ApplicationInfo appInfo,
                List<ProviderInfo> providers, ComponentName instrumentationName,
                ProfilerInfo profilerInfo, Bundle instrumentationArgs,
                IInstrumentationWatcher instrumentationWatcher,
                IUiAutomationConnection instrumentationUiConnection, int debugMode,
                boolean enableBinderTracking, boolean trackAllocation,
                boolean isRestrictedBackupMode, boolean persistent, Configuration config,
                CompatibilityInfo compatInfo, Map services, Bundle coreSettings,
                String buildSerial) {
        // ...
        // 这里发送了一个队列消息用于绑定Application
        // 这里的队列消息由 Handler 的子类 H 类发送，进入方法内部即可查看
        // H 类也是 ActivityThread 的内部类
        sendMessage(H.BIND_APPLICATION, data);
    }
    

// ActivityThread 类
// 根据上面发送的队列消息，我们追踪到它会执行此方法
private void handleBindApplication(AppBindData data) {
    // ...
    if (ii != null) {
        // ...
    } else {
        // 这里创建了 mInstrumentation 对象,
        // 该对象在我们后面Activity的生命周期方法调用中起着非常重要的作用
        mInstrumentation = new Instrumentation();
    }
        // ...
        try {
            // 这个方法将会执行我们的Application类的 onCreate()
            mInstrumentation.callApplicationOnCreate(app);
        } catch (Exception e) {
          
        }
    } finally {
        StrictMode.setThreadPolicy(savedPolicy);
    }

}

// Instrumentation 类
public void callApplicationOnCreate(Application app) {
    // 这里即为调用 Application 的 onCreate()
    app.onCreate();
}
```

> **TAG2**

接下来我们看下第二个标签位置的内容，这里系统调用了 `ActivityStackSupervisor`  对象的 `attachApplicationLocked(app)` 方法，同时传入了 `ProcessRecord` 对象，上面的注释中我们提到过，该对象缓存了我们的 `ApplicationThread` 对象；
```Java
// ActivityStackSupervisor 类
boolean attachApplicationLocked(ProcessRecord app) throws RemoteException {
        // ...
        try {
            if (realStartActivityLocked(hr, app, true, true)) {
                didSomething = true;
            }
        } catch (RemoteException e) {
        }
         // ...      
    return didSomething;
}

// ActivityStackSupervisor 类
final boolean realStartActivityLocked(ActivityRecord r, ProcessRecord app,
        boolean andResume, boolean checkConfig) throws RemoteException {
    // ...
    try {
        // ...
        // 这里调用了 ApplicationThread 中的 scheduleLaunchActivity() 方法
        app.thread.scheduleLaunchActivity(new Intent(r.intent), r.appToken,
                System.identityHashCode(r), r.info,
                // TODO: Have this take the merged configuration instead of separate global and
                // override configs.
                mergedConfiguration.getGlobalConfiguration(),
                mergedConfiguration.getOverrideConfiguration(), r.compat,
                r.launchedFromPackage, task.voiceInteractor, app.repProcState, r.icicle,
                r.persistentState, results, newIntents, !andResume,
                mService.isNextTransitionForward(), profilerInfo);
        // ...
    } catch (RemoteException e) { 
    }
    return true;
}
```
我们看到 `ActivityStackSupervisor` 对象最终还是调用了 `ApplicationThread` 对象的 `scheduleLaunchActivity()` 方法，上面的代码分析的是ActivityThread创建的时候系统初始化Application和到我们的第一个Activity创建前的流程，此时我们的Activity还没有创建，但是Application对象已经创建并执行了`onCreate()` 方法，如果我们在代码中调用 `startActivity(Intent intent)`  方法启动Activity则是走的另外一套流程但是最终都会执行到 `realStartActivityLocked()`  方法，这个方法后面即为Activity创建和执行其生命周期的方法的流程；

### Activity的创建和生命周期方法的执行
根据上面代码的分析，`realStartActivityLocked()`  将会调用 ApplicationThread 的 `scheduleLaunchActivity()` 方法，先来看一下这个方法:
```Java
@Override
// ActivityThread --> ApplicationThread类
public final void scheduleLaunchActivity(Intent intent, IBinder token, int ident,
        ActivityInfo info, Configuration curConfig, Configuration overrideConfig,
        CompatibilityInfo compatInfo, String referrer, IVoiceInteractor voiceInteractor,
        int procState, Bundle state, PersistableBundle persistentState,
        List<ResultInfo> pendingResults, List<ReferrerIntent> pendingNewIntents,
        boolean notResumed, boolean isForward, ProfilerInfo profilerInfo) {
    // ...
    // 这里也是发送了一个队列消息用来告诉系统创建一个Activity实例
    sendMessage(H.LAUNCH_ACTIVITY, r);
}

// ActivityThread 类
// 通过分析消息队列的 handleMessage() 方法，最终调用了该方法
@Override
private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent, String reason) {
    // ...
    // TAG3
    // 这里将是Activity实例创建的地方
    Activity a = performLaunchActivity(r, customIntent);
    if (a != null) {
        // ...
        // TAG4
        // 这里执行Activity的onResume() 方法
        handleResumeActivity(r.token, false, r.isForward,
                !r.activity.mFinished && !r.startsNotResumed, r.lastProcessedSeq, reason);
        if (!r.activity.mFinished && r.startsNotResumed) {
            // TAG5
            // 这里判断Activity没有销毁同时失去焦点了将会执行onPause(）方法
            performPauseActivityIfNeeded(r, reason);
            if (r.isPreHoneycomb()) {
                r.state = oldState;
            }
        }
    } else {
      // ...
    }
}
```
同样我也在上述代码中找了三个相对重要的节点来分析:

> **TAG3**

这里执行了`performLaunchActivity(r, customIntent);` 方法并最终返回了一个Activity对象，在这里我们就可以猜测一下Activity对象的实例就是在这个方法里面创建的，接下来我们将通过对该方法的源码分析来验证我们的猜想:
```Java
// ActivityThread 类
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    // ...
    // 该对象缓存了我们创建 Intent 对象时传入的目标Activity 的 Class 信息
    ComponentName component = r.intent.getComponent();
  
    if (r.activityInfo.targetActivity != null) {
        component = new ComponentName(r.activityInfo.packageName,
                r.activityInfo.targetActivity);
    }

    ContextImpl appContext = createBaseContextForActivity(r);
    Activity activity = null;
    try {
        // 获取类加载器
        java.lang.ClassLoader cl = appContext.getClassLoader();
        // 这里通过反射创建我们的目标Activity对象
        // 验证了我们的之前的猜想
        activity = mInstrumentation.newActivity(
                cl, component.getClassName(), r.intent);
       // ...
    } catch (Exception e) {
    }

    try {
        Application app = r.packageInfo.makeApplication(false, mInstrumentation);

        if (activity != null) {
            // ...
            // 这里调用 activity 的方法来初始化 Activity 的一些信息，
            // 包括创建 Activity 中的 PhoneWindow 对象
            activity.attach(appContext, this, getInstrumentation(), r.token,
                    r.ident, app, r.intent, r.activityInfo, title, r.parent,
                    r.embeddedID, r.lastNonConfigurationInstances, config,
                    r.referrer, r.voiceInteractor, window, r.configCallback);
            
            if (r.isPersistable()) {
                mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
            } else {
                // 这个方法将会执行 Activity 的 onCreate() 方法
                // activity.performCreate(icicle); --> onCreate(icicle);
                // 此时Activity刚刚创建同时已经初始化了一些信息,例如Window等
                mInstrumentation.callActivityOnCreate(activity, r.state);
            }
          
            if (!r.activity.mFinished) {
                // 这个方法将会执行 Acrivity 的 onStart() 方法
                activity.performStart();
                r.stopped = false;
            }
            if (!r.activity.mFinished) {
                if (r.isPersistable()) {
                    if (r.state != null || r.persistentState != null) {
                        mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state,
                                r.persistentState);
                    }
                } else if (r.state != null) {
                    // 这里将会调用 Activity 的 onRestoreInstanceState(Bundle savedInstanceState); 
                    // 我们可以发现 Activity 的 onRestoreInstanceState 获取到的Bundle对象和
                    // onCreate 方法中获取到的是一样的，只不过 onCreate 方法里面需要对 Bundle 数据判空
                    mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state);
                }
            }
            // ...
        }
        // 这里将创建好的Activity缓存到系统的集合中
        mActivities.put(r.token, r);
    } catch (SuperNotCalledException e) {
    } catch (Exception e) {
    }

    return activity;
}
```
分析完 `performLaunchActivity()` 方法，发现其内部还是信息量很大的这里我们将其做一下总结：
1. `Activity` 对象的创建是通过反射完成的，这也就可以解释为什么我们在启动 `Activity` 时，在 `Intent` 里面直接传入目标 `Activity` 的 `Class` 对象即可；
2. `Activity` 在创建完对象后会马上调用 `activity.attach()` 方法来初始化必要的信息，包括 Window ，Thread等对象；
3. `Activity`  在初始化完必要的数据后会调用其 `onCreate(Bundle saveInstance)` 方法；
4. `Activity`  在调用完 `onCreate()` 方法后如果没有销毁则会继续调用其 `onStart()` 方法，这里可以做一个设想，如果在 `Activity`  的 `onCreate()` 方法里面调用 `finish()` 方法，则 `Activity`  将不会调用 `onStart()` 方法；
5. 在调用完 `onStart()` 方法后，如果在Activity存在系统缓存在 `Bundle` 中的数据，则会调用 `onRestoreInstanceState(Bundle savedInstanceState);` 方法，此方法中的 `Bundle` 数据和 `onCreate()` 方法中的数据为同一个，只不过 `onCreate()` 方法中的 `Bundle` 需要自己判空；

> **TAG4**

分析完上面一步，我们已经知道了 `Activity` 对象创建和其 `onCreate()`  和 `onStart()` 方法是在什么时候创建的，根据本文最开始的图片接下来调用的将会是 `Activity` 的 `onResume()`  方法，也就是我们接下来要分析的点：
```Java
// ActivityThread 类
final void handleResumeActivity(IBinder token,
        boolean clearHide, boolean isForward, boolean reallyResume, int seq, String reason) {
    // ...
    // 这里将会执行 Activity 的 onResume() 方法
    r = performResumeActivity(token, clearHide, reason);

    if (r != null) {
        final Activity a = r.activity;
        // 这下面的代码是将在 onCreate() 方法中创建的DecorView 对象
        // 和当前的Activity中的Window绑定，然后执行View的生命周期方法，
        // 这也就是为什么在Activity的第一次 onResume() 方法中无法获取到
        // 控件的宽高等信息的原因，关于View的绘制流程，我们将会在另外
        // 一篇文章中分析，这里不做扩展；
        if (r.window == null && !a.mFinished && willBeVisible) {
            r.window = r.activity.getWindow();
            View decor = r.window.getDecorView();
            decor.setVisibility(View.INVISIBLE);
            ViewManager wm = a.getWindowManager();
            WindowManager.LayoutParams l = r.window.getAttributes();
            a.mDecor = decor;
            l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
            l.softInputMode |= forwardBit;
            if (r.mPreserveWindow) {
                a.mWindowAdded = true;
                r.mPreserveWindow = false;
                // ...
                ViewRootImpl impl = decor.getViewRootImpl();
                if (impl != null) {
                    impl.notifyChildRebuilt();
                }
            }
            if (a.mVisibleFromClient) {
                if (!a.mWindowAdded) {
                    a.mWindowAdded = true;
                    wm.addView(decor, l);
                } else {
                    // ...
                    a.onWindowAttributesChanged(l);
                }
            }
        } 
    } 
}
```
这里关于执行 `onResume()` 方法的逻辑还是比较简单的，不过`handleResumeActivity()` 方法中不仅仅是执行 `onResume()` 方法这么简单，他还负责将我们在 `onCreate()` 方法中创建的视图对象 `DecorView`  同我们`Activity` 的窗口绑定，同时执行 `View` 的绘制流程方法，这里不做展开，后期会专门对此模块分析；
根据上面的两段代码的分析，我们发现了 `Activity` 的 `onCreate()` 和`onStart()` 的调用和 `Activity` 创建是同一个方法中，而 `onResume()` 方法则是独立出来的一个方法，这样设计的好处在于 `Activity` 失去焦点和获得焦点的频率是比较高的，比如被其它 `Activity` 覆盖，而创建和销毁只有在内存被回收或者用户手动关闭才会调用，因此使用频率相对比较低，通过 `Activity` 的生命周期图也可以证明为什么是这么设计；
到这里 `Activity` 就可以展示到屏幕上了；