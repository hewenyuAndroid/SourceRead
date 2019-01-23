### Activity���
Activity��Ϊ����Android���û�������������ڿ����������Ǳز����ٵģ���Ϊһ��Android������Ա�������ڽӴ�Android�����ĵ�һ��϶��Ϳ�������Google������Activity��������ͼ:
![Activity��������](https://upload-images.jianshu.io/upload_images/7082912-27e31f9f6c5dbc6c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
ͼ�п��Ժ������Ŀ���Activity�ĸ����������ڵĵ���˳��on��ͷ�ı�����ĳ���ڵ���ã�,�������ÿ�������Ǹ����õĴ�ҿ���������һ�£����ǹ�עһ��Activity��������������ΰ������˳����õ���Щ����������ֱ�Ӵ�ActivityThread��ʼ������

### ActivityThread
main()������JavaӦ�ó������ڣ��ڳ������е�ʱ���һ��ִ�еķ�������main()��������ڽӴ�Java��ʱ�򿴵��ģ�������дAndroidӦ�õ�ʱ��������û�нӴ���main()������Ĭ�ϵľ��� `MainActivity` �� `onCreate()` ������ʼִ����,���ƺ������Ǹտ�ʼѧ���е㲻һ����
��Ϊ������ڵ�main()�������ǿ�����ActivityThread�������ҵ��������ǵ�Activity���������ٵ�һϵ�е��������ڵķ������������������ҵ���
��������������Ϊ������ڵ�main()����:
```Java
// ActivityThread��
public static void main(String[] args) {
    // ...
    // ������ActivityThread���󣬲�����attach����
    ActivityThread thread = new ActivityThread();
    thread.attach(false);
    // ...
}

// ActivityThread��
private void attach(boolean system) {
    // ...
    // ���ﴫ������Ϊfalse
    if (!system) {
        final IActivityManager mgr = ActivityManager.getService();
        try {
            // ����õ��Ķ���Ϊ������ActivityManagerService����
            // �������ֱ��ת��ActivityManagerService�в鿴
            mgr.attachApplication(mAppThread);
        } catch (RemoteException ex) {
            throw ex.rethrowFromSystemServer();
        }
    } 
    // ...
}

```
����������Ҫ��עһ�� `mgr.attachApplication(mAppThread);` ���д����д���Ĳ���Ϊ `ApplicationThread` ���󣬸���Ϊ `ActivityThread` ���ڲ��࣬����Activity�Ķ�����Ϣ��������ڲ��෢����
```Java
// ActivityManagerService��
@Override
public final void attachApplication(IApplicationThread thread) {
    //  ...
    // ��������˸÷���
    attachApplicationLocked(thread, callingPid);
    // ...
}

// ActivityManagerService��
 private final boolean attachApplicationLocked(IApplicationThread thread,
            int pid) {

    // ������󽫻Ỻ�����ǵ� thread �������� ApplicationThread ����
    ProcessRecord app;
    // ...
    // ����Ỻ����һ��ע����˵�� thread ����
    app.makeActive(thread, mProcessStats);
   
    try {
        // ...
        // TAG1
        // �����thread��������ǰ��������� ApplicationThread ����
        // �������ֱ�Ӳ鿴 ApplicationThread  �е� bindApplication() ������
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
            // �����������ִ�� ApplicationThread �е� scheduleLaunchActivity() ����
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
������Դ��������ָ���������ȽϹؼ��ĵ� TAG1 �� TAG2, ���������Ƿֱ�����������������:

> **TAG1**

����ƽ��д��Ŀ�ж������Ŀָ��һ��Application�࣬����Ҳ��ӵ�� `onCreate()` ���������ڷ���������Application�� `onCreate` ������ȵ�һ��Activity��`onCreate()` ��������ִ�У����������Ǵ�Դ��ĽǶ���������Ϊʲô��������ִ�е�:
```Java
// ActivityThread --> ApplicationThread��
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
        // ���﷢����һ��������Ϣ���ڰ�Application
        // ����Ķ�����Ϣ�� Handler ������ H �෢�ͣ����뷽���ڲ����ɲ鿴
        // H ��Ҳ�� ActivityThread ���ڲ���
        sendMessage(H.BIND_APPLICATION, data);
    }
    

// ActivityThread ��
// �������淢�͵Ķ�����Ϣ������׷�ٵ�����ִ�д˷���
private void handleBindApplication(AppBindData data) {
    // ...
    if (ii != null) {
        // ...
    } else {
        // ���ﴴ���� mInstrumentation ����,
        // �ö��������Ǻ���Activity���������ڷ������������ŷǳ���Ҫ������
        mInstrumentation = new Instrumentation();
    }
        // ...
        try {
            // �����������ִ�����ǵ�Application��� onCreate()
            mInstrumentation.callApplicationOnCreate(app);
        } catch (Exception e) {
          
        }
    } finally {
        StrictMode.setThreadPolicy(savedPolicy);
    }

}

// Instrumentation ��
public void callApplicationOnCreate(Application app) {
    // ���ＴΪ���� Application �� onCreate()
    app.onCreate();
}
```

> **TAG2**

���������ǿ��µڶ�����ǩλ�õ����ݣ�����ϵͳ������ `ActivityStackSupervisor`  ����� `attachApplicationLocked(app)` ������ͬʱ������ `ProcessRecord` ���������ע���������ᵽ�����ö��󻺴������ǵ� `ApplicationThread` ����
```Java
// ActivityStackSupervisor ��
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

// ActivityStackSupervisor ��
final boolean realStartActivityLocked(ActivityRecord r, ProcessRecord app,
        boolean andResume, boolean checkConfig) throws RemoteException {
    // ...
    try {
        // ...
        // ��������� ApplicationThread �е� scheduleLaunchActivity() ����
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
���ǿ��� `ActivityStackSupervisor` �������ջ��ǵ����� `ApplicationThread` ����� `scheduleLaunchActivity()` ����������Ĵ����������ActivityThread������ʱ��ϵͳ��ʼ��Application�͵����ǵĵ�һ��Activity����ǰ�����̣���ʱ���ǵ�Activity��û�д���������Application�����Ѿ�������ִ����`onCreate()` ��������������ڴ����е��� `startActivity(Intent intent)`  ��������Activity�����ߵ�����һ�����̵������ն���ִ�е� `realStartActivityLocked()`  ����������������漴ΪActivity������ִ�����������ڵķ��������̣�

### Activity�Ĵ������������ڷ�����ִ��
�����������ķ�����`realStartActivityLocked()`  ������� ApplicationThread �� `scheduleLaunchActivity()` ������������һ���������:
```Java
@Override
// ActivityThread --> ApplicationThread��
public final void scheduleLaunchActivity(Intent intent, IBinder token, int ident,
        ActivityInfo info, Configuration curConfig, Configuration overrideConfig,
        CompatibilityInfo compatInfo, String referrer, IVoiceInteractor voiceInteractor,
        int procState, Bundle state, PersistableBundle persistentState,
        List<ResultInfo> pendingResults, List<ReferrerIntent> pendingNewIntents,
        boolean notResumed, boolean isForward, ProfilerInfo profilerInfo) {
    // ...
    // ����Ҳ�Ƿ�����һ��������Ϣ��������ϵͳ����һ��Activityʵ��
    sendMessage(H.LAUNCH_ACTIVITY, r);
}

// ActivityThread ��
// ͨ��������Ϣ���е� handleMessage() ���������յ����˸÷���
@Override
private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent, String reason) {
    // ...
    // TAG3
    // ���ｫ��Activityʵ�������ĵط�
    Activity a = performLaunchActivity(r, customIntent);
    if (a != null) {
        // ...
        // TAG4
        // ����ִ��Activity��onResume() ����
        handleResumeActivity(r.token, false, r.isForward,
                !r.activity.mFinished && !r.startsNotResumed, r.lastProcessedSeq, reason);
        if (!r.activity.mFinished && r.startsNotResumed) {
            // TAG5
            // �����ж�Activityû������ͬʱʧȥ�����˽���ִ��onPause(������
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
ͬ����Ҳ�������������������������Ҫ�Ľڵ�������:

> **TAG3**

����ִ����`performLaunchActivity(r, customIntent);` ���������շ�����һ��Activity�������������ǾͿ��Բ²�һ��Activity�����ʵ������������������洴���ģ����������ǽ�ͨ���Ը÷�����Դ���������֤���ǵĲ���:
```Java
// ActivityThread ��
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    // ...
    // �ö��󻺴������Ǵ��� Intent ����ʱ�����Ŀ��Activity �� Class ��Ϣ
    ComponentName component = r.intent.getComponent();
  
    if (r.activityInfo.targetActivity != null) {
        component = new ComponentName(r.activityInfo.packageName,
                r.activityInfo.targetActivity);
    }

    ContextImpl appContext = createBaseContextForActivity(r);
    Activity activity = null;
    try {
        // ��ȡ�������
        java.lang.ClassLoader cl = appContext.getClassLoader();
        // ����ͨ�����䴴�����ǵ�Ŀ��Activity����
        // ��֤�����ǵ�֮ǰ�Ĳ���
        activity = mInstrumentation.newActivity(
                cl, component.getClassName(), r.intent);
       // ...
    } catch (Exception e) {
    }

    try {
        Application app = r.packageInfo.makeApplication(false, mInstrumentation);

        if (activity != null) {
            // ...
            // ������� activity �ķ�������ʼ�� Activity ��һЩ��Ϣ��
            // �������� Activity �е� PhoneWindow ����
            activity.attach(appContext, this, getInstrumentation(), r.token,
                    r.ident, app, r.intent, r.activityInfo, title, r.parent,
                    r.embeddedID, r.lastNonConfigurationInstances, config,
                    r.referrer, r.voiceInteractor, window, r.configCallback);
            
            if (r.isPersistable()) {
                mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
            } else {
                // �����������ִ�� Activity �� onCreate() ����
                // activity.performCreate(icicle); --> onCreate(icicle);
                // ��ʱActivity�ոմ���ͬʱ�Ѿ���ʼ����һЩ��Ϣ,����Window��
                mInstrumentation.callActivityOnCreate(activity, r.state);
            }
          
            if (!r.activity.mFinished) {
                // �����������ִ�� Acrivity �� onStart() ����
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
                    // ���ｫ����� Activity �� onRestoreInstanceState(Bundle savedInstanceState); 
                    // ���ǿ��Է��� Activity �� onRestoreInstanceState ��ȡ����Bundle�����
                    // onCreate �����л�ȡ������һ���ģ�ֻ���� onCreate ����������Ҫ�� Bundle �����п�
                    mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state);
                }
            }
            // ...
        }
        // ���ｫ�����õ�Activity���浽ϵͳ�ļ�����
        mActivities.put(r.token, r);
    } catch (SuperNotCalledException e) {
    } catch (Exception e) {
    }

    return activity;
}
```
������ `performLaunchActivity()` �������������ڲ�������Ϣ���ܴ���������ǽ�����һ���ܽ᣺
1. `Activity` ����Ĵ�����ͨ��������ɵģ���Ҳ�Ϳ��Խ���Ϊʲô���������� `Activity` ʱ���� `Intent` ����ֱ�Ӵ���Ŀ�� `Activity` �� `Class` ���󼴿ɣ�
2. `Activity` �ڴ�������������ϵ��� `activity.attach()` ��������ʼ����Ҫ����Ϣ������ Window ��Thread�ȶ���
3. `Activity`  �ڳ�ʼ�����Ҫ�����ݺ������� `onCreate(Bundle saveInstance)` ������
4. `Activity`  �ڵ����� `onCreate()` ���������û������������������ `onStart()` ���������������һ�����룬����� `Activity`  �� `onCreate()` ����������� `finish()` �������� `Activity`  ��������� `onStart()` ������
5. �ڵ����� `onStart()` �����������Activity����ϵͳ������ `Bundle` �е����ݣ������� `onRestoreInstanceState(Bundle savedInstanceState);` �������˷����е� `Bundle` ���ݺ� `onCreate()` �����е�����Ϊͬһ����ֻ���� `onCreate()` �����е� `Bundle` ��Ҫ�Լ��пգ�

> **TAG4**

����������һ���������Ѿ�֪���� `Activity` ���󴴽����� `onCreate()`  �� `onStart()` ��������ʲôʱ�򴴽��ģ����ݱ����ʼ��ͼƬ���������õĽ����� `Activity` �� `onResume()`  ������Ҳ�������ǽ�����Ҫ�����ĵ㣺
```Java
// ActivityThread ��
final void handleResumeActivity(IBinder token,
        boolean clearHide, boolean isForward, boolean reallyResume, int seq, String reason) {
    // ...
    // ���ｫ��ִ�� Activity �� onResume() ����
    r = performResumeActivity(token, clearHide, reason);

    if (r != null) {
        final Activity a = r.activity;
        // ������Ĵ����ǽ��� onCreate() �����д�����DecorView ����
        // �͵�ǰ��Activity�е�Window�󶨣�Ȼ��ִ��View���������ڷ�����
        // ��Ҳ����Ϊʲô��Activity�ĵ�һ�� onResume() �������޷���ȡ��
        // �ؼ��Ŀ�ߵ���Ϣ��ԭ�򣬹���View�Ļ������̣����ǽ���������
        // һƪ�����з��������ﲻ����չ��
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
�������ִ�� `onResume()` �������߼����ǱȽϼ򵥵ģ�����`handleResumeActivity()` �����в�������ִ�� `onResume()` ������ô�򵥣��������������� `onCreate()` �����д�������ͼ���� `DecorView`  ͬ����`Activity` �Ĵ��ڰ󶨣�ͬʱִ�� `View` �Ļ������̷��������ﲻ��չ�������ڻ�ר�ŶԴ�ģ�������
������������δ���ķ��������Ƿ����� `Activity` �� `onCreate()` ��`onStart()` �ĵ��ú� `Activity` ������ͬһ�������У��� `onResume()` �������Ƕ���������һ��������������Ƶĺô����� `Activity` ʧȥ����ͻ�ý����Ƶ���ǱȽϸߵģ����类���� `Activity` ���ǣ�������������ֻ�����ڴ汻���ջ����û��ֶ��رղŻ���ã����ʹ��Ƶ����ԱȽϵͣ�ͨ�� `Activity` ����������ͼҲ����֤��Ϊʲô����ô��ƣ�
������ `Activity` �Ϳ���չʾ����Ļ���ˣ�