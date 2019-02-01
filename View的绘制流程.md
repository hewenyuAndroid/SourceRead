### View�Ļ�������&Activity����������
�տ�ʼѧAndroid������ʱ�򣬶��Զ���ؼ��Ƚϸ���Ȥ��Ҳ��������һЩ����ֵֹ����⣬������ `Activity` �� `onCreate()` / `onResume()` ������ȥ��ȡ `View` �Ŀ�ߣ���������񲢲�����ô���е�ͨ��
��[Activity��������](https://www.jianshu.com/p/c417f21d7a44)�����Ƿ����� `Activity` ����ʱ�������ڷ����Ĵ���ʱ����ͬ��`View`Ҳ���Լ���һ��ִ������ `onMeasure()` ��`onLayout()` ��`onDraw()` ��`View` ����һ�����̺� `Activity` �Ĵ���������ʲô��ϵ��Ϊʲô������ `Activity` �� `onCreate()` �������޷���ȡ�� `View` �Ŀ�ߣ��� `onResume()` ��������ʱ���ܻ�ȡ����ߣ���ʱ���ֲ����ԣ�����������ͨ��Դ����з�����

### setContentView()���ز����ļ�
�ڴ���Activity��ʱ������ͨ���������� `onCreate()` �����е��� `setContentView();` ����������д�õĲ����ļ����سɶ�Ӧ��`View` ���󣬹��ڵ��ô˷���������ν����ǵ� XML ���ֽ�������ͼ�Ŷ���Ŀ��Բ鿴�ҵ���ƪ���� [Android �����ļ����� LayoutInflater Դ�����](https://www.jianshu.com/p/846c36af2c2b)������򵥵Ľ����¾���ͨ�� XML �ļ��������õ���Ӧ��View��ȫ���������ͨ�������������ǵĶ���
�������� `setContentView()` ������Դ�룬�����������ǵ����� `Window` �� `setContentView()` ������ǰ��������[Activity��������](https://www.jianshu.com/p/c417f21d7a44)�����Ƿ����� `Activity` ��ͨ�����䴴����������� `Activity` �� `attach()` ��������ʼ�������Ϣ�������оͰ��� `Window`  ���󣬶����������� `PhoneWindow` ����
```Java
// Activity ��
final void attach(Context context, ActivityThread aThread,
        Instrumentation instr, IBinder token, int ident,
        Application application, Intent intent, ActivityInfo info,
        CharSequence title, Activity parent, String id,
        NonConfigurationInstances lastNonConfigurationInstances,
        Configuration config, String referrer, IVoiceInteractor voiceInteractor,
        Window window, ActivityConfigCallback activityConfigCallback) {
    // ...
    mWindow = new PhoneWindow(this, window, activityConfigCallback);
   // ...
}
```
���������ǽ��뵽 `PhoneWindow` ����� `setContentView()` �����з�����Դ��:
```Java
// PhoneWindow ��
@Override
public void setContentView(int layoutResID) {
    if (mContentParent == null) {
        // ���� DecorView
        installDecor();
    } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        mContentParent.removeAllViews();
    }

    if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
                getContext());
        transitionTo(newScene);
    } else {
        // �����ǵ�XML����������View���ص�ϵͳ������
        mLayoutInflater.inflate(layoutResID, mContentParent);
    }
    // ...
}
```
������������ `installDecor();` ����������������ڴ��� `DecorView` �� `mContentParent` ���󣬺������ǻ��������������ϸ���������������������������Դ��:
```Java
// PhoneWindow ��
private void installDecor() {
    // ...
    if (mDecor == null) {
        // TAG1
        // ���� DecorView ����
        mDecor = generateDecor(-1);
        // ...
    } else {
        mDecor.setWindow(this);
    }
    if (mContentParent == null) {
        // TAG2
        // ���� mContentParent ����
        mContentParent = generateLayout(mDecor);
        // ...
    }
}
```
> **TAG1** mDecor ����Ĵ���

���ﴴ����һ�� `DecorView` ������һ�� `FrameLayout` ���͵Ķ������Ͽ��˺ܶ಩�ͣ�����ǰ� `DecorView` ������Activity�еĸ����֣������������Ǳ����ص�һ��ʵ���� `ViewParent` �ӿڵ� `ViewRootImpl` �����У�Ȼ��ű����ص�Window �����ϣ��������ǻ������һ�����ݣ������Ըо���仰Ҳ������ô�����У�
```Java
// PhoneWindow ��
protected DecorView generateDecor(int featureId) {
    // ...
    // ����ֱ��ͨ�� new ���󴴽��� DecorView ����
    return new DecorView(context, featureId, this, getAttributes());
}
```

> **TAG2** mContentParent ����Ĵ���

����������������������ܹ��²��������Ӧ�þ������� XML ���ֵĸ����֣��� `mContentParent` ����ĸ��໹���� `mDecor` �����������滹������һ�� `ViewGroup` �����������ñ���ȣ��鿴��ͬ�Ĳ����ļ������ǿ��Է�������� `mContentParent` ������ʵ��һ���̶� Id = **com.android.internal.R.id.content** ��FrameLayout �����õ����ID��ʵ���ǿ������ܶ����飬����: API�汾Ϊ 19(4.4) �� 21(5.0)֮��ĳ���ʽ״̬�������õȣ�
```Java
// PhoneWindow ��
protected ViewGroup generateLayout(DecorView decor) {
    // ...
    // ���ض�Ӧ�Ĳ����ļ�������Ĳ����ļ���������������õ�����������
    // ����: noTitle �Ĳ��ֵ�
    int layoutResource;
    int features = getLocalFeatures();
    // System.out.println("Features: 0x" + Integer.toHexString(features));
    if ((features & (1 << FEATURE_SWIPE_TO_DISMISS)) != 0) {
        layoutResource = R.layout.screen_swipe_dismiss;
        setCloseOnSwipeEnabled(true);
    }else if(){
        // ...
        // �������Ĵ���Ͳ��������ˣ���ҿ����Լ��鿴Դ��
        // �ҿ��İ汾�� 26
    } else {
        layoutResource = R.layout.screen_simple;
    }
    
    // ����Ӧ���ԵĲ����ļ����ص� DecorView ��
    mDecor.onResourcesLoaded(mLayoutInflater, layoutResource);
    // ����Id��ȡ contentParent���󣬴� Id �̶�Ϊ: com.android.internal.R.id.content
    // ����� contentParent ������ʵΪ FrameLayout ���͵Ķ���
    ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);
    // ...
    return contentParent;
}
```
ִ�е��������ǵ� `mContentParent` �����Ѿ�������ϣ�Ȼ��ط��ص����������ᵽ���� `PhoneWindow` ��`setConentView()` �����н������Լ�д�Ĳ����ļ����ص� `mContentParent` �����У�
```Java
// PhoneWindow �� --> setContentView(int layoutId) ����;
// �������Լ�д�Ĳ����ļ����ص� mConentParent ������
mLayoutInflater.inflate(layoutResID, mContentParent);
```
��ʱ `Activity` �� `onCreate()` ����ϵͳ���ֵ�Ҳ��ִ������ˣ�**��ʱ�����Ѿ������ֽ�����ϣ����ǲ�û�ж��κζ�����в��������ֺͻ��ƣ�����ֻ�Ǵ����˶�Ӧ��** `View` **���󣬲�û��ִ��** `View` **�Ļ������̷���������������ò���** `View` **�Ŀ����Ϣ�ģ���Ҳ����Ϊʲô�����޷���** `onCreate()` **�����л�ȡ�ؼ���ߵ�ԭ��**�� 

### View ���Ƶ����
[Activity��������](https://www.jianshu.com/p/c417f21d7a44) �������ڷ��� `handleResumeActivity()` �������ᵽ��һ����� `View` �Ļ������̵���û��չ�����������ǽ�������ϸ�ķ�����
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
            // ����� decor ��������ǰ������� mDecorView 
            // Ҳ����ͨ����Ϊ�� �����Ĳ���
            View decor = r.window.getDecorView();
            decor.setVisibility(View.INVISIBLE);
            // �����õ��Ķ���ʵ������ WindowManagerImpl ����
            // ����ͬ��������Դ��׷�ٵ�
            ViewManager wm = a.getWindowManager();
            WindowManager.LayoutParams l = r.window.getAttributes();
            // �����ǵ� mDecorView ���ø� Activity
            a.mDecor = decor;
            // ...
            if (a.mVisibleFromClient) {
                if (!a.mWindowAdded) {
                    a.mWindowAdded = true;
                    // TAG3
                    // �����ǽ����� mDecorView ���ص������е��ص�
                    wm.addView(decor, l);
                } 
                // ...
            }
        } 
    } 
}
```
> **TAG3** WindowManagerImpl

����������� wm ����Ϊʲô�� `WindowManagerImpl` ����ͬ��������Դ�����ҵ��������� `Activity`  �� `attach()` �������Է����������ʵ���������� `Window` ���󣬶� `Window` ��������ҵ�����һ�δ���:
```Java
// Window ��
public void setWindowManager(WindowManager wm, IBinder appToken, String appName,
        boolean hardwareAccelerated) {
    // ...
    mWindowManager = ((WindowManagerImpl)wm).createLocalWindowManager(this);
}
```
�����������ֱ�ӽ��� `WindowManagerImpl` ����鿴 `addView()` ������
```Java
// WindowManagerImpl ��
@Override
public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
    applyDefaultToken(params);
    // ��������� mGloble ����� addView() ����
    mGlobal.addView(view, params, mContext.getDisplay(), mParentWindow);
}
```
```Java
// WindowManagerGloble ��
public void addView(View view, ViewGroup.LayoutParams params,
        Display display, Window parentWindow) {
    // ...
    // ����������������ᵽ���� ViewRootImpl ����
    ViewRootImpl root;

    synchronized (mLock) {
        // ...
        // ����ʵ�����˸ö�������ͬ���Ĵ������
        root = new ViewRootImpl(view.getContext(), display);
        view.setLayoutParams(wparams);
        mViews.add(view);
        mRoots.add(root);
        mParams.add(wparams);

        try {
            // TAG4
            // ���ｫ���ǵ� mDecorView ���õ��� root ������
            root.setView(view, wparams, panelParentView);
        } catch (RuntimeException e) {
        }
    }
}
```
> TAG4 ViewRootImpl

���������Դ������֪�����������ǵ� `mDecorView` �������ձ����õ��� `ViewRootImpl` �����У����ǻ���û���п���ִ�� `View` �������̵ķ��������Ǽ������¿�Դ��:
```Java
// ViewRootImpl ��
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
    synchronized (this) {
        if (mView == null) {
            mView = view;
            // ...
            // Schedule the first layout -before- adding to the window
            // manager, to make sure we do the relayout before receiving
            // any other events from the system.
            // �����������Ӧ�ú���Ϥ�������ұ�����ԭ����ע�ͣ�
            // ��ŵ���˼���ǽ����ǵ� mDecorView ��ӵ����ֹ�������
            requestLayout();
            // ...
        }
    }
}

// ViewRootImpl ��
@Override
public void requestLayout() {
    if (!mHandlingLayoutInLayoutRequest) {
        // ...
        // ���ű������ǵ���ͼ��
        scheduleTraversals();
    }
}

// ViewRootImpl ��
// ִ�б�����ͼ��
void scheduleTraversals() {
    if (!mTraversalScheduled) {
        // ...
        // ����ܹؼ������ｫ��ִ�� mTraversalRunnable ��� Runnable �����
        // run() ���������ǿ����ҵ��������Ȼ��鿴���� run() ����
        mChoreographer.postCallback(
                Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
        // ...
    }
}

// ViewRootImpl ��
// TraversalRunnable Ϊ ViewRootImpl ���ڲ���
final class TraversalRunnable implements Runnable {
    @Override
    public void run() {
        // ��ʼ����
        doTraversal();
    }
}

// ViewRootImpl ��
void doTraversal() {
    if (mTraversalScheduled) {
        // ...
        // ��������������ǲ��Ǿ͸о���Ī������Ϥ�У�
        // ���Ϻö಩�Ͷ��Ǵ�����࿪ʼ������
        // ���Ĵ� setContentView() ��ʼ����ϵͳ�����ִ�е��������
        performTraversals();
       // ...
    }
}
```
����Ĵ������������Ǻ��ѣ���Щ��������`ViewRootImpl` ������ģ����������ҵ���һ�� `performTraversals()` ���������Ϻܶ����¶��Ǵ����������ʼ���� `View` �Ļ������̵ģ��������Ұ�����������Ϊִ��`View` �������̵���ڣ��������Ǿ�������`View` �Ļ���˳��

### View�Ļ���
ǰ�������ᵽ�� `performTraversals();`  �����ǻ���`View` ����ڣ����������ǾͿ�ʼ����������������������Դ��ܳ����кü����У��������ǵ�ֻ��Ҫ�����ǹ��ĵ��ص㼴��:
```Java
// ViewRootImpl ��
private void performTraversals() {
    // �����������ǳ��࣬�����ص����ִ������������
    // ִ�в���
    performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
    // ִ�в��֣�ViewGroup���вŻ���
    performLayout(lp, mWidth, mHeight);
    // ִ�л���
    performDraw();
}
```
��������������ֻ�ǳ�ȡ������������Ҫ�ĵ㣬�ֱ��Ӧ����һ��ʼ˵��View�Ļ��������е�`onMeasure()`��`onLayout()`��`onDraw()`�������������������ǿ�������������������α�ִ�е���;

#### onMeasure()��onLayout()��onDraw()
 �鿴 `performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);`  Դ�룬���Ƿ���������������`mDecorView` �� `measure()` ���������յ��õ�`mDecorView` �� `onMeasure()` �����õ��ؼ��Ŀ�ߣ�����Ͳ�չ���鿴 `mDecorView` ����α���ִ�����пؼ��� `measure()` ������������ר�ŷ�����һ�飬������������Ҳ��һ����
```Java
// ViewRootImpl ��
private void performMeasure(int childWidthMeasureSpec, int childHeightMeasureSpec) {
    // ...
    try {
        // ִ�� DecorView �� measure() ����
        // ������Ѿ��ǵ����ǵ� View ����ִ���ˣ����ﲻ��չ��
        mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    } finally {
    }
}
```
`performLayout(lp, mWidth, mHeight);` Դ�룬ͬ��������Ҳ������ `mDecorView` �� `layout()` ���������յ��õ� `mDecorVeiw` �� `onLayout()` ������
```Java
// ViewRootImpl ��
private void performLayout(WindowManager.LayoutParams lp, int desiredWindowWidth,
        int desiredWindowHeight) {
    // ...
    // ���ｫ DecorView ��ֵ���ֲ�����
    final View host = mView;
    // ...
    try {
        // ����ִ�����ǵ� View �� layout() ����
        host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());
        // ...
    } finally {
    }
}
```
`performDraw();`  Դ�룬������Ը���һ�㣬����Ҳ�����ҵ������� `mDecorView` �� `draw()` ���������յ��õ�`mDecorView` �� `onDraw()` ������
```Java
// ViewRootImpl ��
private void performDraw() {
    // ...
    try {
        draw(fullRedrawNeeded);
    } finally {
    }
    // ...
}

// ViewRootImpl ��
private void draw(boolean fullRedrawNeeded) {
      // ...
      // �������ȥ��
      if (!drawSoftware(surface, mAttachInfo, xOffset, yOffset, scalingRequired, dirty)) {
            return;
      }
      // ...
}

// ViewRootImpl ��
private boolean drawSoftware(Surface surface, AttachInfo attachInfo, int xoff, int yoff,
        boolean scalingRequired, Rect dirty) {
    // ...
    // ��������� mDecorView �� draw() ����
    mView.draw(canvas);
    // ...
    return true;
}
```

### �������
1. `onCreate()` �������ܻ�ȡ�� `View` �Ŀ�ߣ�
    ������걾��Ӧ�þͺܺ������,`View` �Ļ����������� `Activity` �ĵ�һ�� `onResume()`  ������ʼִ�еģ�������ò����ģ�
2. `onResume()` ������ʱ�����õ� `View` �Ŀ�ߣ���ʱ���ò�����
    ���Ҳ�ܺ���⣬����ǵ�һ��ִ�� `onResume()` ��������ô��ʱ `View` �Ļ�������Ҳû��ִ��������ò����ģ���������� `Activity` ǰ��̨�л���������� `Activity` ���� `onResume()` ����ʱ����ʱ�� `View` �Ѿ����ƹ��ˣ�����ǿ����õ���ߵģ�
3. Ϊʲô�Զ���View���� `requestLayout()` ���������� `View` ����ִ�����׻������̣�����͸��������,��Ϊ�÷���������ִ��һ�� `
performTraversals()` ����;

�����������������ͼƬ:
![UI�㼶](https://upload-images.jianshu.io/upload_images/7082912-0fac452db4960341.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![Activity�е�UI��](https://upload-images.jianshu.io/upload_images/7082912-1e2ee4d47af15d37.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

��һ��ͼƬ�����Լ����ݵڶ���ͼƬ���Ƶ�UI�㼶ͼ����ԱȽϺ���⣬������Ҫ�����µڶ���ͼƬ��
1. ���Է������ǵ� `mContentParent` ����� `mDecorView` ����֮�仹���кܶ�������;
2. ���ǵ� `ActionBar` ���󲢲����� `mContentParent` �����ж�����һ��IDΪ `action_bar_container` �������У�
3. �����ٴβ鿴`mDecorView` �ĺ��ӣ��������ǵ�״̬���͵����������亢�ӣ�������Ҫ��ȡ��״̬���ĸ߶Ⱥ͵������ĸ߶����ǿ���ͨ����ȡ `mDecorView` �����ȡ����Ȼ��� `Activity` ��ȫ���ģ����޷���ȡ��״̬���ĸ߶ȣ�

