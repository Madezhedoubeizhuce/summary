## 事件分发

## MotionEvent

### action

触摸事件的基本类型有三种：

- ACTION_DOWN: 表示手指按下屏幕
- ACTION_MOVE: 手指在屏幕上滑动时，会产生一系列的MOVE事件
- ACTION_UP: 手指抬起，离开屏幕

**正常情况下一个完整的触摸事件系列是：从ACTION_DOWN开始，到ACTION_UP结束** 。而如果出现了一些异常的情况，事件序列被中断，那么会产生一个取消事件：

- ACTION_CANCEL：当出现异常情况事件序列被中断，会产生该类型事件

多指操作时：

- ACTION_POINTER_DOWN: 当已经有一个手指按下的情况下，另一个手指按下会产生该事件
- ACTION_POINTER_UP: 多个手指同时按下的情况下，抬起其中一个手指会产生该事件

### 触控点

- 在MotionEvent对象内部，维护有一个数组。这个数组中的每一项对应不同的触摸点的信息
- MOVE事件和CANCEL事件是没有包含触控点索引的，只有DOWN类型和UP类型的事件才包含触控点索引
- 事件分离是把一个motionEvent中的触控点信息进行分离，只向子view发送其感兴趣的触控点信息

## ViewGroup对于触摸事件的处理

### TouchTarget

- TouchTarget本身是个链表，每个节点记录了子view所对应的触控点id。在viewGroup中，该链表的链表头是mFirstTouchTarget，如果他为null，表示没有任何子view接收了down事件。
- 使用一个整型变量来记录所有的触控id，整型变量中哪一个二进制位为1，则对应绑定该id的触控点。

### 事件拦截

- 安全拦截：当一个控件a被另一个非全屏控件b遮挡住的时候，行为由两个标志控制：

  - FILTER_TOUCHES_WHEN_OBSCURED：这个标志可以手动给控件设置，表示被非全屏控件覆盖时，直接过滤掉所有触摸事件。
  - FLAG_WINDOW_IS_OBSCURED：这个标志表示当前窗口被一个非全屏控件覆盖。

  ```java
  View.java api29
  public boolean onFilterTouchEventForSecurity(MotionEvent event) {
      // 两个标志，前者表示当被覆盖时不处理；后者表示当前窗口是否被非全屏窗口覆盖
      if ((mViewFlags & FILTER_TOUCHES_WHEN_OBSCURED) != 0
              && (event.getFlags() & MotionEvent.FLAG_WINDOW_IS_OBSCURED) != 0) {
          // Window is obscured, drop this touch.
          return false;
      }
      return true;
  }
  ```

- 逻辑拦截：只有需要分发到子view的事件才需要拦截 ，判断是否拦截主要依靠两个因素：FLAG_DISALLOW_INTERCEPT标志和 `onInterceptTouchEvent()` 方法。

  1. 子view可以通过requestDisallowInterceptTouchEvent方法强制要求viewGroup不要拦截事件，viewGroup中会设置一个FLAG_DISALLOW_INTERCEPT标志表示不拦截事件。但是当前事件序列结束后，这个标志会被清除。如果需要的话需要再次调用requestDisallowInterceptTouchEvent方法进行设置。
  2. 如果子view没有强制要求不拦截，那么会调用`onInterceptTouchEvent()` 方法判断是否需要拦截。onInterceptTouchEvent方法默认只对一种特殊情况作了拦截，一般情况下我们会重写这个方法来拦截事件。

  ```java
  ViewGroup.java api29
  public boolean dispatchTouchEvent(MotionEvent ev) {
      ...
          
      // 对遮盖状态进行过滤
      if (onFilterTouchEventForSecurity(ev)) {
          
          ...
  
          // 判断是否需要拦截
          final boolean intercepted;
          // down事件或者有target的非down事件则需要判断是否需要拦截
          // 否则不需要进行拦截判断，因为一定是交给自己处理
          if (actionMasked == MotionEvent.ACTION_DOWN
              || mFirstTouchTarget != null) {
              // 此标志为子view通过requestDisallowInterupt方法设置
              // 禁止viewGroup拦截事件
              final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
              if (!disallowIntercept) {
                  // 调用onInterceptTouchEvent判断是否需要拦截
                  intercepted = onInterceptTouchEvent(ev);
                  // 恢复事件状态
                  ev.setAction(action); 
              } else {
                  intercepted = false;
              }
          } else {
              // 自己消费了down事件，那么后续的事件非down事件都是自己处理
              intercepted = true;
          }
          ...;
      }
      ...;
  }
  ```

### 寻找down事件的消费者

```java
ViewGroup.java api29
public boolean dispatchTouchEvent(MotionEvent ev) {
    ...
         
    // 对遮盖状态进行过滤
    if (onFilterTouchEventForSecurity(ev)) {
        
        // action的高9-16位表示索引值
        // 低1-8位表示事件类型
        // 只有down或者up事件才有索引值
        final int action = ev.getAction();
        // 获取到真正的事件类型
        final int actionMasked = action & MotionEvent.ACTION_MASK;

        ...

        // 拦截内容的逻辑
        if (actionMasked == MotionEvent.ACTION_DOWN
            || mFirstTouchTarget != null) {
            ...
        } 

        ...

        // 三个变量：
        // split表示是否需要对事件进行分裂，对应多点触摸事件
        // newTouchTarget 如果是down或pointer_down事件的新的绑定target
        // alreadyDispatchedToNewTouchTarget 表示事件是否已经分发给targetview了
        final boolean split = (mGroupFlags & FLAG_SPLIT_MOTION_EVENTS) != 0;
        TouchTarget newTouchTarget = null;
        boolean alreadyDispatchedToNewTouchTarget = false;
        
        // 如果没有取消和拦截进入分发
        if (!canceled && !intercepted) {
			...
			// down或pointer_down事件，表示新的手指按下了，需要寻找接收事件的view
            if (actionMasked == MotionEvent.ACTION_DOWN
                || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
                || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
                
                // 多点触控会有不同的索引，获取索引号
                // 该索引位于MotionEvent中的一个数组，索引值就是数组下标值
                // 只有up或down事件才会携带索引值
                final int actionIndex = ev.getActionIndex(); 
                // 这个整型变量记录了TouchTarget中view所对应的触控点id
                // 触控点id的范围是0-31，整型变量中哪一个二进制位为1，则对应绑定该id的触控点
                // 例如 00000000 00000000 00000000 10001000
                // 则表示绑定了id为3和id为7的两个触控点
                // 这里根据是否需要分离，对触控点id进行记录，
                // 而如果不需要分离，则默认接收所有触控点的事件
                final int idBitsToAssign = split ? 1 << ev.getPointerId(actionIndex)
                    : TouchTarget.ALL_POINTER_IDS;

                // down事件表示该触控点事件序列是一个新的序列
                // 清除之前绑定到到该触控id的TouchTarget
                removePointersFromTouchTargets(idBitsToAssign);

                final int childrenCount = mChildrenCount;
                // 如果子控件数目不为0而且还没绑定到新的id
                if (newTouchTarget == null && childrenCount != 0) {
                    // 使用触控点索引获取触控点位置
                    final float x = ev.getX(actionIndex);
                    final float y = ev.getY(actionIndex);
                    // 从前到后创建view列表
                    final ArrayList<View> preorderedList = buildTouchDispatchChildList();
                    // 判断是否是自定义view顺序
                    final boolean customOrder = preorderedList == null
                        && isChildrenDrawingOrderEnabled();
                    final View[] children = mChildren;
                    
                    // 遍历所有子控件
                    for (int i = childrenCount - 1; i >= 0; i--) {
                        // 从子控件列表中获取到子控件
                        final int childIndex = getAndVerifyPreorderedIndex(
                            childrenCount, i, customOrder);
                        final View child = getAndVerifyPreorderedView(
                            preorderedList, children, childIndex);
                        
                        ...

                        // 检查该子view是否可以接受触摸事件和是否在点击的范围内
                        if (!child.canReceivePointerEvents()
                            || !isTransformedTouchPointInView(x, y, child, null)) {
                            ev.setTargetAccessibilityFocus(false);
                            continue;
                        }

                        // 检查该子view是否在touchTarget链表中
                        newTouchTarget = getTouchTarget(child);
                        if (newTouchTarget != null) {
                            // 链表中已经存在该子view，说明这是一个多点触摸事件
                            // 即两次都触摸到同一个view上
                            // 将新的触控点id绑定到该TouchTarget上
                            newTouchTarget.pointerIdBits |= idBitsToAssign;
                            break;
                        }

                        resetCancelNextUpFlag(child);
                        // 找到合适的子view，把事件分发给他，看该子view是否消费了down事件
                        // 如果消费了，需要生成新的TouchTarget
                        // 如果没有消费，说明子view不接受该down事件，继续循环寻找合适的子控件
                        if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                            // 保存该触控事件的相关信息
                            mLastTouchDownTime = ev.getDownTime();
                            if (preorderedList != null) {
                                // childIndex points into presorted list, find original index
                                for (int j = 0; j < childrenCount; j++) {
                                    if (children[childIndex] == mChildren[j]) {
                                        mLastTouchDownIndex = j;
                                        break;
                                    }
                                }
                            } else {
                                mLastTouchDownIndex = childIndex;
                            }
                            mLastTouchDownX = ev.getX();
                            mLastTouchDownY = ev.getY();
                            // 保存该view到target链表
                            newTouchTarget = addTouchTarget(child, idBitsToAssign);
                            // 标记已经分发给子view，退出循环
                            alreadyDispatchedToNewTouchTarget = true;
                            break;
                        }

                        ...
                    }// 这里对应for (int i = childrenCount - 1; i >= 0; i--)
                    ...
                }// 这里对应判断：(newTouchTarget == null && childrenCount != 0)

                if (newTouchTarget == null && mFirstTouchTarget != null) {
                    // 没有子view接收down事件，直接选择链表尾的view作为target
                    newTouchTarget = mFirstTouchTarget;
                    while (newTouchTarget.next != null) {
                        newTouchTarget = newTouchTarget.next;
                    }
                    newTouchTarget.pointerIdBits |= idBitsToAssign;
                }
                
            }// 这里对应if (actionMasked == MotionEvent.ACTION_DOWN...)
        }// 这里对应if (!canceled && !intercepted)
        ...
    }// 这里对应if (onFilterTouchEventForSecurity(ev))
    ...
}
```

### 分发事件

```java
ViewGroup.java api29
// 该方法接收原MotionEvent事件、是否进行取消、目标子view、以及目标子view感兴趣的触控id
// 如果不是取消事件这个方法会把原MotionEvent中的触控点信息拆分出目标view感兴趣的触控点信息
// 如果是取消事件则不需要拆分直接发送取消事件即可
private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,
        View child, int desiredPointerIdBits) {
    final boolean handled;

    // 如果是取消事件，那么不需要做其他额外的操作，直接派发事件即可，然后直接返回
    // 因为对于取消事件最重要的内容就是事件本身，无需对事件的内容进行设置
    final int oldAction = event.getAction();
    if (cancel || oldAction == MotionEvent.ACTION_CANCEL) {
        event.setAction(MotionEvent.ACTION_CANCEL);
        if (child == null) {
            handled = super.dispatchTouchEvent(event);
        } else {
            handled = child.dispatchTouchEvent(event);
        }
        event.setAction(oldAction);
        return handled;
    }

    // oldPointerIdBits表示现在所有的触控id
    // desirePointerIdBits来自于该view所在的touchTarget，表示该view感兴趣的触控点id
    // 因为desirePointerIdBits有可能全是1，所以需要和oldPointerIdBits进行位与
    // 得到真正可接收的触控点信息
    final int oldPointerIdBits = event.getPointerIdBits();
    final int newPointerIdBits = oldPointerIdBits & desiredPointerIdBits;

    // 控件处于不一致的状态。正在接受事件序列却没有一个触控点id符合
    if (newPointerIdBits == 0) {
        return false;
    }

    // 来自原始MotionEvent的新的MotionEvent，只包含目标感兴趣的触控点
    // 最终派发的是这个MotionEvent
    final MotionEvent transformedEvent;
    
    // 两者相等，表示该view接受所有的触控点的事件
    // 这个时候transformedEvent相当于原始MotionEvent的复制
    if (newPointerIdBits == oldPointerIdBits) {
        // 当目标控件不存在通过setScaleX()等方法进行的变换时，
        // 为了效率会将原始事件简单地进行控件位置与滚动量变换之后
        // 发送给目标的dispatchTouchEvent()方法并返回。
        if (child == null || child.hasIdentityMatrix()) {
            if (child == null) {
                handled = super.dispatchTouchEvent(event);
            } else {
                final float offsetX = mScrollX - child.mLeft;
                final float offsetY = mScrollY - child.mTop;
                event.offsetLocation(offsetX, offsetY);

                handled = child.dispatchTouchEvent(event);

                event.offsetLocation(-offsetX, -offsetY);
            }
            return handled;
        }
        // 复制原始MotionEvent
        transformedEvent = MotionEvent.obtain(event);
    } else {
        // 如果两者不等，说明需要对事件进行拆分
        // 只生成目标感兴趣的触控点的信息
        // 这里分离事件包括了修改事件的类型、触控点索引等
        transformedEvent = event.split(newPointerIdBits);
    }

    // 对MotionEvent的坐标系，转换为目标控件的坐标系并进行分发
    if (child == null) {
        handled = super.dispatchTouchEvent(transformedEvent);
    } else {
        // 计算滚动量偏移
        final float offsetX = mScrollX - child.mLeft;
        final float offsetY = mScrollY - child.mTop;
        transformedEvent.offsetLocation(offsetX, offsetY);
        // 存在scale等变换，需要进行矩阵转换
        if (! child.hasIdentityMatrix()) {
            transformedEvent.transform(child.getInverseMatrix());
        }
		// 调用子view的方法进行分发
        handled = child.dispatchTouchEvent(transformedEvent);
    }

    // 分发完毕，回收MotionEvent
    transformedEvent.recycle();
    return handled;
}


ViewGroup.java api29
public boolean dispatchTouchEvent(MotionEvent ev) {
    ...
        
    // 对遮盖状态进行过滤
    if (onFilterTouchEventForSecurity(ev)) {
		...

		
        if (mFirstTouchTarget == null) {
            // 经过了前面的处理，到这里touchTarget依旧为null，说明没有找到处理down事件的子控件
            // 或者down事件被viewGroup本身消费了，所以该事件由viewGroup自己处理
            // 这里调用了dispatchTransformedTouchEvent方法来分发事件
            handled = dispatchTransformedTouchEvent(ev, canceled, null,
                                                    TouchTarget.ALL_POINTER_IDS);
        } else {
            // 已经有子view消费了down事件
            TouchTarget predecessor = null;
            TouchTarget target = mFirstTouchTarget;
            // 遍历所有的TouchTarget并把事件分发下去
            while (target != null) {
                final TouchTarget next = target.next;
                if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
                    // 表示事件在前面已经处理了，不需要重复处理
                    handled = true;
                } else {
                    // 正常分发事件或者分发取消事件
                    final boolean cancelChild = resetCancelNextUpFlag(target.child)
                        || intercepted;
                    // 这里调用了dispatchTransformedTouchEvent方法来分发事件
                    if (dispatchTransformedTouchEvent(ev, cancelChild,
                                                      target.child, target.pointerIdBits)) {
                        handled = true;
                    }
                    // 如果发送了取消事件，则移除该target
                    if (cancelChild) {
                        if (predecessor == null) {
                            mFirstTouchTarget = next;
                        } else {
                            predecessor.next = next;
                        }
                        target.recycle();
                        target = next;
                        continue;
                    }
                }
                predecessor = target;
                target = next;
            }
        }

        // 如果接收到取消获取up事件，说明事件序列结束
        // 直接删除所有的TouchTarget
        if (canceled
            || actionMasked == MotionEvent.ACTION_UP
            || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
            // 清除记录的信息
            resetTouchState();
        } else if (split && actionMasked == MotionEvent.ACTION_POINTER_UP) {
            final int actionIndex = ev.getActionIndex();
            final int idBitsToRemove = 1 << ev.getPointerId(actionIndex);
            // 如果仅仅只是一个PONITER_UP
            // 清除对应触控点的触摸信息
            removePointersFromTouchTargets(idBitsToRemove);
        }
        
    }// 这里对应if (onFilterTouchEventForSecurity(ev))

    if (!handled && mInputEventConsistencyVerifier != null) {
        mInputEventConsistencyVerifier.onUnhandledEvent(ev, 1);
    }
    return handled;
}
```



## View处理触摸事件

到这里为止，我们已经了解到ViewGroup是如何将事件分发到View的，接下来就要看下View中如何处理触摸事件。

```java
public boolean dispatchTouchEvent(MotionEvent event) {
	...

    boolean result = false;

    ...

    if (onFilterTouchEventForSecurity(event)) {
        //noinspection SimplifiableIfStatement
        ListenerInfo li = mListenerInfo;
        if (li != null && li.mOnTouchListener != null
            && (mViewFlags & ENABLED_MASK) == ENABLED
            && li.mOnTouchListener.onTouch(this, event)) {
            result = true;
        }

        if (!result && onTouchEvent(event)) {
            result = true;
        }
    }

    ...

    return result;
}
```

如果OnTouchListener不是空的，则直接调用OnTouchListener的onTouch方法处理此事件，当OnTouchListener的onTouch方法返回true之后，便不再调用onTouchEvent处理该事件了。

如果OnTouchListener为空或者其的onTouch方法返回false，则调用onTouchEvent方法处理当前事件：

```java
public boolean onTouchEvent(MotionEvent event) {
    final float x = event.getX();
    final float y = event.getY();
    final int viewFlags = mViewFlags;
    final int action = event.getAction();

    ...

    if (((viewFlags & CLICKABLE) == CLICKABLE ||
         (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE) ||
        (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE) {
        switch (action) {
            case MotionEvent.ACTION_UP:
                boolean prepressed = (mPrivateFlags & PFLAG_PREPRESSED) != 0;
                if ((mPrivateFlags & PFLAG_PRESSED) != 0 || prepressed) {
                    // take focus if we don't have it already and we should in
                    // touch mode.
                    boolean focusTaken = false;
                    ...

                    if (!mHasPerformedLongPress && !mIgnoreNextUpEvent) {
                        // This is a tap, so remove the longpress check
                        removeLongPressCallback();

                        // Only perform take click actions if we were in the pressed state
                        if (!focusTaken) {
                            // Use a Runnable and post this rather than calling
                            // performClick directly. This lets other visual state
                            // of the view update before click actions start.
                            if (mPerformClick == null) {
                                mPerformClick = new PerformClick();
                            }
                            if (!post(mPerformClick)) {
                                performClick();
                            }
                        }
                    }

                   	...
                }
                mIgnoreNextUpEvent = false;
                break;

            case MotionEvent.ACTION_DOWN:
                mHasPerformedLongPress = false;

                if (performButtonActionOnTouchDown(event)) {
                    break;
                }

                // Walk up the hierarchy to determine if we're inside a scrolling container.
                boolean isInScrollingContainer = isInScrollingContainer();

                // For views inside a scrolling container, delay the pressed feedback for
                // a short period in case this is a scroll.
                if (isInScrollingContainer) {
                    ...
                } else {
                    // Not inside a scrolling container, so show the feedback right away
                    setPressed(true, x, y);
                    checkForLongClick(0);
                }
                break;

            case MotionEvent.ACTION_CANCEL:
                setPressed(false);
                removeTapCallback();
                removeLongPressCallback();
                mInContextButtonPress = false;
                mHasPerformedLongPress = false;
                mIgnoreNextUpEvent = false;
                break;

            case MotionEvent.ACTION_MOVE:
                drawableHotspotChanged(x, y);

                // Be lenient about moving outside of buttons
                if (!pointInView(x, y, mTouchSlop)) {
                    // Outside button
                    removeTapCallback();
                    if ((mPrivateFlags & PFLAG_PRESSED) != 0) {
                        // Remove any future long press/tap checks
                        removeLongPressCallback();

                        setPressed(false);
                    }
                }
                break;
        }

        return true;
    }

    return false;
}
```

这里是对ACTION_UP、ACTION_DOWN、ACTION_CANCEL和ACTION_MOVE事件的处理，主要就是在ACTION_UP事件时调用performClick触发点击事件，在ACTION_DOWN事件时调用checkForLongClick方法根据具体方法触发长按事件，这里就不详细介绍了。

view辨别单击和长按的方法是**设置延时任务**，在源码中会看到很多的类似的代码，这里延时任务使用handler来实现。当一个down事件来临时，会添加一个延时任务到消息队列中。如果时间到还没有接收到up事件，说明这是个长按事件，那么就会调用onLongClickListener监听器，而如果在延时时间内收到了up事件，那么说明这是个单击事件，取消这个延时的任务，并调用onClickListener。判断是否是一个长按事件，调用的是 `checkForLongClick` 方法来设置延时任务：

```java
private void checkForLongClick(int delayOffset) {
    if ((mViewFlags & LONG_CLICKABLE) == LONG_CLICKABLE) {
        // 标志还没触发长按
        // 如果延迟时间到，触发长按监听，这个变量 就会被设置为true
        // 那么当up事件到来时，就不会触摸单击监听，也就是onClickListener
        mHasPerformedLongPress = false;

        if (mPendingCheckForLongPress == null) {
            mPendingCheckForLongPress = new CheckForLongPress();
        }
        mPendingCheckForLongPress.rememberWindowAttachCount();
        postDelayed(mPendingCheckForLongPress,
                    ViewConfiguration.getLongPressTimeout() - delayOffset);
    }
}

private final class CheckForLongPress implements Runnable {
    private int mOriginalWindowAttachCount;

    @Override
    public void run() {
        if (isPressed() && (mParent != null)
            && mOriginalWindowAttachCount == mWindowAttachCount) {
            // 在延时时间到之后，就会运行这个任务
            // 调用onLongClickListener监听器
            // 并设置mHasPerformedLongPress为true
            if (performLongClick()) {
                mHasPerformedLongPress = true;
            }
        }
    }

    public void rememberWindowAttachCount() {
        mOriginalWindowAttachCount = mWindowAttachCount;
    }
}
```