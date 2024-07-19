---
title: View的Touch事件分发
tags: 自定义View
permalink: android-source/dc-view-5
key: android-source-dc-view-5
sidebar:
  nav: android-source
---

## 前言

Android中的View事件分发机制是指当用户触摸屏幕时，系统如何将触摸事件分发给各个View，并由它们来处理事件的过程。事件分发机
制主要包括三个部分：事件的产生、事件的分发和事件的处理。

事件的产生：当用户触摸屏幕时，系统会产生一个MotionEvent对象，该对象包含了触摸点的坐标、触摸的时间、触摸的压力等信息

事件的分发：事件分发是由ViewGroup来完成的，它会将事件分发给它的子View，并根据子View的返回值来决定是否继续分发事件。如果子View处理了事件，那么事件就不会再传递给其他View

事件的处理：事件的处理是由View来完成的，它会根据事件的类型来调用相应的回调方法，如onTouchEvent()、onClickListener()等。

总的来说，Android的View事件分发机制是一个复杂的过程，需要开发者深入理解和掌握，才能编写出高效、稳定的应用程序。

<!--more-->

## 基础认识

### 事件分发的概念

将触摸事件（MotionEvent）传递到某个具体的`View` & 处理的整个过程

### 事件在哪些对象之间进行传递

**Activity、ViewGroup、View**。`Android`的`UI`界面由`Activity`、`ViewGroup`、`View` 及其派生类组成

<img src="https://raw.githubusercontent.com/QingDian-Fan/ImageRepository/master/images/image-20240719104925361.png" alt="image-20240719104925361" style="zoom:50%;" />

| **类型**  | 简介                  | 备注                                                         |
| --------- | --------------------- | ------------------------------------------------------------ |
| Activity  | 控制生命周期&处理事件 | - 统筹UI视图的显示<br />- 通过回调方法与Window、View进行交互 |
| ViewGroup | 一组View的集合        | - 本身也是View的子类<br />- Android中所有布局的父类          |
| View      | 所有UI组件的基类      | 所有控件都是继承自View                                       |

### 事件分发的顺序

事件传递的顺序：`Activity` -> `ViewGroup` -> `View`

即：1个点击事件发生后，事件先传到`Activity`、再传到`ViewGroup`、最终再传到 `View`

### 事件分发过程有哪些主要方法

dispatchTouchEvent()、onInterceptTouchEvent()和onTouchEvent()

| 方法                    | 作用                             | 调用时刻                                |
| ----------------------- | -------------------------------- | --------------------------------------- |
| dispatchTouchEvent()    | 分发事件<br />（DOWN、MOVE、UP） | 当事件传递给当前View时会被调用          |
| onInterceptTouchEvent() | 事件拦截<br />只存在于ViewGroup  | 在ViewGroup的dispatchTouchEvent()中调用 |
| onTouchEvent()          | 处理事件                         | 在dispatchTouchEvent()中调用            |

## 源码分析

`Android`事件分发流程 = **Activity -> ViewGroup -> View**

即要想充分理解Android分发机制，本质上是要理解：

1. `Activity`对事件的分发机制
2. `ViewGroup`对事件的分发机制
3. `View`事件的分发机制

### Activity对事件的分发机制

#### 源码分析

Android事件分发机制首先会将事件传递到Activity中，具体是执行dispatchTouchEvent()进行事件分发。

```
    //Activity.dispatchTouchEvent()-事件分发开始
    public boolean dispatchTouchEvent(MotionEvent ev) {
        ...
        //分析入口1
        //若getWindow().superDispatchTouchEvent(ev)的返回true，则事件停止往下传递，事件结束
        //否则：继续往下调用Activity.onTouchEvent
        if (getWindow().superDispatchTouchEvent(ev)) {
            return true;
        }
        //分析入口2
        return onTouchEvent(ev);
    }
    
    
    // PhoneWindow.superDispatchTouchEvent()
    //	1.getWindow() = 获取Window类的对象
    //	2.Window类是抽象类，其唯一实现类 = PhoneWindow类
    //	3.Window类的superDispatchTouchEvent() = 1个抽象方法，由子类PhoneWindow类实现
    @Override
    public boolean superDispatchTouchEvent(MotionEvent event) {
    		//这里mDecor即顶层View(DecorView)
        return mDecor.superDispatchTouchEvent(event);
    }
    
    
    //DecorView.superDispatchTouchEvent(event)
    //DecorView就是一个ViewGroup，继承自FrameLayout
    public boolean superDispatchTouchEvent(MotionEvent event) {
    		//调用父类的方法 = ViewGroup的dispatchTouchEvent()
    		//即 将事件传递给ViewGroup
        return super.dispatchTouchEvent(event);
    }
    
    // 回到最初的分析2入口处
    // Activity.onTouchEvent()
    // 当一个事件未被Activity下任何一个View接收/处理时，就会调用该方法
    public boolean onTouchEvent(MotionEvent event) {
        if (mWindow.shouldCloseOnTouch(this, event)) {
            finish();
            return true;
        }
        // 即 只有在点击事件在Window边界外才会返回true，一般情况都返回false，分析完毕
        return false;
    }
    
    
    //Window.shouldCloseOnTouch()
    //主要是对于处理边界外事件的判断：是否是DOWN事件，event的坐标是否在边界内等
    public boolean shouldCloseOnTouch(Context context, MotionEvent event) {
        final boolean isOutside =
                event.getAction() == MotionEvent.ACTION_UP && isOutOfBounds(context, event)
                || event.getAction() == MotionEvent.ACTION_OUTSIDE;
        if (mCloseOnTouchOutside && peekDecorView() != null && isOutside) {
        // 返回true：说明事件在边界外，即 消费事件
            return true;
        }
        // 返回false：在边界内，即未消费（默认）
        return false;
    }
```

#### 总结

![image-20240719114036708](https://raw.githubusercontent.com/QingDian-Fan/ImageRepository/master/images/image-20240719114036708.png)

### ViewGroup 对事件的分发机制

从上面Activity的事件分发机制可知，在Activity.dispatchTouchEvent()实现了将事件从Activity->ViewGroup的传递，ViewGroup的事件分发机制从dispatchTouchEvent()开始

#### 源码分析

```
 public boolean dispatchTouchEvent(MotionEvent ev) {
      
				...
				
        boolean handled = false;
        if (onFilterTouchEventForSecurity(ev)) {
            ...
            //如果是按下事件的话
            if (actionMasked == MotionEvent.ACTION_DOWN) {
            		//清除target，将mFirstTouchTarget置为空
                cancelAndClearTouchTargets(ev);
                resetTouchState();
            }

            
            final boolean intercepted;
            //如果是按下事件或者mFirstTouchTarget不为空的话，进入
            if (actionMasked == MotionEvent.ACTION_DOWN
                    || mFirstTouchTarget != null) {
                final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
                //如果子控件没有请求禁止拦截的话，则进入onInterceptTouchEvent方法，否则intercepted =true
                if (!disallowIntercept) {
                    intercepted = onInterceptTouchEvent(ev);
                    ev.setAction(action); 
                } else {
                    intercepted = false;
                }
            } else {
                intercepted = true;
            }
            
				  	...
						//如果没有被取消或者没有被拦截的情况下，进入
            if (!canceled && !intercepted) {
								...
                    if (newTouchTarget == null && childrenCount != 0) {
                    		// 获取触摸事件点的 x，y坐标
                        final float x =
                                isMouseEvent ? ev.getXCursorPosition() : ev.getX(actionIndex);
                        final float y =
                                isMouseEvent ? ev.getYCursorPosition() : ev.getY(actionIndex);
      
                     	...
                        //循环便利所有子View
                        for (int i = childrenCount - 1; i >= 0; i--) {
                         	...
														//判断子View是否包含事件点
                            if (!child.canReceivePointerEvents()
                                    || !isTransformedTouchPointInView(x, y, child, null)) {
                                ev.setTargetAccessibilityFocus(false);
                                continue;
                            }
													
                         	...

                            resetCancelNextUpFlag(child);
                            //在dispatchTransformedTouchEvent中将事件传递给子View
                            if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                        				...
                                //给mFirstTouchTarget进行赋值
                                newTouchTarget = addTouchTarget(child, idBitsToAssign);
                                break;
                            }

                            ev.setTargetAccessibilityFocus(false);
                        }
                        if (preorderedList != null) preorderedList.clear();
                    }

                }
            }
            //如果没有子View处理该事件，或遍历没有找到子View
 					 if (mFirstTouchTarget == null) {
               
                handled = dispatchTransformedTouchEvent(ev, canceled, null,
                        TouchTarget.ALL_POINTER_IDS);
            } 



        return handled;
    }
    
    
 		public boolean onInterceptTouchEvent(MotionEvent ev) {
        if (ev.isFromSource(InputDevice.SOURCE_MOUSE)
                && ev.getAction() == MotionEvent.ACTION_DOWN
                && ev.isButtonPressed(MotionEvent.BUTTON_PRIMARY)
                && isOnScrollbarThumb(ev.getX(), ev.getY())) {
            return true;
        }
        return false;
    }
    
    
     private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,
            View child, int desiredPointerIdBits) {
        final boolean handled;
				...
				
        if (child == null) {
        		//调用父类View的dispatchTouchEvent
            handled = super.dispatchTouchEvent(transformedEvent);
        } else {
            final float offsetX = mScrollX - child.mLeft;
            final float offsetY = mScrollY - child.mTop;
            transformedEvent.offsetLocation(offsetX, offsetY);
            if (! child.hasIdentityMatrix()) {
                transformedEvent.transform(child.getInverseMatrix());
            }
            //调用子View 的dispatchTouchEvent()
            handled = child.dispatchTouchEvent(transformedEvent);
        }

 
        transformedEvent.recycle();
        return handled;
    }
    
    
    

```

#### 总结



### View事件的分发机制

从上面`ViewGroup`事件分发机制知道，`View`事件分发机制从`dispatchTouchEvent()`开始

#### 源码分析

```
    public boolean dispatchTouchEvent(MotionEvent event) {
       ...
        boolean result = false;

        if (mInputEventConsistencyVerifier != null) {
            mInputEventConsistencyVerifier.onTouchEvent(event, 0);
        }

        final int actionMasked = event.getActionMasked();
        if (actionMasked == MotionEvent.ACTION_DOWN) {
            // Defensive cleanup for new gesture
            stopNestedScroll();
        }

        if (onFilterTouchEventForSecurity(event)) {
            if ((mViewFlags & ENABLED_MASK) == ENABLED && handleScrollBarDragging(event)) {
                result = true;
            }
            // ListenerInfo是包含View事件的所有监听方法的一个封装类
            //  li.mOnTouchListener.onTouch(this, event)调用的是OnTouchListener
            ListenerInfo li = mListenerInfo;
            if (li != null && li.mOnTouchListener != null
                    && (mViewFlags & ENABLED_MASK) == ENABLED
                    && li.mOnTouchListener.onTouch(this, event)) {
                result = true;
            }
						//调用自己的onTouchEvent时间
            if (!result && onTouchEvent(event)) {
                result = true;
            }
        }

        ...

        return result;
    }
    
    
    
    public boolean onTouchEvent(MotionEvent event) {
       	...

        if (clickable || (viewFlags & TOOLTIP) == TOOLTIP) {
            switch (action) {
                case MotionEvent.ACTION_UP:
                    mPrivateFlags3 &= ~PFLAG3_FINGER_DOWN;
                    if ((viewFlags & TOOLTIP) == TOOLTIP) {
                        handleTooltipUp();
                    }
                    if (!clickable) {
                        removeTapCallback();
                        removeLongPressCallback();
                        mInContextButtonPress = false;
                        mHasPerformedLongPress = false;
                        mIgnoreNextUpEvent = false;
                        break;
                    }
                    boolean prepressed = (mPrivateFlags & PFLAG_PREPRESSED) != 0;
                    if ((mPrivateFlags & PFLAG_PRESSED) != 0 || prepressed) {
                        // take focus if we don't have it already and we should in
                        // touch mode.
                        boolean focusTaken = false;
                        if (isFocusable() && isFocusableInTouchMode() && !isFocused()) {
                            focusTaken = requestFocus();
                        }

                        if (prepressed) {
                            // The button is being released before we actually
                            // showed it as pressed.  Make it show the pressed
                            // state now (before scheduling the click) to ensure
                            // the user sees it.
                            setPressed(true, x, y);
                        }

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
                                    performClickInternal();
                                }
                            }
                        }

                        if (mUnsetPressedState == null) {
                            mUnsetPressedState = new UnsetPressedState();
                        }

                        if (prepressed) {
                            postDelayed(mUnsetPressedState,
                                    ViewConfiguration.getPressedStateDuration());
                        } else if (!post(mUnsetPressedState)) {
                            // If the post failed, unpress right now
                            mUnsetPressedState.run();
                        }

                        removeTapCallback();
                    }
                    mIgnoreNextUpEvent = false;
                    break;

                case MotionEvent.ACTION_DOWN:
                    if (event.getSource() == InputDevice.SOURCE_TOUCHSCREEN) {
                        mPrivateFlags3 |= PFLAG3_FINGER_DOWN;
                    }
                    mHasPerformedLongPress = false;

                    if (!clickable) {
                        checkForLongClick(
                                ViewConfiguration.getLongPressTimeout(),
                                x,
                                y,
                                TOUCH_GESTURE_CLASSIFIED__CLASSIFICATION__LONG_PRESS);
                        break;
                    }

                    if (performButtonActionOnTouchDown(event)) {
                        break;
                    }

                    // Walk up the hierarchy to determine if we're inside a scrolling container.
                    boolean isInScrollingContainer = isInScrollingContainer();

                    // For views inside a scrolling container, delay the pressed feedback for
                    // a short period in case this is a scroll.
                    if (isInScrollingContainer) {
                        mPrivateFlags |= PFLAG_PREPRESSED;
                        if (mPendingCheckForTap == null) {
                            mPendingCheckForTap = new CheckForTap();
                        }
                        mPendingCheckForTap.x = event.getX();
                        mPendingCheckForTap.y = event.getY();
                        postDelayed(mPendingCheckForTap, ViewConfiguration.getTapTimeout());
                    } else {
                        // Not inside a scrolling container, so show the feedback right away
                        setPressed(true, x, y);
                        checkForLongClick(
                                ViewConfiguration.getLongPressTimeout(),
                                x,
                                y,
                                TOUCH_GESTURE_CLASSIFIED__CLASSIFICATION__LONG_PRESS);
                    }
                    break;

                case MotionEvent.ACTION_CANCEL:
                    if (clickable) {
                        setPressed(false);
                    }
                    removeTapCallback();
                    removeLongPressCallback();
                    mInContextButtonPress = false;
                    mHasPerformedLongPress = false;
                    mIgnoreNextUpEvent = false;
                    mPrivateFlags3 &= ~PFLAG3_FINGER_DOWN;
                    break;

                case MotionEvent.ACTION_MOVE:
                    if (clickable) {
                        drawableHotspotChanged(x, y);
                    }

                    final int motionClassification = event.getClassification();
                    final boolean ambiguousGesture =
                            motionClassification == MotionEvent.CLASSIFICATION_AMBIGUOUS_GESTURE;
                    int touchSlop = mTouchSlop;
                    if (ambiguousGesture && hasPendingLongPressCallback()) {
                        if (!pointInView(x, y, touchSlop)) {
                            // The default action here is to cancel long press. But instead, we
                            // just extend the timeout here, in case the classification
                            // stays ambiguous.
                            removeLongPressCallback();
                            long delay = (long) (ViewConfiguration.getLongPressTimeout()
                                    * mAmbiguousGestureMultiplier);
                            // Subtract the time already spent
                            delay -= event.getEventTime() - event.getDownTime();
                            checkForLongClick(
                                    delay,
                                    x,
                                    y,
                                    TOUCH_GESTURE_CLASSIFIED__CLASSIFICATION__LONG_PRESS);
                        }
                        touchSlop *= mAmbiguousGestureMultiplier;
                    }

                    // Be lenient about moving outside of buttons
                    if (!pointInView(x, y, touchSlop)) {
                        // Outside button
                        // Remove any future long press/tap checks
                        removeTapCallback();
                        removeLongPressCallback();
                        if ((mPrivateFlags & PFLAG_PRESSED) != 0) {
                            setPressed(false);
                        }
                        mPrivateFlags3 &= ~PFLAG3_FINGER_DOWN;
                    }

                    final boolean deepPress =
                            motionClassification == MotionEvent.CLASSIFICATION_DEEP_PRESS;
                    if (deepPress && hasPendingLongPressCallback()) {
                        // process the long click action immediately
                        removeLongPressCallback();
                        checkForLongClick(
                                0 /* send immediately */,
                                x,
                                y,
                                TOUCH_GESTURE_CLASSIFIED__CLASSIFICATION__DEEP_PRESS);
                    }

                    break;
            }

            return true;
        }

        return false;
    }
```

#### 总结

## 总结



