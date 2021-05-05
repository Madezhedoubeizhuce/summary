# Activity将事件分发到根View中

在decorview创建流程和Activity如何接收输入事件的分析中我们知道了，Activity的根Viewt是DecorView，并且底层的输入事件被分发到Activity的dispatchTouchEvent方法中，那么Activity又是如何将触摸事件分发到ContentView中呢？
首先看一下Activity的dispatchTouchEvent：

```java
public boolean dispatchTouchEvent(MotionEvent ev) {
    if (ev.getAction() == MotionEvent.ACTION_DOWN) {
        onUserInteraction();
    }
    if (getWindow().superDispatchTouchEvent(ev)) {
        return true;
    }
    return onTouchEvent(ev);
}
```

如果当前事件是ACTION_DOWN事件，则调用onUserInteraction，已通知Activity当前用户正在交互，可以进行一些特殊的处理。然后会调用window的superDispatchTouchEvent方法，经前面的分析我们知道这里的window是PhoneWindow对象，所以直接查看PhoneWIndow的superDispatchTouchEvent：

```java
@Override
public boolean superDispatchTouchEvent(MotionEvent event) {
    return mDecor.superDispatchTouchEvent(event);
}

// DecorView
public boolean superDispatchTouchEvent(MotionEvent event) {
    return super.dispatchTouchEvent(event);
}
```

PhoneWIndow的superDispatchTouchEvent直接调用了DecorView的superDispatchTouchEvent，而DecorView直接调用了父类的dispatchTouchEvent，因为DecorView的父类是FrameLayout，而FrameLayout没有重写dispatchTouchEvent，因此这里就是直接调用ViewGroup的dispatchTouchEvent方法。

到这里为止，Activity就将事件分发到根View中了，总结一下，当事件到达DecorView的时候，他并没有直接开始view层级的分发流程，而是先把事件转发到Activity中，再由Activity把事件转到superDispatchTouchEvent中，在superDispatchTouchEvent中才开始view层级的事件分发流程，这样做就能吧Activity当做顶层View，进程一些特殊的处理。