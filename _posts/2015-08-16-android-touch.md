---
layout: post
title: "Android touch事件处理流程"
description: "Android对touch事件的处理流程，以及何时产生onClick事件等"
category: Android 
tags: [Android, touch]
---
# 序言
工作中遇到一个业务场景：对WebView做一个封装，能根据webview某个点的透明度决定事件是被消费掉还是透传下去。有两个思路，一个是继承，一个是引用，能继承当然是最好的，但是有的时候也许根本没办法继承webview，比如为了资源回收等问题系统使用同一个webview组件，但不巧的是，为了模块解耦，在别的模块根本拿不到webview的类，只能通过bridge拿到一个引用，这个就必须通过引用来实现了。无论哪种方法都涉及到了touch事件的处理流程，简单梳理下。

# View对touch事件的处理流程
对Touch事件的处理涉及四个重要的方法：

dispatchTouchEvent(MotionEvent event) 
setOnTouchListener(OnTouchListener l)
onTouchEvent(MotionEvent event)


先了解一下`dispatchTouchEvent(MotionEvent event) `:

    /**
     * Pass the touch screen motion event down to the target view, or this
     * view if it is the target.
     *
     * @param event The motion event to be dispatched.
     * @return True if the event was handled by the view, false otherwise.
     */
    public boolean dispatchTouchEvent(MotionEvent event) {
        // If the event should be handled by accessibility focus first.
        if (event.isTargetAccessibilityFocus()) {
            // We don't have focus or no virtual descendant has it, do not handle the event.
            if (!isAccessibilityFocusedViewOrHost()) {
                return false;
            }
            // We have focus and got the event, then use normal event dispatch.
            event.setTargetAccessibilityFocus(false);
        }

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

        if (!result && mInputEventConsistencyVerifier != null) {
            mInputEventConsistencyVerifier.onUnhandledEvent(event, 0);
        }

        // Clean up after nested scrolls if this is the end of a gesture;
        // also cancel it if we tried an ACTION_DOWN but we didn't want the rest
        // of the gesture.
        if (actionMasked == MotionEvent.ACTION_UP ||
                actionMasked == MotionEvent.ACTION_CANCEL ||
                (actionMasked == MotionEvent.ACTION_DOWN && !result)) {
            stopNestedScroll();
        }

        return result;
    }
   
   代码略长，但是和重点部分是下面的这几句：

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
这段代码先判断了通过`setOnTouchEvent(OnTouchListener l)` 设置的`onTouchListener`, 如果listener不为空，View 的点击状态是ENABLE，并且`onTouchListener.onTouch` 返回true，那么给Result置为true。代码接着判断Result为false，才会执行View 自身的`onTouchEvent` 方法，如果`onTouchEvent` 返回true则将Result置为true。如果经过这两步Result为true，代表当前事件已经被消费掉了，触发下一次事件。否则是不会触发下一次事件的，典型的表现就是action_down被触发而action_move, action_up不会得到响应。

也就是说：touch事件会首先触发dispatchTouchEvent方法，并且如果外围设置了onTouchListener会先执行onTouchListener的onTouch方法，只有当onTouch返回false才会执行自身的onTouchEvent方法。

看一下 `onTouchEvent(MotionEvent event)`:

    /**
     * Implement this method to handle touch screen motion events.
     * <p>
     * If this method is used to detect click actions, it is recommended that
     * the actions be performed by implementing and calling
     * {@link #performClick()}. This will ensure consistent system behavior,
     * including:
     * <ul>
     * <li>obeying click sound preferences
     * <li>dispatching OnClickListener calls
     * <li>handling {@link AccessibilityNodeInfo#ACTION_CLICK ACTION_CLICK} when
     * accessibility features are enabled
     * </ul>
     *
     * @param event The motion event.
     * @return True if the event was handled, false otherwise.
     */
    public boolean onTouchEvent(MotionEvent event) {
        final float x = event.getX();
        final float y = event.getY();
        final int viewFlags = mViewFlags;

        if ((viewFlags & ENABLED_MASK) == DISABLED) {
            if (event.getAction() == MotionEvent.ACTION_UP && (mPrivateFlags & PFLAG_PRESSED) != 0) {
                setPressed(false);
            }
            // A disabled view that is clickable still consumes the touch
            // events, it just doesn't respond to them.
            return (((viewFlags & CLICKABLE) == CLICKABLE ||
                    (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE));
        }

        if (mTouchDelegate != null) {
            if (mTouchDelegate.onTouchEvent(event)) {
                return true;
            }
        }

        if (((viewFlags & CLICKABLE) == CLICKABLE ||
                (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)) {
            switch (event.getAction()) {
                case MotionEvent.ACTION_UP:
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

                        if (!mHasPerformedLongPress) {
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
                        checkForLongClick(0);
                    }
                    break;

                case MotionEvent.ACTION_CANCEL:
                    setPressed(false);
                    removeTapCallback();
                    removeLongPressCallback();
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
代码的逻辑还是非常清晰的，在该方法中主要是做了Event的action产生click，longPress等事件。

# ViewGroup对touch事件的处理
ViewGroup和Touch事件相关的方法比起View多了一个onInterceptTouchEvent(MotionEvent ev)，而这个方法代码非常简单：

    public boolean onInterceptTouchEvent(MotionEvent ev) {
        return false;
    }
但是它的注释却长的吓人：

![](https://github.com/helloyingying/helloyingying.github.io/blob/master/assets/image/onInterceptTouchEvent.png)

用一段测试程序来说明这些流程，假设有一个ViewGroup比如FrameLayout，里面包含了一个Button。

Activity的代码：

    findViewById(R.id.touch_btn).setOnTouchListener(new       View.OnTouchListener() {
            @Override
            public boolean onTouch(View v, MotionEvent event) {
                Log.d("MyButton", "OnTouchListener");
                return false;
            }
        });

        findViewById(R.id.touch_frame_layout).setOnTouchListener(new View.OnTouchListener() {
            @Override
            public boolean onTouch(View v, MotionEvent event) {
                Log.d("MyFrameLayout", "OnTouchListener");
                return false;
            }
        });


MyButton的代码：

    public class MyButton extends Button {
	    public MyButton(Context context) {
	        super(context);
	    }
	    
	    public MyButton(Context context, AttributeSet attrs) {
	        super(context, attrs);
	    }
	    
	    public MyButton(Context context, AttributeSet attrs, int defStyleAttr) {
	        super(context, attrs, defStyleAttr);
	    }
	    
	    @Override
	    public boolean onTouchEvent(MotionEvent event) {
	        Log.d("MyButton", "onTouchEvent");
	        return super.onTouchEvent(event);
	    }
	    
	    @Override
	    public boolean dispatchTouchEvent(MotionEvent event) {
	        Log.d("MyButton", "dispatchTouchEvent");
	        return super.dispatchTouchEvent(event);
	    }
    }

MyFrameLayout的代码如下:

    public class MyTouchFrameLayout extends FrameLayout {
	    public MyTouchFrameLayout(Context context) {
	        super(context);
	    }
	
	    public MyTouchFrameLayout(Context context, AttributeSet attrs) {
	        super(context, attrs);
	    }
	
	    public MyTouchFrameLayout(Context context, AttributeSet attrs, int defStyleAttr) {
	        super(context, attrs, defStyleAttr);
	    }
	
	    @Override
	    public boolean onInterceptTouchEvent(MotionEvent ev) {
	        Log.d("MyFrameLayout", "onInterceptTouchEvent");
	        return super.onInterceptTouchEvent(ev);
	    }
	
	    @Override
	    public boolean dispatchTouchEvent(MotionEvent ev) {
	        Log.d("MyFrameLayout", "dispatchTouchEvent");
	        return super.dispatchTouchEvent(ev);
	    }
	
	    @Override
	    public boolean onTouchEvent(MotionEvent event) {
	        Log.d("MyFrameLayout", "onTouchEvent");
	        return super.onTouchEvent(event);
	    }
    }

点击button时事件的传递流程如下：

**action_down**
1. `MyFrameLayout﹕ dispatchTouchEvent`
2. `MyFrameLayout﹕ onInterceptTouchEvent`
3. `MyButton﹕ dispatchTouchEvent`
4. `MyButton﹕ OnTouchListener`
5. `MyButton﹕ onTouchEvent`
**action_up**
`MyFrameLayout﹕ dispatchTouchEvent`
`MyFrameLayout﹕ onInterceptTouchEvent`
`MyButton﹕ dispatchTouchEvent`
`MyButton﹕ OnTouchListener`
`MyButton﹕ onTouchEvent`

也就是说任何一个action都会先触发父类布局的dispatchTouchEvent -> onInterceptTouchEvent, 但是FrameLayout的onTouchEvent方法并没有被调用。

如果将onInterceptTouchEvent的返回值改为true，测试结果如下所示：

1. `MyFrameLayout﹕ dispatchTouchEvent`
2. `MyFrameLayout﹕ onInterceptTouchEvent`
3. `MyFrameLayout﹕ OnTouchListener`
4. `MyFrameLayout﹕ onTouchEvent`

可以发现执行到了FrameLayout的onTouchEvent方法，但是MyButton的所有方法都没有被执行到。

去ViewGroup的dispatchTouchEvent方法一探究竟（代码过长只说部分代码和结论，有兴趣直接看源码）：

![](https://github.com/techsummary/techsummary.github.io/tree/master/assets/image/dispatch_intercept.png)

结论：

* 在`dispatchTouchEvent`方法中会首先执行`onInterceptTouchEvent`判断是否拦截事件，如果`onInterceptTouchEvent`返回false表示不拦截则将**倒序**遍历子View（布局中后添加的View会先做出响应），并调用`dispatchTransformedTouchEvent`方法递归调用子View的`dispatchTouchEvent`方法，如果子View为Viewgroup并且没有被拦截那么递归调用`dispatchTouchEvent`，如果子View为View调用`onTouchEvent`方法。

* 如果`onInterceptTouchEvent`返回true，表示将事件拦截下来不往子View传递，调用自身的`onTouchEvent`事件。


# 基于继承关系的PenetrateWebView（可穿透WebView）

明白了touch事件在View中的传递流程，使用继承来实现PenetrateWebView也就非常简单了。

首先有一个工具方法来获取某一点的alpha值：

    private static int getAlphaOfViewPoint(View view, int x, int y) {
        view.setDrawingCacheEnabled(true);
        Bitmap bmp = view.getDrawingCache(true);
        int pixel = bmp.getPixel(x, y);
        int alpha = Color.alpha(pixel);
        TMDebugLog.logv("pop_layer.point_alpha ", alpha+"");
        view.destroyDrawingCache();
        return alpha;
    }

然后再自定义WebView中重写onTouchEvent方法：

     public boolean onTouchEvent(MotionEvent event) {
        try {
            if (event.getY() < 0 || event.getX() < 0)
                return super.onTouchEvent(event);
               
            //如果某点的alpha值小于等于阈值，将事件透传下去
            //反之将事件消费掉
            if (getAlphaOfViewPoint(this, (int) event.getX(), (int) event.getY()) <= mPenetrateAlpha) {
                return false;
            }
            return super.onTouchEvent(event);
        } catch (Throwable e) {
            e.printStackTrace();
            return super.onTouchEvent(event);
        }
    }
    
基于继承关系的PenetrateWebView实现起来还是非常容易的，下面讨论一下基于引用关系的PenetrateWebView。

# 基于引用关系的PenetrateWebView

当没有办法继承的时候我首先想到的是设置OnTouchListener，但是onTouch返回true虽然将事件消费掉了，但是就不会执行onTouchEvent方法，也就无法产生click事件，整个WebView失去了交互的能力。返回false更加没有任何意义了，还是没办法控制onTouchEvent。
于是想到了在WebView的外部嵌套一层布局实现事件的拦截，一开始没搞清楚ViewGroup中事件的传递流程，在dispatchTouchEvent和onTouchEvent中写逻辑，发现还是没办法实现很好的控制，在阅读了touch相关的代码后，使用下面的方式实现了PenetrateWebView。

    @Override
    public boolean onInterceptTouchEvent(MotionEvent event) {
        try {
            if (event.getY() < 0 || event.getX() < 0)
                return super.onTouchEvent(event);
            // 如果该点的alpha小于等于阈值，在该布局中拦截下事件，交给上层布局处理，onTouchEvent事件会被执行，onTouchEvent始终返回false，将事件透传下去
            // 如果该点的alpha大于阈值，将事件交给子View处理，onTouchEvent事件并不会被执行，将事件拦截下来
            return getAlphaOfViewPoint(this, (int) event.getX(), (int) event.getY()) <= mPenetrateAlpha;
        } catch (Throwable e) {
            e.printStackTrace();
            return false;
        }
    }

    @Override
    public boolean onTouchEvent(MotionEvent event) {
        return false;//当该方法被执行的时候，将事件交给父类布局去处理
    }

# 总结

1. View设置OnTouchListener之后，如果onTouch返回true那么onTouchEvent方法不会被执行。
2. click事件是在onTouchEvent方法中产生的。
3. ViewGroup中Touch事件传递是从外到内的，子View倒序遍历。
4. ViewGroup中onInterceptTouchEvent方法返回false的时候不会执行该ViewGroup的onTouchEvent方法，返回true则会执行。
