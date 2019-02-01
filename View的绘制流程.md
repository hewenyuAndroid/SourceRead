### View的绘制流程&Activity的生命周期
刚开始学Android开发的时候，对自定义控件比较感兴趣，也经常遇到一些奇奇怪怪的问题，比如在 `Activity` 的 `onCreate()` / `onResume()` 方法中去获取 `View` 的宽高，但是这好像并不是那么的行得通；
在[Activity创建流程](https://www.jianshu.com/p/c417f21d7a44)中我们分析了 `Activity` 创建时生命周期方法的创建时机，同样`View`也有自己的一套执行流程 `onMeasure()` 、`onLayout()` 、`onDraw()` ，`View` 的这一套流程和 `Activity` 的创建流程有什么联系，为什么我们在 `Activity` 的 `onCreate()` 方法中无法获取到 `View` 的宽高，在 `onResume()` 方法中有时候能获取到宽高，有时候又不可以，下面我们来通过源码进行分析；

### setContentView()加载布局文件
在创建Activity的时候，我们通常都会在其 `onCreate()` 方法中调用 `setContentView();` 方法将我们写好的布局文件加载成对应的`View` 对象，关于调用此方法后是如何将我们的 XML 布局解析成视图杜对象的可以查看我的这篇文章 [Android 布局文件加载 LayoutInflater 源码解析](https://www.jianshu.com/p/846c36af2c2b)，这里简单的介绍下就是通过 XML 文件解析器拿到对应的View的全类名，最后通过反射生成我们的对象；
我们深入 `setContentView()` 方法的源码，发现其最总是调用了 `Window` 的 `setContentView()` 方法，前面我们在[Activity创建流程](https://www.jianshu.com/p/c417f21d7a44)中我们分析了 `Activity` 在通过反射创建对象后会调用 `Activity` 的 `attach()` 方法来初始化相关信息，这其中就包括 `Window`  对象，而且是其子类 `PhoneWindow` 对象：
```Java
// Activity 类
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
接下来我们进入到 `PhoneWindow` 对象的 `setContentView()` 方法中分析下源码:
```Java
// PhoneWindow 类
@Override
public void setContentView(int layoutResID) {
    if (mContentParent == null) {
        // 创建 DecorView
        installDecor();
    } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        mContentParent.removeAllViews();
    }

    if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
                getContext());
        transitionTo(newScene);
    } else {
        // 将我们的XML解析出来的View加载到系统布局中
        mLayoutInflater.inflate(layoutResID, mContentParent);
    }
    // ...
}
```
我们先来看下 `installDecor();` 方法，这个方法用于创建 `DecorView` 和 `mContentParent` 对象，后面我们会对这两个对象详细分析，这里先来看下这个方法的源码:
```Java
// PhoneWindow 类
private void installDecor() {
    // ...
    if (mDecor == null) {
        // TAG1
        // 创建 DecorView 对象
        mDecor = generateDecor(-1);
        // ...
    } else {
        mDecor.setWindow(this);
    }
    if (mContentParent == null) {
        // TAG2
        // 创建 mContentParent 对象
        mContentParent = generateLayout(mDecor);
        // ...
    }
}
```
> **TAG1** mDecor 对象的创建

这里创建了一个 `DecorView` ，它是一个 `FrameLayout` 类型的对象。网上看了很多博客，大多是把 `DecorView` 当作是Activity中的根布局，但是它最总是被加载到一个实现了 `ViewParent` 接口的 `ViewRootImpl` 对象中，然后才被加载到Window 对象上（后面我们会分析这一块内容），所以感觉这句话也不是那么的贴切；
```Java
// PhoneWindow 类
protected DecorView generateDecor(int featureId) {
    // ...
    // 这里直接通过 new 对象创建了 DecorView 对象
    return new DecorView(context, featureId, this, getAttributes());
}
```

> **TAG2** mContentParent 对象的创建

从这个参数的名称中我们能够猜测这个对象应该就是我们 XML 布局的父布局，而 `mContentParent` 对象的父类还不是 `mDecor` 对象，它的外面还包裹了一层 `ViewGroup` 对象，用于设置标题等，查看不同的布局文件，我们可以发现这里的 `mContentParent` 对象其实是一个固定 Id = **com.android.internal.R.id.content** 的FrameLayout 对象，拿到这个ID其实我们可以做很多事情，例如: API版本为 19(4.4) 到 21(5.0)之间的沉浸式状态栏的设置等；
```Java
// PhoneWindow 类
protected ViewGroup generateLayout(DecorView decor) {
    // ...
    // 加载对应的布局文件，这里的布局文件将会根据我们设置的属性来加载
    // 例如: noTitle 的布局等
    int layoutResource;
    int features = getLocalFeatures();
    // System.out.println("Features: 0x" + Integer.toHexString(features));
    if ((features & (1 << FEATURE_SWIPE_TO_DISMISS)) != 0) {
        layoutResource = R.layout.screen_swipe_dismiss;
        setCloseOnSwipeEnabled(true);
    }else if(){
        // ...
        // 这里具体的代码就不贴上来了，大家可以自己查看源码
        // 我看的版本的 26
    } else {
        layoutResource = R.layout.screen_simple;
    }
    
    // 将对应属性的布局文件加载到 DecorView 中
    mDecor.onResourcesLoaded(mLayoutInflater, layoutResource);
    // 根据Id获取 contentParent对象，此 Id 固定为: com.android.internal.R.id.content
    // 这里的 contentParent 对象其实为 FrameLayout 类型的对象
    ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);
    // ...
    return contentParent;
}
```
执行到这里我们的 `mContentParent` 对象已经创建完毕，然后回返回到我们上面提到过的 `PhoneWindow` 的`setConentView()` 方法中将我们自己写的布局文件加载到 `mContentParent` 对象中；
```Java
// PhoneWindow 类 --> setContentView(int layoutId) 方法;
// 将我们自己写的布局文件加载到 mConentParent 对象中
mLayoutInflater.inflate(layoutResID, mContentParent);
```
此时 `Activity` 的 `onCreate()` 方法系统部分的也就执行完毕了，**此时我们已经将布局解析完毕，但是并没有对任何对象进行测量，布局和绘制，仅仅只是创建了对应的** `View` **对象，并没有执行** `View` **的绘制流程方法，因此我们是拿不到** `View` **的宽高信息的，这也就是为什么我们无法在** `onCreate()` **方法中获取控件宽高的原因**； 

### View 绘制的入口
[Activity创建流程](https://www.jianshu.com/p/c417f21d7a44) 中我们在分析 `handleResumeActivity()` 方法中提到过一点关于 `View` 的绘制流程但是没有展开，这里我们将进行详细的分析；
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
            // 这里的 decor 就是我们前面分析的 mDecorView 
            // 也就是通常认为的 最外层的布局
            View decor = r.window.getDecorView();
            decor.setVisibility(View.INVISIBLE);
            // 这里拿到的对象实际上是 WindowManagerImpl 对象
            // 这里同样可以在源码追踪到
            ViewManager wm = a.getWindowManager();
            WindowManager.LayoutParams l = r.window.getAttributes();
            // 将我们的 mDecorView 设置给 Activity
            a.mDecor = decor;
            // ...
            if (a.mVisibleFromClient) {
                if (!a.mWindowAdded) {
                    a.mWindowAdded = true;
                    // TAG3
                    // 这里是将我们 mDecorView 加载到窗口中的重点
                    wm.addView(decor, l);
                } 
                // ...
            }
        } 
    } 
}
```
> **TAG3** WindowManagerImpl

关于这里面的 wm 对象为什么是 `WindowManagerImpl` 对象同样可以在源码中找到，我们在 `Activity`  的 `attach()` 方法可以发现这个对象实际上是来自 `Window` 对象，而 `Window` 里面可以找到这样一段代码:
```Java
// Window 类
public void setWindowManager(WindowManager wm, IBinder appToken, String appName,
        boolean hardwareAccelerated) {
    // ...
    mWindowManager = ((WindowManagerImpl)wm).createLocalWindowManager(this);
}
```
因此我们这里直接进入 `WindowManagerImpl` 对象查看 `addView()` 方法；
```Java
// WindowManagerImpl 类
@Override
public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
    applyDefaultToken(params);
    // 这里调用了 mGloble 对象的 addView() 方法
    mGlobal.addView(view, params, mContext.getDisplay(), mParentWindow);
}
```
```Java
// WindowManagerGloble 类
public void addView(View view, ViewGroup.LayoutParams params,
        Display display, Window parentWindow) {
    // ...
    // 这里就是上面我们提到过的 ViewRootImpl 对象
    ViewRootImpl root;

    synchronized (mLock) {
        // ...
        // 这里实例化了该对象，是在同步的代码块中
        root = new ViewRootImpl(view.getContext(), display);
        view.setLayoutParams(wparams);
        mViews.add(view);
        mRoots.add(root);
        mParams.add(wparams);

        try {
            // TAG4
            // 这里将我们的 mDecorView 设置到了 root 对象中
            root.setView(view, wparams, panelParentView);
        } catch (RuntimeException e) {
        }
    }
}
```
> TAG4 ViewRootImpl

看了上面的源码我们知道了最终我们的 `mDecorView` 对象最终被设置到了 `ViewRootImpl` 对象中，但是还是没看有看到执行 `View` 绘制流程的方法，我们继续往下看源码:
```Java
// ViewRootImpl 类
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
    synchronized (this) {
        if (mView == null) {
            mView = view;
            // ...
            // Schedule the first layout -before- adding to the window
            // manager, to make sure we do the relayout before receiving
            // any other events from the system.
            // 这个方法我们应该很熟悉，这里我保留了原来的注释，
            // 大概的意思就是将我们的 mDecorView 添加到布局管理器中
            requestLayout();
            // ...
        }
    }
}

// ViewRootImpl 类
@Override
public void requestLayout() {
    if (!mHandlingLayoutInLayoutRequest) {
        // ...
        // 安排遍历我们的视图树
        scheduleTraversals();
    }
}

// ViewRootImpl 类
// 执行遍历视图树
void scheduleTraversals() {
    if (!mTraversalScheduled) {
        // ...
        // 这里很关键，这里将会执行 mTraversalRunnable 这个 Runnable 对象的
        // run() 方法，我们可以找到这个对象，然后查看它的 run() 方法
        mChoreographer.postCallback(
                Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
        // ...
    }
}

// ViewRootImpl 类
// TraversalRunnable 为 ViewRootImpl 的内部类
final class TraversalRunnable implements Runnable {
    @Override
    public void run() {
        // 开始遍历
        doTraversal();
    }
}

// ViewRootImpl 类
void doTraversal() {
    if (mTraversalScheduled) {
        // ...
        // 看到这个方法类是不是就感觉到莫名的熟悉感，
        // 网上好多博客都是从这个类开始分析的
        // 本文从 setContentView() 开始分析系统是如何执行到这个方法
        performTraversals();
       // ...
    }
}
```
上面的代码找起来不是很难，这些方法都是`ViewRootImpl` 类里面的，最终我们找到了一个 `performTraversals()` 方法，网上很多文章都是从这个方法开始讲解 `View` 的绘制流程的，我们暂且把这个方法理解为执行`View` 绘制流程的入口，下面我们就来分析`View` 的绘制顺序；

### View的绘制
前面我们提到了 `performTraversals();`  方法是绘制`View` 的入口，接下来我们就开始分析这个方法，这个方法的源码很长，有好几百行，但是我们的只需要看我们关心的重点即可:
```Java
// ViewRootImpl 类
private void performTraversals() {
    // 这个方法代码非常多，但是重点就是执行这三个方法
    // 执行测量
    performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
    // 执行布局（ViewGroup）中才会有
    performLayout(lp, mWidth, mHeight);
    // 执行绘制
    performDraw();
}
```
上述代码中我们只是抽取出来了三个重要的点，分别对应我们一开始说的View的绘制流程中的`onMeasure()`、`onLayout()`、`onDraw()`三个方法，解析来我们看下这三个方法都是如何被执行到的;

#### onMeasure()、onLayout()、onDraw()
 查看 `performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);`  源码，我们发现他调用了我们`mDecorView` 的 `measure()` 方法，最终调用到`mDecorView` 的 `onMeasure()` 方法得到控件的宽高；这里就不展开查看 `mDecorView` 是如何遍历执行所有控件的 `measure()` 方法，后续会专门分析这一块，其它两个方法也是一样；
```Java
// ViewRootImpl 类
private void performMeasure(int childWidthMeasureSpec, int childHeightMeasureSpec) {
    // ...
    try {
        // 执行 DecorView 的 measure() 方法
        // 这里就已经是到我们的 View 里面执行了，这里不做展开
        mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    } finally {
    }
}
```
`performLayout(lp, mWidth, mHeight);` 源码，同样这里面也调用了 `mDecorView` 的 `layout()` 方法，最终调用到 `mDecorVeiw` 的 `onLayout()` 方法；
```Java
// ViewRootImpl 类
private void performLayout(WindowManager.LayoutParams lp, int desiredWindowWidth,
        int desiredWindowHeight) {
    // ...
    // 这里将 DecorView 赋值到局部变量
    final View host = mView;
    // ...
    try {
        // 这里执行我们的 View 的 layout() 方法
        host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());
        // ...
    } finally {
    }
}
```
`performDraw();`  源码，这里相对复杂一点，不过也是能找到调用了 `mDecorView` 的 `draw()` 方法，最终调用到`mDecorView` 的 `onDraw()` 方法；
```Java
// ViewRootImpl 类
private void performDraw() {
    // ...
    try {
        draw(fullRedrawNeeded);
    } finally {
    }
    // ...
}

// ViewRootImpl 类
private void draw(boolean fullRedrawNeeded) {
      // ...
      // 继续点进去看
      if (!drawSoftware(surface, mAttachInfo, xOffset, yOffset, scalingRequired, dirty)) {
            return;
      }
      // ...
}

// ViewRootImpl 类
private boolean drawSoftware(Surface surface, AttachInfo attachInfo, int xoff, int yoff,
        boolean scalingRequired, Rect dirty) {
    // ...
    // 这里调用了 mDecorView 的 draw() 方法
    mView.draw(canvas);
    // ...
    return true;
}
```

### 问题解析
1. `onCreate()` 方法不能获取到 `View` 的宽高；
    这个看完本文应该就很好理解了,`View` 的绘制流程是在 `Activity` 的第一次 `onResume()`  方法后开始执行的，因此是拿不到的；
2. `onResume()` 方法有时候能拿到 `View` 的宽高，有时候拿不到；
    这个也很好理解，如果是第一次执行 `onResume()` 方法，那么此时 `View` 的绘制流程也没有执行因此是拿不到的，但是如果是 `Activity` 前后台切换等情况触发 `Activity` 调用 `onResume()` 方法时，此时的 `View` 已经绘制过了，因此是可以拿到宽高的；
3. 为什么自定义View调用 `requestLayout()` 方法可以让 `View` 重新执行整套绘制流程，这个就更好理解了,因为该方法会重新执行一遍 `
performTraversals()` 方法;

最后我们再来看两张图片:
![UI层级](https://upload-images.jianshu.io/upload_images/7082912-0fac452db4960341.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![Activity中的UI树](https://upload-images.jianshu.io/upload_images/7082912-1e2ee4d47af15d37.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

第一张图片是我自己根据第二张图片绘制的UI层级图，相对比较好理解，我们主要分析下第二张图片：
1. 可以发下我们的 `mContentParent` 对象和 `mDecorView` 对象之间还是有很多层包裹的;
2. 我们的 `ActionBar` 对象并不是在 `mContentParent` 对象中而是在一个ID为 `action_bar_container` 的容器中；
3. 我们再次查看`mDecorView` 的孩子，发现我们的状态栏和导航栏都是其孩子，因此如果要获取到状态栏的高度和导航栏的高度我们可以通过获取 `mDecorView` 对象获取，当然如果 `Activity` 是全屏的，则无法获取到状态栏的高度；

