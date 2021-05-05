# Activity如何接收输入事件

很多介绍android事件分发的文章都说事件是从Activity的dispatchTouchEvent开始的，都没有说明白底层的输入事件时怎么分发到Activity的，是Activity直接监听了底层的输入事件吗，其实不是的，底层的输入事件是在ViewRootImpl中接收的，然后ViewRootImpl将其分发到Activity，最后才由Activity进行分发的。下面就先来看一下ViewRootImpl是如何接收底层的输入事件的。

## ViewRootImpl接收输入事件

在DecorView的创建流程中我们说到WindowManagerGlobal会创建ViewRootImpl并通过ViewRootImpl的setView将view添加到windowManager中，其实ViewRootImpl的setView还设置了InputEventReceiver来接收底层的输入事件：

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
            if ((mWindowAttributes.inputFeatures
                 & WindowManager.LayoutParams.INPUT_FEATURE_NO_INPUT_CHANNEL) == 0) {
                mInputChannel = new InputChannel();
            }
            try {
                ...
                res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
                                                  getHostVisibility(), 	   	  		        											   mDisplay.getDisplayId(),
                                                  mAttachInfo.mContentInsets,             													mAttachInfo.mStableInsets,
                                                  mAttachInfo.mOutsets, mInputChannel);
            } catch (RemoteException e) {
                ...
                throw new RuntimeException("Adding window failed", e);
            } finally {
                ...
            }

            ...
            if (mInputChannel != null) {
                if (mInputQueueCallback != null) {
                    mInputQueue = new InputQueue();
                    mInputQueueCallback.onInputQueueCreated(mInputQueue);
                }
                mInputEventReceiver = new WindowInputEventReceiver(mInputChannel,
                                                                   Looper.myLooper());
            }

            ...

            // Set up the input pipeline.
            CharSequence counterSuffix = attrs.getTitle();
            mSyntheticInputStage = new SyntheticInputStage();
            InputStage viewPostImeStage = new ViewPostImeInputStage(mSyntheticInputStage);
            InputStage nativePostImeStage = new NativePostImeInputStage(viewPostImeStage,
                                                                        "aq:native-post-ime:" + counterSuffix);
            InputStage earlyPostImeStage = new EarlyPostImeInputStage(nativePostImeStage);
            InputStage imeStage = new ImeInputStage(earlyPostImeStage,
                                                    "aq:ime:" + counterSuffix);
            InputStage viewPreImeStage = new ViewPreImeInputStage(imeStage);
            InputStage nativePreImeStage = new NativePreImeInputStage(viewPreImeStage,
                                                                      "aq:native-pre-ime:" + counterSuffix);

            mFirstInputStage = nativePreImeStage;
            mFirstPostImeInputStage = earlyPostImeStage;
            mPendingInputEventQueueLengthCounterName = "aq:pending:" + counterSuffix;
        }
    }
}
```

这里主要添加了setView方法中与输入事件相关的代码，在requestLayout触发view绘制流程之后，会创建inputChannel实例，inputChannel用于传递底层的输入事件。在添加当前的window到wms之后，会使用刚创建的inputChannel实例来创建一个WindowInputEventReceiver用于接收输入事件。接着会创建一系列的InputState对象用于处理不同的输入事件，一般view层的touchevent事件是由ViewPostImeInputStage对象处理的，这个稍后进行分析。下面先来看一下WindowInputEventReceiver的实现：

```java
final class WindowInputEventReceiver extends InputEventReceiver {
    public WindowInputEventReceiver(InputChannel inputChannel, Looper looper) {
        super(inputChannel, looper);
    }

    @Override
    public void onInputEvent(InputEvent event) {
        enqueueInputEvent(event, this, 0, true);
    }

    @Override
    public void onBatchedInputEventPending() {
        if (mUnbufferedInputDispatch) {
            super.onBatchedInputEventPending();
        } else {
            scheduleConsumeBatchedInput();
        }
    }

    @Override
    public void dispose() {
        unscheduleConsumeBatchedInput();
        super.dispose();
    }
}
```

构造器中调用了父类InputEventReceiver的构造器，在父类的构造器中主要是调用nativeInit进行初始化，然后在底层发生输入事件时就会回调onInputEvent进行处理。

## ViewRootImpl处理输入事件

上一节分析了ViewRootImpl会注册WindowInputEventReceiver来接收底层输入事件，那么接收到输入事件之后是如何处理的呢？前面简单介绍过是由setView最后注册的一系列inputstate对象来处理的，接下去我们就详细分析一下，先回到ViewRootImpl的setView方法中回顾一下：

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
            if ((mWindowAttributes.inputFeatures
                 & WindowManager.LayoutParams.INPUT_FEATURE_NO_INPUT_CHANNEL) == 0) {
                mInputChannel = new InputChannel();
            }
            try {
                ...
                res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
                                                  getHostVisibility(), 	   	  		        											   mDisplay.getDisplayId(),
                                                  mAttachInfo.mContentInsets,             													mAttachInfo.mStableInsets,
                                                  mAttachInfo.mOutsets, mInputChannel);
            } catch (RemoteException e) {
                ...
                throw new RuntimeException("Adding window failed", e);
            } finally {
                ...
            }

            ...
            if (mInputChannel != null) {
                if (mInputQueueCallback != null) {
                    mInputQueue = new InputQueue();
                    mInputQueueCallback.onInputQueueCreated(mInputQueue);
                }
                mInputEventReceiver = new WindowInputEventReceiver(mInputChannel,
                                                                   Looper.myLooper());
            }

            ...

            // Set up the input pipeline.
            CharSequence counterSuffix = attrs.getTitle();
            mSyntheticInputStage = new SyntheticInputStage();
            InputStage viewPostImeStage = new ViewPostImeInputStage(mSyntheticInputStage);
            InputStage nativePostImeStage = new NativePostImeInputStage(viewPostImeStage,
                                                                        "aq:native-post-ime:" + counterSuffix);
            InputStage earlyPostImeStage = new EarlyPostImeInputStage(nativePostImeStage);
            InputStage imeStage = new ImeInputStage(earlyPostImeStage,
                                                    "aq:ime:" + counterSuffix);
            InputStage viewPreImeStage = new ViewPreImeInputStage(imeStage);
            InputStage nativePreImeStage = new NativePreImeInputStage(viewPreImeStage,
                                                                      "aq:native-pre-ime:" + counterSuffix);

            mFirstInputStage = nativePreImeStage;
            mFirstPostImeInputStage = earlyPostImeStage;
            mPendingInputEventQueueLengthCounterName = "aq:pending:" + counterSuffix;
        }
    }
}
```

可以看到在InputReceiver实例被创建之后，就创建了SyntheticInputStage、ViewPostImeInputStage等InputState，并将最后的nativePreImeStage保存为mFirstInputStage。那么我们先来看一下InputState类的定义，再查看下几个主要实现类的定义：

```java
/**
  * Base class for implementing a stage in the chain of responsibility
  * for processing input events.
  * <p>
  * Events are delivered to the stage by the {@link #deliver} method.  The stage
  * then has the choice of finishing the event or forwarding it to the next stage.
  * </p>
  */
abstract class InputStage {
    private final InputStage mNext;

    protected static final int FORWARD = 0;
    protected static final int FINISH_HANDLED = 1;
    protected static final int FINISH_NOT_HANDLED = 2;

    /**
      * Creates an input stage.
      * @param next The next stage to which events should be forwarded.
      */
    public InputStage(InputStage next) {
        mNext = next;
    }

    /**
      * Delivers an event to be processed.
      */
    public final void deliver(QueuedInputEvent q) {
        ...
    }

    /**
      * Marks the the input event as finished then forwards it to the next stage.
      */
    protected void finish(QueuedInputEvent q, boolean handled) {
        ...
    }

    /**
      * Forwards the event to the next stage.
      */
    protected void forward(QueuedInputEvent q) {
        ...
    }

    /**
      * Applies a result code from {@link #onProcess} to the specified event.
      */
    protected void apply(QueuedInputEvent q, int result) {
        ...
    }

    /**
      * Called when an event is ready to be processed.
      * @return A result code indicating how the event was handled.
      */
    protected int onProcess(QueuedInputEvent q) {
        return FORWARD;
    }

    /**
      * Called when an event is being delivered to the next stage.
      */
    protected void onDeliverToNext(QueuedInputEvent q) {
        ...
    }

    ...
}
```

从类注释可以知道InputStage使用责任链模式处理输入事件，使用deliver方法传递事件，state可以选择结束当前事件或者将事件传递到下一个状态进行处理。下面来分析一下主要的方法实现：

```java
public final void deliver(QueuedInputEvent q) {
    if ((q.mFlags & QueuedInputEvent.FLAG_FINISHED) != 0) {
        forward(q);
    } else if (shouldDropInputEvent(q)) {
        finish(q, false);
    } else {
        apply(q, onProcess(q));
    }
}
```

deliver中先判断当前输入事件是否已经完成，也就是说当前事件是否已经被处理过了，如果被处理过了，就直接调用forward方法将其传递到下一个状态中，否则就判断当前事件是否需要丢弃，如果需要丢弃，则直接调用finish方法将事件标记成finish状态。如果事件及为结束也不能丢弃，就调用onProcess处理该事件，并使用apply处理处理结果。我们先看一下forward的实现：

```java
protected void forward(QueuedInputEvent q) {
    onDeliverToNext(q);
}

protected void onDeliverToNext(QueuedInputEvent q) {
    if (DEBUG_INPUT_STAGES) {
        Log.v(TAG, "Done with " + getClass().getSimpleName() + ". " + q);
    }
    if (mNext != null) {
        mNext.deliver(q);
    } else {
        finishInputEvent(q);
    }
}
```

forward直接将事件交由onDeliverToNext方法处理，onDeliverToNext的逻辑也比较简单，就判断是否还有下一个状态，如果有的话就将事件交由下一个状态的deliver方法处理，否则就用finishInputEvent结束事件。

接下去我们再看一下finish的实现：

```java
protected void finish(QueuedInputEvent q, boolean handled) {
    q.mFlags |= QueuedInputEvent.FLAG_FINISHED;
    if (handled) {
        q.mFlags |= QueuedInputEvent.FLAG_FINISHED_HANDLED;
    }
    forward(q);
}
```

finish中设置事件的FINISH标志位，并将事件转发到下一状态中。

最后我们来看下onProcess和apply的实现：

```java
protected int onProcess(QueuedInputEvent q) {
    return FORWARD;
}

protected void apply(QueuedInputEvent q, int result) {
    if (result == FORWARD) {
        forward(q);
    } else if (result == FINISH_HANDLED) {
        finish(q, true);
    } else if (result == FINISH_NOT_HANDLED) {
        finish(q, false);
    } else {
        throw new IllegalArgumentException("Invalid result: " + result);
    }
}
```

这里onprocess默认返回FORWARD，表示当前事件没有被处理，apply方法则根据处理结果，调用forward或者finish方法传递事件。

InputState子类几个子类定义如下：

```java
/**
  * Performs synthesis of new input events from unhandled input events.
  */
final class SyntheticInputStage extends InputStage {
    ...
}

/**
  * Delivers post-ime input events to the view hierarchy.
  */
final class ViewPostImeInputStage extends InputStage {
    ...
}

/**
  * Performs early processing of post-ime input events.
  */
final class EarlyPostImeInputStage extends InputStage {
    ...
}

/**
  * Delivers pre-ime input events to the view hierarchy.
  * Does not support pointer events.
  */
final class ViewPreImeInputStage extends InputStage {
    ...
}

/**
  * Delivers pre-ime input events to a native activity.
  * Does not support pointer events.
  */
final class NativePreImeInputStage extends AsyncInputStage
    implements InputQueue.FinishedInputEventCallback {
    ...
}
```

从类注释中可以知道，ViewPostImeInputStage用来处理view层级的输入事件，也就说一般的触摸事件就是使用ViewPostImeInputStage来处理的，下面就来分析一下当ViewRooImpl是如和进行分发的：

首先事件会回调WindowInputEventReceiver的onInputEvent方法，onInputEvent中直接调用ViewRootImpl的enqueueInputEvent处理此事件：

```java
    void enqueueInputEvent(InputEvent event,
            InputEventReceiver receiver, int flags, boolean processImmediately) {
        adjustInputEventForCompatibility(event);
        QueuedInputEvent q = obtainQueuedInputEvent(event, receiver, flags);

        // Always enqueue the input event in order, regardless of its time stamp.
        // We do this because the application or the IME may inject key events
        // in response to touch events and we want to ensure that the injected keys
        // are processed in the order they were received and we cannot trust that
        // the time stamp of injected events are monotonic.
        QueuedInputEvent last = mPendingInputEventTail;
        if (last == null) {
            mPendingInputEventHead = q;
            mPendingInputEventTail = q;
        } else {
            last.mNext = q;
            mPendingInputEventTail = q;
        }
        mPendingInputEventCount += 1;
        Trace.traceCounter(Trace.TRACE_TAG_INPUT, mPendingInputEventQueueLengthCounterName,
                mPendingInputEventCount);

        if (processImmediately) {
            doProcessInputEvents();
        } else {
            scheduleProcessInputEvents();
        }
    }
```

enqueueInputEvent先将当前的输入事件添加到QueuedInputEvent队列中，然后通过doProcessInputEvents处理输入事件（scheduleProcessInputEvents最终也是通过doProcessInputEvents处理）。

```java
void doProcessInputEvents() {
    // Deliver all pending input events in the queue.
    while (mPendingInputEventHead != null) {
        QueuedInputEvent q = mPendingInputEventHead;
        mPendingInputEventHead = q.mNext;
        if (mPendingInputEventHead == null) {
            mPendingInputEventTail = null;
        }
        q.mNext = null;

        mPendingInputEventCount -= 1;
        ...

        deliverInputEvent(q);
    }
```

doProcessInputEvents从QueuedInputEvent队列中取出输入事件，调用deliverInputEvent进行处理：

```java
private void deliverInputEvent(QueuedInputEvent q) {
    ...

    InputStage stage;
    if (q.shouldSendToSynthesizer()) {
        stage = mSyntheticInputStage;
    } else {
        stage = q.shouldSkipIme() ? mFirstPostImeInputStage : mFirstInputStage;
    }

    if (stage != null) {
        stage.deliver(q);
    } else {
        finishInputEvent(q);
    }
}
```

这里是根据输入事件的状态，获取不同的状态对象进行处理，一般情况这里都是取的都是mFirstInputStage对象，接下去就是调用state的deliver方法处理事件了。

## TouchEvent事件的处理

前面讲到ViewPostImeInputStage是用于view树的事件处理类，因此一般情况下touchevent都是传递给它进行处理，下面我们看一下它的onProcess方法：

```java
@Override
protected int onProcess(QueuedInputEvent q) {
    if (q.mEvent instanceof KeyEvent) {
        return processKeyEvent(q);
    } else {
        // If delivering a new non-key event, make sure the window is
        // now allowed to start updating.
        handleDispatchWindowAnimationStopped();
        final int source = q.mEvent.getSource();
        if ((source & InputDevice.SOURCE_CLASS_POINTER) != 0) {
            return processPointerEvent(q);
        } else if ((source & InputDevice.SOURCE_CLASS_TRACKBALL) != 0) {
            return processTrackballEvent(q);
        } else {
            return processGenericMotionEvent(q);
        }
    }
}
```

这里根据事件类型来调用相应的处理方法，触摸事件使用processPointerEvent方法进行处理：

```java
private int processPointerEvent(QueuedInputEvent q) {
    final MotionEvent event = (MotionEvent)q.mEvent;

    mAttachInfo.mUnbufferedDispatchRequested = false;
    boolean handled = mView.dispatchPointerEvent(event);
    ...
    return handled ? FINISH_HANDLED : FORWARD;
}

// View.java
public final boolean dispatchPointerEvent(MotionEvent event) {
    if (event.isTouchEvent()) {
        return dispatchTouchEvent(event);
    } else {
        return dispatchGenericMotionEvent(event);
    }
}
```

这里调用View的dispatchPointerEvent方法，然后再调用dispatchTouchEvent进行分发。在Activity中，根据之前的DecorView加载流程可以知道，mView就是DecorView，因此底层事件就由ViewRootImpl分发到DecorView中了。

```java
@Override
public boolean dispatchTouchEvent(MotionEvent ev) {
    final Callback cb = getCallback();
    return cb != null && !isDestroyed() && mFeatureId < 0 ? cb.dispatchTouchEvent(ev)
        : super.dispatchTouchEvent(ev);
}
```

这里先调用PhoneWindow的getCallback方法获取callback，然后将事件通过dispatchTouchEvent方法分发到callback中。那么这个callback是什么呢？还记得DecorView加载流程中说到在创建Window之后会将Activity设为window的callback么，因此这里的callback就是Activity，也就是说到这里，底层的事件就分发到Activity中了。