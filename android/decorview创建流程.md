# decorview创建流程

## window创建

启动Activity时，最终会通过ActivityThread的handleLaunchActivity方法创建Activity并调用相关的生命周期方法，Activity创建之后会通过Activity的attach方法来创建window：

先看一下ActivityThread的handleLaunchActivity：

```java
private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    ...

    // Initialize before creating the activity
    WindowManagerGlobal.initialize();

    Activity a = performLaunchActivity(r, customIntent);
    if (a != null) {
        ...
        handleResumeActivity(r.token, false, r.isForward,
                             !r.activity.mFinished && !r.startsNotResumed);
        ...
    }
    ...
}
```

这里先通过WindowManagerGlobal.initialize()将WindowManagerGlobal初始化，initialize方法内部会获取并保存wms的实例。然后通过performLaunchActivity启动Activity方法，并调用Activity的attach方法初始化window。

```java
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    ...

    try {
        Application app = r.packageInfo.makeApplication(false, mInstrumentation);

        if (activity != null) {
            Context appContext = createBaseContextForActivity(r, activity);
            ...
            
            // 这里会创建PhoneWindow实例
            activity.attach(appContext, this, getInstrumentation(), r.token,
                            r.ident, app, r.intent, r.activityInfo, title, r.parent,
                            r.embeddedID, r.lastNonConfigurationInstances, config,
                            r.referrer, r.voiceInteractor);

            ...
            if (r.isPersistable()) {
                mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
            } else {
                mInstrumentation.callActivityOnCreate(activity, r.state);
            }
            ...
}
```

performLaunchActivity内部会先创建Activity对象，并创建或获取Application及BaseContext实例，然后会调用

activity.attach方法创建Window的实例，最后调用Activity的onCreate方法。

```java
final void attach(Context context,..., ActivityInfo info,...) {
    ...

    mWindow = new PhoneWindow(this);
    mWindow.setCallback(this);
    mWindow.setOnWindowDismissedCallback(this);
    mWindow.getLayoutInflater().setPrivateFactory(this);
    ...

    mWindow.setWindowManager(
        (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
        mToken, mComponent.flattenToString(),
        (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
    ...
    mWindowManager = mWindow.getWindowManager();
    ...
}
```

attach中首先创建了一个PhoneWindow实例，由于Activity实现了Window.Callback接口，因此可以将当前的Activity设置到Window中。然后通过window的setWindowManager方法，将WindowManager和PhoneWindow绑定：

```java
public void setWindowManager(WindowManager wm, IBinder appToken, String appName,
                             boolean hardwareAccelerated) {
    mAppToken = appToken;
    mAppName = appName;
    mHardwareAccelerated = hardwareAccelerated
        || SystemProperties.getBoolean(PROPERTY_HARDWARE_UI, false);
    if (wm == null) {
        wm = (WindowManager)mContext.getSystemService(Context.WINDOW_SERVICE);
    }
    mWindowManager = ((WindowManagerImpl)wm).createLocalWindowManager(this);
}
```

从最后一行可以看出，这里通过将WindowManager封装成WindowManagerImpl对象将PhoneWindow和WindowManager绑定的。

## DecorView创建及用户布局加载

回到performLaunchActivity方法中，在调用attach之后会回调Activity的onCreate方法，在onCreate中我们一般都会使用setContentView来设置布局文件，因此接下去看一下Activity的setContentView方法：

```java
public void setContentView(@LayoutRes int layoutResID) {
    getWindow().setContentView(layoutResID);
    initWindowDecorActionBar();
}
```

可以看出，这里又调用了window的setContentView设置布局，经过上一节的分析我们知道，Activity的window就是PhoneWindow，因此我们看一下PhoneWindow的setContentView：

```java
public void setContentView(int layoutResID) {
    // Note: FEATURE_CONTENT_TRANSITIONS may be set in the process of installing the window
    // decor, when theme attributes and the like are crystalized. Do not check the feature
    // before this happens.
    if (mContentParent == null) {
        installDecor();
    } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        mContentParent.removeAllViews();
    }

    if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
                                                       getContext());
        transitionTo(newScene);
    } else {
        mLayoutInflater.inflate(layoutResID, mContentParent); // -----1
    }
    mContentParent.requestApplyInsets();
    final Callback cb = getCallback();
    if (cb != null && !isDestroyed()) {
        cb.onContentChanged();
    }
}
```

首先如果mContentParent为空，则通过installDecor来创建DecorView并加载布局。installDecor内部会先创建DecorView实例，再用LayoutInflater创建DecorView的布局，这里就不贴相关的代码了，感兴趣的可以自己看一下。

一般情况下DecorView的布局可以分为Title和ContentView两部分，其中ContentView就是的mContentParent，它是一个ID为android.R.id.content的FrameLayout，用户指定的布局文件都是加载到ContentView中的。DecorView的布局大致长这样：

![decorView](.\img\decorview.image)

installDecor()创建完DecorView之后，在注释“-----1”处通过LayoutInflater将用户指定的布局加载到ContentView中，这样就将用户指定的布局文件加载到DecorView中了。

## 将DecorView添加到屏幕

前面两节介绍了Activity的create流程中会创建PhoneWindow和DecorView，但还没有介绍创建的DecorView是如何添加到Activity中的，下面就来介绍一下。

先回顾一下ActivityThread的handleLaunchActivity方法：

```java
private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    ...

    // Initialize before creating the activity
    WindowManagerGlobal.initialize();

    Activity a = performLaunchActivity(r, customIntent);
    if (a != null) {
        ...
        handleResumeActivity(r.token, false, r.isForward,
                             !r.activity.mFinished && !r.startsNotResumed);
        ...
    }
    ...
}
```

前面分析过，performLaunchActivity中会启动Activity、调用onCreate方法，从而创建Window和DecorView，如果你仔细分析performLaunchActivity，会发现找不到将DecorView添加到界面显示的逻辑，那是因为performLaunchActivity主要是负责启动Activity，并不负责显示页面，页面的显示都在handleResumeActivity中处理，下面来分析一下handleResumeActivity：

```java
final void handleResumeActivity(IBinder token,boolean clearHide, 
                                boolean isForward, boolean reallyResume) {
    ...
    ActivityClientRecord r = performResumeActivity(token, clearHide);

    if (r != null) {
        ...    
        if (r.window == null && !a.mFinished && willBeVisible) {
            r.window = r.activity.getWindow();
            View decor = r.window.getDecorView();
            decor.setVisibility(View.INVISIBLE);
            ViewManager wm = a.getWindowManager();
            WindowManager.LayoutParams l = r.window.getAttributes();
            a.mDecor = decor;
            l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
            l.softInputMode |= forwardBit;
            if (a.mVisibleFromClient) {
                a.mWindowAdded = true;
                wm.addView(decor, l);
            }
        }
        ...
}
```

handleResumeActivity中首先通过performResumeActivity回调Activity的onResume方法，然后获取之前创建的DecorView，通过WindowManager的addView方法添加到界面显示。

## WindowManager添加DecorView

上一节我们讲到DecorView最后是通过WindowManager的addView被添加到界面的，这里其实DecorView其实是作为一个window添加到WindowManager中的，为什么要这么做呢？我们知道，应用中会存在Activity、dialog以及popupWindow等多种类型的页面，这些页面有着各自不同的层级关系，比如dialog总是显示在Activity的上面。因此使用window就能方便的管理层级次序。

那么WindowManager是如何添加DecorView的呢，从Window的创建流程中我们知道了Activity中的WindowManager是WIndowManagerImpl的实例，因此我们先看一下WIndowManagerImpl的addView：

```java
@Override
public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
    applyDefaultToken(params);
    mGlobal.addView(view, params, mDisplay, mParentWindow);
}
```

可以看出，这里调用了mGlobal的addView将DecorView添加到界面中，这里的mGlobal其实是一个WindowManagerGlobal的实例，我们继续查看它的addView实现：

```java
public void addView(View view, ViewGroup.LayoutParams params,
                    Display display, Window parentWindow) {
    

    ViewRootImpl root;
    View panelParentView = null;

    synchronized (mLock) {
        ...

        root = new ViewRootImpl(view.getContext(), display);

        view.setLayoutParams(wparams);

        mViews.add(view);
        mRoots.add(root);
        mParams.add(wparams);
    }

    try {
        root.setView(view, wparams, panelParentView);
    } catch (RuntimeException e) {
        ...
        throw e;
    }
}
```

这里首先创建了一个ViewRootImpl的实例root，然后将root、view和wparams分别保存到本地的列表中，最后调用ViewRootImpl的setView方法将view添加到windowmanger中，它的实现如下：

```java
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
    synchronized (this) {
        if (mView == null) {
            mView = view;

            ...

            // Schedule the first layout -before- adding to the window
            // manager, to make sure we do the relayout before receiving
            // any other events from the system.
            requestLayout();
            ...
            try {
                ...
                res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
                                                  getHostVisibility(), 		                       										   mDisplay.getDisplayId(),
                                                  mAttachInfo.mContentInsets, 	        												  mAttachInfo.mStableInsets,
                                                  mAttachInfo.mOutsets, mInputChannel);
            } catch (RemoteException e) {
                ...
                throw new RuntimeException("Adding window failed", e);
            } finally {
                ...
            }

            ...
        }
    }
}
```

首先，通过requestLayout触发view的绘制流程，然后通过mWindowSession.addToDisplay将

mWindow添加到windowmangerservice中。mWindow是ViewRootImple的一个内部类W的实例，类W继承了IWindow.Stub，W的定义如下：

```java
static class W extends IWindow.Stub {
    ...
}
```

到这里，DecorView也就被当做一个Window添加到WindowManager中了，整体的DecorView加载流程也就完了。

## 总结

最后总结一下Activity中window和decorview的加载流程：

- 首先，在Activity的attach方法中创建PhoneWindow实例和封装WindowManagerImple实例。

- 然后，在onCreate方法中调用setContentView来创建并加载DecorView。
- 最后，在ActivityThread的handleResumeActivity中将decorview作为window添加到windowManager中。