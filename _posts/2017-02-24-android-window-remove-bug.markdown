---
layout:     post
title:      "记Android WindowManager.removeView的一个Bug"
subtitle:   "Encountered a bug in WindowManager.removeView"
date:       2017-02-24 00:00:00
author:     "Lizhe"
header-mask: 0.2
catalog:    true
tags:
    - Android
---

最近在使用WindowManager.removeView(View view)时遇到一个坑。

<img src="/img/in-post/post-android-window-remove-bug/window-removeview-bug.gif" />

我的做法是：把LockWindow动画到透明度为0，并在动画的onAnimationEnd里把LockWindow移除掉。

```java
public void removeLockWindow() {
    ObjectAnimator fadeOut = ObjectAnimator.ofFloat(LockWindow.this, View.ALPHA, 0);
    fadeOut.setDuration(DURATION_HIDE_WINDOW);
    fadeOut.addListener(new AnimatorListenerAdapter() {
        @Override public void onAnimationEnd(Animator animation) {
            mWindowManager.removeView(LockWindow.this);
        }
    });
    fadeOut.start();
}
```

但是发现会闪一下，Fuck！！！这也能有bug，只能好去读源码。

### 1. WindowManager和ViewManager

首先我们找到，removeView方法在interface ViewManager中，而interface WindowManager继承自ViewManager

```java
/** Interface to let you add and remove child views to an Activity. To get an instance
  * of this class, call {@link android.content.Context#getSystemService(java.lang.String) Context.getSystemService()}.
  */
public interface ViewManager
{
    public void addView(View view, ViewGroup.LayoutParams params);
    public void updateViewLayout(View view, ViewGroup.LayoutParams params);
    public void removeView(View view);
}

public interface WindowManager extends ViewManager {
	// 省略，声明了一些window的type，以及LayoutParams等
}
```

### 2. WindowManagerImpl

接着，需要找到WindowManager的实现类——WindowManagerImpl。看它removeView方法的具体实现,发现具体实现在WindowManagerGlobal的removeView中。
```java
public final class WindowManagerImpl implements WindowManager {
    // 定义mGlobal为WindowManagerGlobal的单例对象
    private final WindowManagerGlobal mGlobal = WindowManagerGlobal.getInstance();

    @Override
    public void removeView(View view) {
        mGlobal.removeView(view, false);
    }
}
```

### 3. WindowManagerGlobal

```java
public void removeView(View view, boolean immediate) {
    if (view == null) {
        throw new IllegalArgumentException("view must not be null");
    }

    synchronized (mLock) {
        int index = findViewLocked(view, true);
        View curView = mRoots.get(index).getView();
        removeViewLocked(index, immediate);
        if (curView == view) {
            return;
        }

        throw new IllegalStateException("Calling with view " + view
                + " but the ViewAncestor is attached to " + curView);
    }
}

private void removeViewLocked(int index, boolean immediate) {
    ViewRootImpl root = mRoots.get(index);
    View view = root.getView();

    if (view != null) {
        InputMethodManager imm = InputMethodManager.getInstance();
        if (imm != null) {
            imm.windowDismissed(mViews.get(index).getWindowToken());
        }
    }
    boolean deferred = root.die(immediate);
    if (view != null) {
        view.assignParent(null);
        if (deferred) {
            mDyingViews.add(view);
        }
    }
}
```

WindowManagerGlobal里的removeView方法主要调用了removeViewLocked。removeViewLocked里找到了要remove掉的这个View的ViewRootImpl对象，然后调用了ViewRootImpl的die()方法。

### 4. ViewRootImpl
```java
boolean die(boolean immediate) {
    // Make sure we do execute immediately if we are in the middle of a traversal or the damage
    // done by dispatchDetachedFromWindow will cause havoc on return.
    if (immediate && !mIsInTraversal) {
        doDie();
        return false;
    }

    if (!mIsDrawing) {
        destroyHardwareRenderer();
    } else {
        Log.e(TAG, "Attempting to destroy the window while drawing!\n" +
                "  window=" + this + ", title=" + mWindowAttributes.getTitle());
    }
    mHandler.sendEmptyMessage(MSG_DIE);
    return true;
}

private void destroyHardwareRenderer() {
    HardwareRenderer hardwareRenderer = mAttachInfo.mHardwareRenderer;

    if (hardwareRenderer != null) {
        if (mView != null) {
            hardwareRenderer.destroyHardwareResources(mView);
        }
        hardwareRenderer.destroy();
        hardwareRenderer.setRequested(false);

        mAttachInfo.mHardwareRenderer = null;
        mAttachInfo.mHardwareAccelerated = false;
    }
}
```
die()方法主要调用了destroyHardwareRenderer，destroyHardwareRenderer里面会通过HardwareRenderer递归删除掉所有View的HardwareLayer缓存，用来释放GPU显存，避免泄漏和因GPU显存不足报错。

好吧，线索到这看起来断掉了，这些东西和闪一下有什么关系呢？我们得到的信息有：WindowManager.removeView会同步地将HardwareLayer缓存清理掉，而这关系到View的绘制，所以应该从绘制流程上找原因。

### 5. Choreographer
Choreographer是Android4.1以后用来统一调度界面绘制的关键类。每一次界面的重绘都要调用Choreographer.doFrame()方法。

```java
/**
 * Callback type: Animation callback.  Runs before traversals.
 * @hide
 */
public static final int CALLBACK_ANIMATION = 1;

/**
 * Callback type: Traversal callback.  Handles layout and draw.  Runs
 * after all other asynchronous messages have been handled.
 * @hide
 */
public static final int CALLBACK_TRAVERSAL = 2;

void doFrame(long frameTimeNanos, int frame) {
    
    ***

    try {
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "Choreographer#doFrame");
        AnimationUtils.lockAnimationClock(frameTimeNanos / TimeUtils.NANOS_PER_MS);

        mFrameInfo.markInputHandlingStart();
        doCallbacks(Choreographer.CALLBACK_INPUT, frameTimeNanos);

        mFrameInfo.markAnimationsStart();
        doCallbacks(Choreographer.CALLBACK_ANIMATION, frameTimeNanos);

        mFrameInfo.markPerformTraversalsStart();
        doCallbacks(Choreographer.CALLBACK_TRAVERSAL, frameTimeNanos);

        doCallbacks(Choreographer.CALLBACK_COMMIT, frameTimeNanos);
    } finally {
        AnimationUtils.unlockAnimationClock();
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }

	***
}
```
通过读代码发现，在一个消息周期内，动画回调CALLBACK_ANIMATION发生在绘制回调CALLBACK_TRAVERSAL之前。结合篇首贴出的代码，在一个绘制周期内，动画回调里调用了WindowManager.removeView()方法，然后马上运行到doCallbacks(Choreographer.CALLBACK_TRAVERSAL, frameTimeNanos)，又做了一遍绘制！而View真正从界面上消失是在下一个消息周期，这之间的时间差值就是闪这一下的原因。

至于为何removeView之后的这次绘制无视了最外层View的alpha属性，并没有进一步探讨。

### 6. 解决

通过上一节的分析，只要在removeView之后不重绘界面就行了。即只需要把removeView的操作移到下一个消息周期。

```java
public void removeLockWindow() {
    ObjectAnimator fadeOut = ObjectAnimator.ofFloat(LockWindow.this, View.ALPHA, 0);
    fadeOut.setDuration(DURATION_HIDE_WINDOW);
    fadeOut.addListener(new AnimatorListenerAdapter() {
        @Override public void onAnimationEnd(Animator animation) {
            // 将removeView方法post到下一个消息周期
            post(new Runnable() {
                @Override public void run() {
                    mWindowManager.removeView(LockWindow.this);
                }
            });
        }
    });
    fadeOut.start();
}
```

其实这个问题也不算是removeView的bug，只是mWindowManager.removeView的异步特性导致的。以后碰到这个坑的小伙伴可以借鉴一下。

最后上个解决后的图，很柔和嗯

<img src="/img/in-post/post-android-window-remove-bug/window-removeview-normal.gif" />