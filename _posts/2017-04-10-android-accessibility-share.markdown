---
layout:     post
title:      "Android无障碍设计（Accessibility）分享"
subtitle:   "Android Accessibility Service Introduction"
date:       2017-04-10 12:00:00
author:     "Lizhe"
header-mask: 0.2
catalog:    true
tags:
    - Android
    - Accessibility
---

> Android is for Everyone,  and Everyone has the access to Everything.

Google在API 4加入Accessibility Service的初衷就是为了服务残疾人，或者无法正常和设备交互的人，比如正在开车的司机、带小孩的母亲、视力弱化的老人等。

[https://realm.io/news/kelly-shuster-android-is-for-everyone/](https://realm.io/news/kelly-shuster-android-is-for-everyone/)

* 全世界安卓设备: 14亿
* 全世界永久伤残人数: 10亿
* 美国永久伤残人口比例: 20%

Google对Accessibility Service的描述为：

> Accessibility services should only be used to assist users with disabilities in using Android devices and apps.

>They run in the background and receive callbacks by the system when AccessibilityEvents are fired. Such events denote some state transition in the user interface, for example, the focus has changed, a button has been clicked, etc.

>Such a service can optionally request the capability for querying the content of the active window. Development of an accessibility service requires extending this class and implementing its abstract methods.


* Accessibility Service应该**只被**用来辅助残障用户使用Android设备和应用。

* 它运行在后台，能够在AccessibilityEvent发出后接到系统的**异步回调**。

* 这些事件代表着用户界面的状态变化，例如，焦点变更、按钮点击等。

* Accessibility Service也可以通过额外的配置，获取遍历界面上所有内容的能力。

## 一、Accessibility基础知识

### 1.1 声明

```java
package com.lizhe.tech;

import android.accessibilityservice.AccessibilityService;
import android.view.accessibility.AccessibilityEvent;

public class MyAccessibilityService extends AccessibilityService {

    @Override public void onAccessibilityEvent(AccessibilityEvent event) {

    }

    @Override protected boolean onKeyEvent(KeyEvent event) {
        return super.onKeyEvent(event);
    }

    @Override protected boolean onGesture(int gestureId) {
        return super.onGesture(gestureId);
    }

    @Override public void onInterrupt() {

    }
}
```

```xml
<service android:name=".MyAccessibilityService"
         android:permission="android.permission.BIND_ACCESSIBILITY_SERVICE">
     <intent-filter>
         <action android:name="android.accessibilityservice.AccessibilityService" />
     </intent-filter>
     . . .
 </service>
```

* 声明android.permission.BIND_ACCESSIBILITY_SERVICE权限
* 声明Service可以处理android.accessibilityservice.AccessibilityService这个Intent

### 1.2 配置

#### 1.2.1 两种配置方式

A、在Manifest里声明meta-data，指定一个xml文件。

[ AndroidManifest.xml ]
```xml
<service android:name=".MyAccessibilityService">
	<intent-filter>
		<action android:name="android.accessibilityservice.AccessibilityService" />
	</intent-filter>
	<meta-data android:name="android.accessibilityservice" android:resource="@xml/accessibilityservice" />
 </service>
```

[ xml/accessibilityservice.xml ]
```xml
 <accessibility-service
     android:accessibilityEventTypes="typeViewClicked|typeWindowStateChanged"
     android:packageNames="foo.bar, foo.baz"
     android:notificationTimeout="100"
     android:accessibilityFlags="flagDefault"
     android:canRetrieveWindowContent="true"
     android:description="@string/boost_plus_accessibility_service_description"
     . . .
 />
```

B、在代码里配置
```java
void setServiceInfo (AccessibilityServiceInfo info)
```

#### 1.2.2 配置项说明
* accessibilityEventTypes 代表希望接收的事件类型

* packageNames 接收哪些应用的AccessibilityEvent

* notificationTime 相邻两个AccessibilityEvent接收的最小时间间隔

* canRetrieveWindowContent 是否能获取窗口内容

* canRequestFilterKeyEvents 是否接收按键消息

* description 权限描述文字

<img src="../../../../img/in-post/post-android-accessibility-share/acc-declare-description.png" style="width:40%"/>

<img src="../../../../img/in-post/post-android-accessibility-share/acc-declare-alert.png" style="width:40%"/>
### 1.3 使用

#### 1.3.1 事件监听(异步回调)

```java
void onAccessibilityEvent (AccessibilityEvent event)
```

- `TYPE_VIEW_CLICKED`  View点击事件

- `TYPE_VIEW_LONG_CLICKED`  View长按事件

- `TYPE_VIEW_TEXT_CHANGED` EditText内容变化事件

- `TYPE_VIEW_SCROLLED` 界面滑动结束

- `TYPE_WINDOW_STATE_CHANGED` 代表任何界面的跳转，如Activity、Dialog、PopupWindow等

- `TYPE_NOTIFICATION_STATE_CHANGED` 通知栏里的通知改变

```java
boolean onKeyEvent (KeyEvent event)
```

```java
boolean onGesture (int gestureId)
```

#### 1.3.2 动作模拟
```java
// 全局性动作
boolean performGlobalAction (int action)
```

+ `GLOBAL_ACTION_BACK` Back键

+ `GLOBAL_ACTION_HOME` Home键

+ `GLOBAL_ACTION_RECENTS` Recent键

```java
// 对某个View做动作
AccessibilityNodeInfo: boolean performAction (int action)
```

+ `ACTION_CLICK` 点击动作

+ `ACTION_LONG_CLICK` 长按动作

+ `ACTION_SCROLL_UP` 向上滑动

+ `ACTION_SCROLL_DOWN` 向下滑动

+ `ACTION_SET_TEXT` 设置文字内容

#### 1.3.3 AccessibilityNodeInfo
View Hierarchy和AccessibilityNodeInfo树状结构。

```java
AccessibilityNodeInfo AccessibilityService.getRootInActiveWindow()
```

```java
public class AccessibilityNodeInfo implements Parcelable {
    // find children
    public int getChildCount ()
    public AccessibilityNodeInfo getChild (int index)

    // find children by filters
    public List<AccessibilityNodeInfo> findAccessibilityNodeInfosByText (String text)
    public List<AccessibilityNodeInfo> findAccessibilityNodeInfosByViewId (String viewId)     

    // get view display information
    public int getWindowId ()
    public void getBoundsInScreen (Rect outBounds)
    public CharSequence getText ()
    public boolean isChecked ()

    // get view class information
    public CharSequence getClassName ()
    public CharSequence getPackageName ()

    // do something
    public boolean performAction (int action)
}
```

## 二、实际应用

### 2.1 获取TopActivity

获取TopActivity（前台应用）的几种方式
* Lollipop之前, 使用ActivityManage的getRunningTasks()方法

* Lollipop及以后，使用AppUsageAccess方法

* AccessibiltyService方式获取

#### 2.1.1 getRunningTasks()方法
需要权限
```xml
<uses-permission android:name="android.permission.GET_TASKS" />
```
实现：
```java
ActivityManager activityManager = (ActivityManager)context.getApplicationContext().getSystemService(Context.ACTIVITY_SERVICE);
ComponentName runningTopActivity = activityManager.getRunningTasks(1).get(0).topActivity;
```

* 不止能获取包名，还能获取Activity名。

* Android 5.0以后，不再对第三方应用提供这种方式来获取前台应用。虽然调用这个方法还是能够返回结果，但是结果只包含你自己的Activity和Launcher了。

#### 2.1.2 App Usage Access方法

需要权限，并需要引导用户开启App Usage Access权限。
```xml
<uses-permission android:name="android.permission.PACKAGE_USAGE_STATS" />
```

实现：
```java
UsageStatsManager mUsageStatsManager = (UsageStatsManager)context.getApplicationContext().getSystemService(Context.USAGE_STATS_SERVICE);
long time = System.currentTimeMillis();
List<UsageStats> stats ;
if (isFirst){
    stats = mUsageStatsManager.queryUsageStats(UsageStatsManager.INTERVAL_DAILY, time - TWENTYSECOND, time);
}else {
    stats = mUsageStatsManager.queryUsageStats(UsageStatsManager.INTERVAL_DAILY, time - THIRTYSECOND, time);
}
// Sort the stats by the last time used
if(stats != null) {
    TreeMap<Long,UsageStats> mySortedMap = new TreeMap<Long,UsageStats>();
    start=System.currentTimeMillis();
    for (UsageStats usageStats : stats) {
        mySortedMap.put(usageStats.getLastTimeUsed(),usageStats);
    }
    LogUtil.e(TAG,"isFirst="+isFirst+",mySortedMap cost:"+ (System.currentTimeMillis()-start));
    if(mySortedMap != null && !mySortedMap.isEmpty()) {                   

        NavigableSet<Long> keySet=mySortedMap.navigableKeySet();
        Iterator iterator=keySet.descendingIterator();
        while(iterator.hasNext()){
            UsageStats usageStats = mySortedMap.get(iterator.next());
            if (mLastEventField==null) {
                try {
                    mLastEventField = UsageStats.class.getField("mLastEvent");
                } catch (NoSuchFieldException e) {
                    break;
                }
            }
            if (mLastEventField!=null) {
                int lastEvent = 0;
                try {
                    lastEvent = mLastEventField.getInt(usageStats);
                } catch (IllegalAccessException e) {
                    break;
                }
                if (lastEvent==1){
                    topPackageName=usageStats.getPackageName();
                    break;
                }
            }else {
                break;
            }
        }    
        if (topPackageName==null){
            topPackageName =  mySortedMap.get(mySortedMap.lastKey()).getPackageName();
        }
        runningTopActivity=new ComponentName(topPackageName,"");
        if (LogUtil.isDebug())LogUtil.d(TAG,topPackageName);
    }
}
```
代码的功能是通过UsageStatsManager 来获取用户使用的程序的列表，然后按照最近使用时间排序，就得到了当前的前台应用，这种方式只能拿到包名，无法精确到Activity。

#### 2.1.3 AccessibilityService方法

```java
public class AccessibilityMonitorService extends AccessibilityService {
    private CharSequence mWindowClassName;
    private String mCurrentPackage;
    @Override
    public void onAccessibilityEvent(AccessibilityEvent event) {
        int type=event.getEventType();
        switch (type){
            case AccessibilityEvent.TYPE_WINDOW_STATE_CHANGED:
                mWindowClassName = event.getClassName();
                mCurrentPackage = event.getPackageName()==null?"":event.getPackageName().toString();                
                break;
            case TYPE_VIEW_CLICKED:
            case TYPE_VIEW_LONG_CLICKED:
                break;
        }
    }
}
```
就这么简单，就可以获取当前前台应用的包名和Activity名了。

涉及到的AccessibilityService相关知识点：
* AccessibilityEvent.TYPE_WINDOW_STATE_CHANGED

* AccessibilityEvent.getClassName()

* AccessibilityEvent.getPackageName()

### 2.2 监听系统按钮

```java
public class KeyEventService extends AccessibilityService {

    private AccessibilityServiceInfo mAccessibilityServiceInfo;

    String back ;
    String home ;
    String recent ;

    @Override
    public void onCreate() {
        super.onCreate();
        back = getVitualNavigationKey(this, "accessibility_back", "com.android.systemui", "");
        home = getVitualNavigationKey(this, "accessibility_home", "com.android.systemui", "");
        recent = getVitualNavigationKey(this, "accessibility_recent", "com.android.systemui", "");

        mAccessibilityServiceInfo=new AccessibilityServiceInfo();
        mAccessibilityServiceInfo.feedbackType=FEEDBACK_GENERIC;
        mAccessibilityServiceInfo.eventTypes=AccessibilityEvent.TYPE_VIEW_CLICKED|AccessibilityEvent.TYPE_VIEW_LONG_CLICKED|AccessibilityEvent.TYPE_WINDOW_STATE_CHANGED;
        int flag=0;
        if (Build.VERSION.SDK_INT>=Build.VERSION_CODES.LOLLIPOP){
            flag=flag|AccessibilityServiceInfo.FLAG_RETRIEVE_INTERACTIVE_WINDOWS;
        }
        if (Build.VERSION.SDK_INT>=Build.VERSION_CODES.JELLY_BEAN_MR2){
            flag=flag|AccessibilityServiceInfo.FLAG_REQUEST_FILTER_KEY_EVENTS;
        }
        mAccessibilityServiceInfo.flags=flag;
        mAccessibilityServiceInfo.notificationTimeout=100;
        setServiceInfo(mAccessibilityServiceInfo);
    }

    @Override
    protected boolean onKeyEvent(KeyEvent paramKeyEvent) {
        // do something

        return false;
    }

    @Override
    public void onAccessibilityEvent(AccessibilityEvent event) {
        //虚拟手机按键处理，优先级高于是否点击分词的判断
        if ("com.android.systemui".equals(event.getPackageName())
        {
            if (TextUtils.isEmpty(event.getContentDescription())){
                return;
            }

            if (!TextUtils.isEmpty(back) && event.getContentDescription().equals(back)){
                // back key logic
            }else if (!TextUtils.isEmpty(home) && event.getContentDescription().equals(home)){
                // home key logic
            }else if (!TextUtils.isEmpty(recent) && event.getContentDescription().equals(recent)){
                // recent key logic
            }
        }
    }
}
```

* 需要配置FLAG_REQUEST_FILTER_KEY_EVENTS。
* 收到的也是异步回调，对这些paramKeyEvent的修改不会影响到应用中接收事件。
* onAccessibilityEvent中做的处理，是为了兼容具有NavigationBar（虚拟导航栏）的手机。因为在这些手机上，点击返回、Home、多任务三个按钮的时候，是作为普通的点击事件来处理的，也就是onAccessibilityEvent中的回调。

### 2.3 获取其他App中的文字

```java
public class GetTextService extends AccessibilityService {

    @Override
    public void onAccessibilityEvent(AccessibilityEvent event) {
        int type = event.getEventType();
        switch (type) {
            case TYPE_VIEW_CLICKED:
            case TYPE_VIEW_LONG_CLICKED:
                getText(event);
                break;
        }
    }

    private synchronized String getText(AccessibilityEvent event){
        AccessibilityNodeInfo info=event.getSource();
        if(info==null){
            return null;
        }
        CharSequence textSequence =info.getText();

        if (TextUtils.isEmpty(textSequence)){
            List<CharSequence> textArray =event.getText();
            if (textArray !=null) {
                StringBuilder sb=new StringBuilder();
                for (CharSequence t : textArray) {
                    sb.append(t);
                }
                textSequence =sb.toString();
            }
        }
        return textSequence.toString();
    }
}
```

相关知识点：
* AccessibilityEvent.getSource()
* AccessibilityNodeInfo.getText()

### 2.4 清理内存（ForceStop)

模拟点击Force Stop按钮

- 扫描正在运行的应用列表，得到[A,B,C,D]

- 通过Intent和IntentExtra，打开A的App Info界面

- 监听`TYPE_WINDOW_STATE_CHANGED`，找到Force Stop节点

- 对Force Stop节点，发出`ACTION_CLICK`，弹出确认对话框

- 监听`TYPE_WINDOW_STATE_CHANGED`，找到弹出对话框的OK节点

- 对OK节点，发出`ACTION_CLICK`

- 发出`GLOBAL_ACTION_BACK`，关闭App Info界面
- ……
- 对B、C、D应用做相同处理
- ……

<video width="40%" style="margin:0 auto;display: block;" src="../../../../img/in-post/post-android-accessibility-share/accessibility-force-stop.mp4" controls="controls" autoplay="autoplay" loop="loop"></video>

### 2.5 自动权限获取

工具类应用一般需要引导用户获取各种权限，比如Notification Listening、5.0以上的App Usage Access、设备管理器权限、悬浮窗权限等等。而用户经常是没有耐心去开启这么多权限的，所以我们可以在拿到Accessibility权限以后，静默开启这些权限。

<video width="40%" style="margin:0 auto;display: block;" src="../../../../img/in-post/post-android-accessibility-share/accessibility-permission-acquire.mp4" controls="controls" autoplay="autoplay" loop="loop"></video>

50%透明红色的覆盖可以替换成不透明的动画，这样就可以在不干扰用户视觉的前提下，帮助用户开启需要的权限。

### 2.6 开发者工具

* GPU Overdraw
* Force RTL
* Layout Bounds
* GPU Rendering
* Stay Awake
* Show CPU Usage
* Kill Activity
* Wait for Debugger

<video width="40%" style="margin:0 auto;display: block;" src="../../../../img/in-post/post-android-accessibility-share/accessibility-dev-tools.mp4" controls="controls" autoplay="autoplay" loop="loop"></video>

### 2.7 获取通知改变

```java
public class NotificationService extends AccessibilityService {

    @Override
    public void onAccessibilityEvent(AccessibilityEvent event) {
        //判断辅助服务触发的事件是否是通知栏改变事件
        if (event.getEventType() == AccessibilityEvent.TYPE_NOTIFICATION_STATE_CHANGED) {
            Parcelable data = event.getParcelableData();
            if (data instanceof Notification) {
                Notification notification = (Notification) data;
                // do something with this notification

            }
        }
    }

    @Override
    protected void onServiceConnected() {
        //设置关心的事件类型
        AccessibilityServiceInfo info = new AccessibilityServiceInfo();
        info.eventTypes = AccessibilityEvent.TYPE_NOTIFICATION_STATE_CHANGED |
                AccessibilityEvent.TYPE_WINDOW_STATE_CHANGED |
                AccessibilityEvent.TYPE_WINDOW_CONTENT_CHANGED;
        info.feedbackType = AccessibilityServiceInfo.FEEDBACK_GENERIC;
        info.notificationTimeout = 100;//两个相同事件的超时时间间隔
        setServiceInfo(info);
    }
}
```

### 2.8 微信检查被删好友

[传送门](http://www.jianshu.com/p/5cac6d439eeb)

### 2.9 抢红包插件
### 2.10 拦截和获取短信验证码
### 2.11 窃取各种隐私

## 三、开发符合Accessibility规范的应用

当App用户数或流行度增长到某个级别之后，需要考虑App的Accessibility性能，来保证App在特殊情况下对用户友好。

大厂们都对这个方向比较重视，因为能标榜自己产品的品味、人性化等。

* Facebook做了Accessibility Debug工具

* Google新发布的AccessibilityScanner

浏览App的导航方式：
* Touch Screen，最普遍

* External Hareware（DPad，trackball, keyboard, etc.），其他肢体残缺人士的辅助设施
* TalkBack (an accessibility service published by Google)，盲人

### 3.1 Focus Navigation
Focus Navigation是Android平台内置的一种导航方式。

忽视它会导致：
* 文本的阅读顺序混乱

* 某些按钮不能被激活交互

### 3.2 问题举例1

<img style="width:50%" src="../../../../img/in-post/post-android-accessibility-share/accessibility-fab.png"/>

```java
fab.setAccessibilityTraversalBefore (R.id.list_email);
```
* The screen reader will read out the tool bar text if there is any.

* Then it will immediately jump to the floating action button and read out that.
* Then it will start running through the content on the screen.

### 3.3 问题举例2

<img style="width:50%" src="../../../../img/in-post/post-android-accessibility-share/accessibility-content.png"/>

```xml
    <android.support.v7.widget.AppCompatImageView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:contentDescription="@null"
        app:srcCompat="@drawable/ic_charging_screen_logo"/>

    <android.support.v7.widget.AppCompatImageView
        android:id="@+id/ic_menu"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:contentDescription="@string/menu"
        app:srcCompat="@drawable/ic_more_vert_black_24dp"/>
```

> “Unlabeled button xxx.”

### 3.4 如何判断一个View是否是Focusable的
![流程图](../../../../img/in-post/post-android-accessibility-share/accessibility-focus.png)

### 3.5 测试Accessibility性能

[通过Stetho调试应用的Accessibility性能](https://code.facebook.com/posts/391276077927020/android-accessibility-debugging-with-stetho/)

[通过Accessibility Scanner测试应用的Accessibility性能](https://support.google.com/accessibility/android/answer/6376570)

[Accessibility Scanner传送门](https://play.google.com/store/apps/details?id=com.google.android.apps.accessibility.auditor&hl=en)

## 四、一些问题

### 4.1 为何Accessibility权限总是莫名消失

#### 4.1.1 ActivityManagerService
触发force stop操作后，都会调用到ActivityManagerService里面。

```java
private void forceStopPackageLocked(final String packageName, int uid, String reason) {
    forceStopPackageLocked(packageName, UserHandle.getAppId(uid), false,
                false, true, false, false, UserHandle.getUserId(uid), reason);

    Intent intent = new Intent(Intent.ACTION_PACKAGE_RESTARTED,
                Uri.fromParts("package", packageName, null));
    //系统启动完毕后,则mProcessesReady=true
    if (!mProcessesReady) {
            intent.addFlags(Intent.FLAG_RECEIVER_REGISTERED_ONLY
                        | Intent.FLAG_RECEIVER_FOREGROUND);
    }
    intent.putExtra(Intent.EXTRA_UID, uid);
    intent.putExtra(Intent.EXTRA_USER_HANDLE, UserHandle.getUserId(uid));
    //发送广播ACTION_PACKAGE_RESTARTED用于停止alarm和清除通知
    broadcastIntentLocked(null, null, intent,
                    null, null, 0, null, null, null, AppOpsManager.OP_NONE,
                    null, false, false, MY_PID, Process.SYSTEM_UID, UserHandle.getUserId(uid));
}
```

#### 4.1.2 PackageMonitor
PackageMonitor是一个HelperClass，用来便捷地监听应用的安装、卸载、更新等。其中就监听了Intent.ACTION_PACKAGE_RESTARTED。
```java
/**
 * Helper class for monitoring the state of packages: adding, removing,
 * updating, and disappearing and reappearing on the SD card.
 */
public abstract class PackageMonitor extends android.content.BroadcastReceiver {
    static final IntentFilter sPackageFilt = new IntentFilter();
    static final IntentFilter sNonDataFilt = new IntentFilter();
    static final IntentFilter sExternalFilt = new IntentFilter();

    static {
        sPackageFilt.addAction(Intent.ACTION_PACKAGE_ADDED);
        sPackageFilt.addAction(Intent.ACTION_PACKAGE_REMOVED);
        sPackageFilt.addAction(Intent.ACTION_PACKAGE_CHANGED);
        sPackageFilt.addAction(Intent.ACTION_QUERY_PACKAGE_RESTART);
        sPackageFilt.addAction(Intent.ACTION_PACKAGE_RESTARTED);
        sPackageFilt.addDataScheme("package");
        sNonDataFilt.addAction(Intent.ACTION_UID_REMOVED);
        sNonDataFilt.addAction(Intent.ACTION_USER_STOPPED);
        sExternalFilt.addAction(Intent.ACTION_EXTERNAL_APPLICATIONS_AVAILABLE);
        sExternalFilt.addAction(Intent.ACTION_EXTERNAL_APPLICATIONS_UNAVAILABLE);
    }

    public void register(Context context, Looper thread, boolean externalStorage) {
        register(context, thread, null, externalStorage);
    }

    public void register(Context context, Looper thread, UserHandle user,
            boolean externalStorage) {
        ......
        if (user != null) {
            context.registerReceiverAsUser(this, user, sPackageFilt, null, mRegisteredHandler);
            context.registerReceiverAsUser(this, user, sNonDataFilt, null, mRegisteredHandler);
            if (externalStorage) {
                context.registerReceiverAsUser(this, user, sExternalFilt, null,
                        mRegisteredHandler);
            }
        } else {
            context.registerReceiver(this, sPackageFilt, null, mRegisteredHandler);
            context.registerReceiver(this, sNonDataFilt, null, mRegisteredHandler);
            if (externalStorage) {
                context.registerReceiver(this, sExternalFilt, null, mRegisteredHandler);
            }
        }
    }

    public boolean onHandleForceStop(Intent intent, String[] packages, int uid, boolean doit) {
        return false;
    }

    @Override
    public void onReceive(Context context, Intent intent) {
        String action = intent.getAction();
        ......
        if (Intent.ACTION_PACKAGE_RESTARTED.equals(action)) {
            mDisappearingPackages = new String[] {getPackageName(intent)};
            mChangeType = PACKAGE_TEMPORARY_CHANGE;
            onHandleForceStop(intent, mDisappearingPackages,
                    intent.getIntExtra(Intent.EXTRA_UID, 0), true);
        }
        ......
    }
}
```

#### 4.1.3 AccessibilityManagerService

AccessibilityManagerService是系统Service中的一个，管理所有应用的Accessibility权限。

它的构造函数里会调用registerPackageChangeAndBootCompletedBroadcastReceiver，注册package changed和boot complete监听。

```java
/**
 * Creates a new instance.
 *
 * @param context A {@link Context} instance.
 */
AccessibilityManagerService(Context context) {
    mContext = context;
    mPackageManager = mContext.getPackageManager();
    mCaller = new HandlerCaller(context, this);
    // 构造函数里注册广播
    registerPackageChangeAndBootCompletedBroadcastReceiver();
    registerSettingsContentObservers();
}
```

```java
/**
 * Registers a {@link BroadcastReceiver} for the events of
 * adding/changing/removing/restarting a package and boot completion.
 */
private void registerPackageChangeAndBootCompletedBroadcastReceiver() {
    Context context = mContext;
    PackageMonitor monitor = new PackageMonitor() {
        ......
        // 重点看onHandleForceStop的逻辑
        @Override
        public boolean onHandleForceStop(Intent intent, String[] packages,
                int uid, boolean doit) {
            synchronized (mLock) {
                boolean changed = false;
                // 遍历mEnabledServices
                Iterator<ComponentName> it = mEnabledServices.iterator();
                while (it.hasNext()) {
                    ComponentName comp = it.next();
                    String compPkg = comp.getPackageName();
                    for (String pkg : packages) {
                        if (compPkg.equals(pkg)) {
                            if (!doit) {
                                return true;
                            }
                            // 移除回调中的package对应的对象
                            it.remove();
                            changed = true;
                        }
                    }
                }
                ......
            }
        }
        ......
    };
    // package changes
    monitor.register(context, true);
    // boot completed
    IntentFilter bootFiler = new IntentFilter(Intent.ACTION_BOOT_COMPLETED);
    mContext.registerReceiver(monitor, bootFiler);
}
```
在onHandleForceStop的回调用，把forceStop对应的app从mEnabledServices中移除，即将Accessibility权限关闭。

#### 4.1.4 我们能做什么？

看到上面的分析，我们可以推测，之所以Accessibility权限莫名其妙消失，是因为app被forceStop了。

要么是第三方清理软件使用模拟点击在系统设置中点了ForceStop按钮；<br>
要么是系统级应用使用am force-stop pkgName命令结束。

那应该怎么应对呢？结果很悲观，技术上什么都做不了。

从用户角度分析，可以引导用户手动将我们的app加入系统或第三方app的白名单。下面是某app对用户的教诲：

We unfortunately has no control over this - Android disabled it on some devices and we cannot prevent this. However, on most devices, you can disable app optimization in Settings to stop it from disabling the accessibility. This is usually found in Settings > Battery.

For example on the Samsung S5, go to Android Settings > General > Battery > look under App Optimization and select Details. Then find our app and turn it off.

On other devices, you may have to specify that our app should be ignored in battery optimization. For example, on Huawei phones, go to Settings > Apps > Advanced Options > Ignore Optimations and toggle "allow".

If app is running on an SD card (external storage), that means it's "unavailable" when the system boots. This can turn off accessibility. Try moving it back to main storage.

On some specific device models, even if the Accessibility service is running, it does not necessarily mean our app receives Accessibility events so that our service can work properly. Our development team is looking for possible solutions to address this issue in the future.

If this is occurring for you, we recommend toggling Accessibility Service off and on. Then retry.

---

综上，该问题的症结在于：Android Framework针对被forceStop的应用回收了Accessibility权限，且没有技术解决方案能跳过这个机制。只能对用户进行引导和培养，减小权限被回收的概率。

### 4.2 Accessibility Service单例

Accessibility权限的赋予是以Service为单元，而不是以Application为单元。

譬如：一个应用里需要用Accessibility Service做两件事
* 清理内存

* 监听系统按键

如果创建两个Accessibility Service，效果如下图

![一个应用中如果有多个Accessibility Service时](http://upload-images.jianshu.io/upload_images/4075712-ba20e8275f22acd3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

为了解决这个问题，需要定义一个公共的CommonAccessibilityService继承自AccessibilityService，其他模块向这个公共Service里注册监听器，当公共Service收到AccessibilityEvent事件时，通过监听器分发给各个模块。

### 4.3 Accessibility Service业务过程模型

核心思想：将面向过程的业务转化成面向对象的逻辑
每一种业务都可以抽象为以下流程：

<div style="text-align: center;"><svg height="784.873046875" version="1.1" width="514.166015625" xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink" style="overflow: hidden; position: relative; left: -0.5px; top: -0.75px;" viewBox="0 0 514.166015625 784.873046875" preserveAspectRatio="xMidYMid meet"><desc style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);">Created with Raphaël 2.1.2</desc><defs style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);"><path stroke-linecap="round" d="M5,0 0,2.5 5,5z" id="raphael-marker-block" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);"></path><marker id="raphael-marker-endblock33-obj16" markerHeight="3" markerWidth="3" orient="auto" refX="1.5" refY="1.5" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);"><use xmlns:xlink="http://www.w3.org/1999/xlink" xlink:href="#raphael-marker-block" transform="rotate(180 1.5 1.5) scale(0.6,0.6)" stroke-width="1.6667" fill="#666666" stroke="none" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);"></use></marker><marker id="raphael-marker-endblock33-obj17" markerHeight="3" markerWidth="3" orient="auto" refX="1.5" refY="1.5" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);"><use xmlns:xlink="http://www.w3.org/1999/xlink" xlink:href="#raphael-marker-block" transform="rotate(180 1.5 1.5) scale(0.6,0.6)" stroke-width="1.6667" fill="#666666" stroke="none" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);"></use></marker><marker id="raphael-marker-endblock33-obj19" markerHeight="3" markerWidth="3" orient="auto" refX="1.5" refY="1.5" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);"><use xmlns:xlink="http://www.w3.org/1999/xlink" xlink:href="#raphael-marker-block" transform="rotate(180 1.5 1.5) scale(0.6,0.6)" stroke-width="1.6667" fill="#666666" stroke="none" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);"></use></marker><marker id="raphael-marker-endblock33-obj21" markerHeight="3" markerWidth="3" orient="auto" refX="1.5" refY="1.5" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);"><use xmlns:xlink="http://www.w3.org/1999/xlink" xlink:href="#raphael-marker-block" transform="rotate(180 1.5 1.5) scale(0.6,0.6)" stroke-width="1.6667" fill="#666666" stroke="none" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);"></use></marker><marker id="raphael-marker-endblock33-obj23" markerHeight="3" markerWidth="3" orient="auto" refX="1.5" refY="1.5" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);"><use xmlns:xlink="http://www.w3.org/1999/xlink" xlink:href="#raphael-marker-block" transform="rotate(180 1.5 1.5) scale(0.6,0.6)" stroke-width="1.6667" fill="#666666" stroke="none" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);"></use></marker><marker id="raphael-marker-endblock33-obj25" markerHeight="3" markerWidth="3" orient="auto" refX="1.5" refY="1.5" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);"><use xmlns:xlink="http://www.w3.org/1999/xlink" xlink:href="#raphael-marker-block" transform="rotate(180 1.5 1.5) scale(0.6,0.6)" stroke-width="1.6667" fill="#666666" stroke="none" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);"></use></marker><marker id="raphael-marker-endblock33-obj26" markerHeight="3" markerWidth="3" orient="auto" refX="1.5" refY="1.5" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);"><use xmlns:xlink="http://www.w3.org/1999/xlink" xlink:href="#raphael-marker-block" transform="rotate(180 1.5 1.5) scale(0.6,0.6)" stroke-width="1.6667" fill="#666666" stroke="none" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);"></use></marker><marker id="raphael-marker-endblock33-obj27" markerHeight="3" markerWidth="3" orient="auto" refX="1.5" refY="1.5" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);"><use xmlns:xlink="http://www.w3.org/1999/xlink" xlink:href="#raphael-marker-block" transform="rotate(180 1.5 1.5) scale(0.6,0.6)" stroke-width="1.6667" fill="#666666" stroke="none" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);"></use></marker><marker id="raphael-marker-endblock33-obj28" markerHeight="3" markerWidth="3" orient="auto" refX="1.5" refY="1.5" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);"><use xmlns:xlink="http://www.w3.org/1999/xlink" xlink:href="#raphael-marker-block" transform="rotate(180 1.5 1.5) scale(0.6,0.6)" stroke-width="1.6667" fill="#666666" stroke="none" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);"></use></marker></defs><rect x="0" y="0" width="49.5625" height="35.5" rx="20" ry="20" fill="#000000" stroke="#666666" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);" fill-opacity="0" stroke-width="2" class="flowchart" id="st" transform="matrix(1,0,0,1,104.375,27.2539)"></rect><text x="10" y="17.75" text-anchor="start" font-family="&quot;Arial&quot;" font-size="14px" stroke="none" fill="#3c3c3c" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0); text-anchor: start; font-family: Arial; font-size: 14px;" id="stt" class="flowchartt" transform="matrix(1,0,0,1,104.375,27.2539)" stroke-width="1"><tspan dy="4.75" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);"> Start</tspan></text><path fill="#000000" stroke="#666666" d="M41.00390625,20.501953125L0,41.00390625L82.0078125,82.0078125L164.015625,41.00390625L82.0078125,0L0,41.00390625" stroke-width="2" fill-opacity="0" id="judgeRom" class="flowchart" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);" transform="matrix(1,0,0,1,47.1484,116.7539)"></path><text x="46.00390625" y="41.00390625" text-anchor="start" font-family="&quot;Arial&quot;" font-size="14px" stroke="none" fill="#3c3c3c" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0); text-anchor: start; font-family: Arial; font-size: 14px;" id="judgeRomt" class="flowchartt" transform="matrix(1,0,0,1,47.1484,116.7539)" stroke-width="1"><tspan dy="4.75390625" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);"> Match Rom?</tspan></text><path fill="#000000" stroke="#666666" d="M38.091796875,19.0458984375L0,38.091796875L76.18359375,76.18359375L152.3671875,38.091796875L76.18359375,0L0,38.091796875" stroke-width="2" fill-opacity="0" id="judgeRule" class="flowchart" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);" transform="matrix(1,0,0,1,52.9727,255.6738)"></path><text x="43.091796875" y="38.091796875" text-anchor="start" font-family="&quot;Arial&quot;" font-size="14px" stroke="none" fill="#3c3c3c" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0); text-anchor: start; font-family: Arial; font-size: 14px;" id="judgeRulet" class="flowchartt" transform="matrix(1,0,0,1,52.9727,255.6738)" stroke-width="1"><tspan dy="4.763671875" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);"> Math Rule?</tspan></text><rect x="0" y="0" width="250.3125" height="35.5" rx="0" ry="0" fill="#000000" stroke="#666666" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);" fill-opacity="0" stroke-width="2" class="flowchart" id="readProcess" transform="matrix(1,0,0,1,4,409.1113)"></rect><text x="10" y="17.75" text-anchor="start" font-family="&quot;Arial&quot;" font-size="14px" stroke="none" fill="#3c3c3c" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0); text-anchor: start; font-family: Arial; font-size: 14px;" id="readProcesst" class="flowchartt" transform="matrix(1,0,0,1,4,409.1113)" stroke-width="1"><tspan dy="4.75" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);">读取Process对应的Intent和ActionList</tspan></text><rect x="0" y="0" width="83.015625" height="35.5" rx="0" ry="0" fill="#000000" stroke="#666666" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);" fill-opacity="0" stroke-width="2" class="flowchart" id="executeIntent" transform="matrix(1,0,0,1,87.6484,521.8652)"></rect><text x="10" y="17.75" text-anchor="start" font-family="&quot;Arial&quot;" font-size="14px" stroke="none" fill="#3c3c3c" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0); text-anchor: start; font-family: Arial; font-size: 14px;" id="executeIntentt" class="flowchartt" transform="matrix(1,0,0,1,87.6484,521.8652)" stroke-width="1"><tspan dy="4.75" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);">执行Intent</tspan></text><rect x="0" y="0" width="108.703125" height="35.5" rx="0" ry="0" fill="#000000" stroke="#666666" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);" fill-opacity="0" stroke-width="2" class="flowchart" id="executeActions" transform="matrix(1,0,0,1,74.8047,634.6191)"></rect><text x="10" y="17.75" text-anchor="start" font-family="&quot;Arial&quot;" font-size="14px" stroke="none" fill="#3c3c3c" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0); text-anchor: start; font-family: Arial; font-size: 14px;" id="executeActionst" class="flowchartt" transform="matrix(1,0,0,1,74.8047,634.6191)" stroke-width="1"><tspan dy="4.75" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);">执行ActionList</tspan></text><rect x="0" y="0" width="44.90625" height="35.5" rx="20" ry="20" fill="#000000" stroke="#666666" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);" fill-opacity="0" stroke-width="2" class="flowchart" id="end" transform="matrix(1,0,0,1,106.7031,747.373)"></rect><text x="10" y="17.75" text-anchor="start" font-family="&quot;Arial&quot;" font-size="14px" stroke="none" fill="#3c3c3c" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0); text-anchor: start; font-family: Arial; font-size: 14px;" id="endt" class="flowchartt" transform="matrix(1,0,0,1,106.7031,747.373)" stroke-width="1"><tspan dy="4.75" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);"> End</tspan><tspan dy="16.8" x="10" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);"></tspan></text><rect x="0" y="0" width="140.5625" height="35.5" rx="0" ry="0" fill="#000000" stroke="#666666" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);" fill-opacity="0" stroke-width="2" class="flowchart" id="defaultProcess" transform="matrix(1,0,0,1,314.2148,276.0156)"></rect><text x="10" y="17.75" text-anchor="start" font-family="&quot;Arial&quot;" font-size="14px" stroke="none" fill="#3c3c3c" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0); text-anchor: start; font-family: Arial; font-size: 14px;" id="defaultProcesst" class="flowchartt" transform="matrix(1,0,0,1,314.2148,276.0156)" stroke-width="1"><tspan dy="4.75" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);">使用默认的Process</tspan></text><path fill="none" stroke="#666666" d="M129.15625,62.75390625C129.15625,62.75390625,129.15625,102.40800619125366,129.15625,113.75434533460066" stroke-width="2" marker-end="url(#raphael-marker-endblock33-obj16)" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);"></path><path fill="none" stroke="#666666" d="M129.15625,198.76171875C129.15625,198.76171875,129.15625,240.93616630649194,129.15625,252.67417172992964" stroke-width="2" marker-end="url(#raphael-marker-endblock33-obj17)" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);"></path><text x="134.15625" y="208.76171875" text-anchor="start" font-family="&quot;Arial&quot;" font-size="14px" stroke="none" fill="#3c3c3c" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0); text-anchor: start; font-family: Arial; font-size: 14px;" stroke-width="1"><tspan dy="4.76171875" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);">yes</tspan></text><path fill="none" stroke="#666666" d="M211.1640625,157.7578125C211.1640625,157.7578125,384.49609375,157.7578125,384.49609375,157.7578125C384.49609375,157.7578125,384.49609375,254.62018838198856,384.49609375,273.01836426972955" stroke-width="2" marker-end="url(#raphael-marker-endblock33-obj19)" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);"></path><text x="216.1640625" y="147.7578125" text-anchor="start" font-family="&quot;Arial&quot;" font-size="14px" stroke="none" fill="#3c3c3c" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0); text-anchor: start; font-family: Arial; font-size: 14px;" stroke-width="1"><tspan dy="4.7578125" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);">no</tspan></text><path fill="none" stroke="#666666" d="M129.15625,331.857421875C129.15625,331.857421875,129.15625,391.86622379068285,129.15625,406.1065937765334" stroke-width="2" marker-end="url(#raphael-marker-endblock33-obj21)" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);"></path><text x="134.15625" y="341.857421875" text-anchor="start" font-family="&quot;Arial&quot;" font-size="14px" stroke="none" fill="#3c3c3c" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0); text-anchor: start; font-family: Arial; font-size: 14px;" stroke-width="1"><tspan dy="4.763671875" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);">yes</tspan></text><path fill="none" stroke="#666666" d="M205.33984375,293.765625C205.33984375,293.765625,293.7009215056896,293.765625,311.2178148498351,293.765625" stroke-width="2" marker-end="url(#raphael-marker-endblock33-obj23)" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);"></path><text x="210.33984375" y="283.765625" text-anchor="start" font-family="&quot;Arial&quot;" font-size="14px" stroke="none" fill="#3c3c3c" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0); text-anchor: start; font-family: Arial; font-size: 14px;" stroke-width="1"><tspan dy="4.75" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);">no</tspan></text><path fill="none" stroke="#666666" d="M129.15625,444.611328125C129.15625,444.611328125,129.15625,504.62013004068285,129.15625,518.8605000265334" stroke-width="2" marker-end="url(#raphael-marker-endblock33-obj25)" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);"></path><path fill="none" stroke="#666666" d="M129.15625,557.365234375C129.15625,557.365234375,129.15625,617.3740362906829,129.15625,631.6144062765334" stroke-width="2" marker-end="url(#raphael-marker-endblock33-obj26)" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);"></path><path fill="none" stroke="#666666" d="M129.15625,670.119140625C129.15625,670.119140625,129.15625,730.1279425406829,129.15625,744.3683125265334" stroke-width="2" marker-end="url(#raphael-marker-endblock33-obj27)" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);"></path><path fill="none" stroke="#666666" d="M384.49609375,311.515625C384.49609375,311.515625,384.49609375,384.111328125,384.49609375,384.111328125C384.49609375,384.111328125,129.15625,384.111328125,129.15625,384.111328125C129.15625,384.111328125,129.15625,399.48477268218994,129.15625,406.12057589925826" stroke-width="2" marker-end="url(#raphael-marker-endblock33-obj28)" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);"></path></svg> </div>

#### 4.3.1 ROM——机型分类
```java
class RomItem {
    private int mId;
    private String mName;
    private List<FeatureItem> mFeatureItems;
}

class FeatureItem {
    private String mKey;
    private String mValue;
    private String mCondition;
}
```
```json
   {
      "rom_id": 1,
      "rom_name": "samsung",
      "feature_items": [
        {
          "key": "MANUFACTURER",
          "value": "samsung",
          "condition": "equal"
        },
        {
          "key": "SDK_INT",
          "value": "20",
          "condition": "greater"
        },
        {
          "key": "SDK_INT",
          "value": "23",
          "condition": "less"
        }
      ]
    }
```
#### 4.3.2 Rule——规则匹配
```java
public class RuleItem {
    int rom;
    int app;
    String title;
    int processId;
    int type;
    int priority;
}
```
```json
   {
      "rom": 1,
      "title": "samsung-正在获取通知栏权限...",
      "process_id": 2,
      "type": 2,
      "priority": 1
    },
    {
      "rom": 1,
      "title": "samsung-正在获取应用活动信息...",
      "process_id": 408,
      "type": 4,
      "priority": 1
    }
```
#### 4.3.3 Process——过程
```java
public class ProcessItem {
    int mId = -1;
    public int mIntentId = -1;
    public List<Integer> mActionIdList;
}
```
```json
   {
      "id": 2,
      "describe": "打开通知栏权限(nexus,sony,motorola, miv7)",
      "intent_id": 2,
      "action_id": [
        1001,
        9001
      ]
    }
```
#### 4.3.4 Intent——界面跳转
```java
public class IntentItem {
    public int mId = -1;
    public String mAction;
    public String mActivity;
    public String mPkgName;
    public String mData;
    public String mExtra;
}
```
```json
{
      "id":4,
      "describe":"打开获取应用活动信息权限界面(samsung 5.0以上)",
      "package":"com.android.settings",
      "action":"android.settings.USAGE_ACCESS_SETTINGS"
}
```
#### 4.3.5 Action——动作
```java
public class ActionItem extends BaseNodeInfo {
    public int mId = -1;
    public IdentifyNodeInfo mIdentifyNodeInfo;
    public LocateNodeInfo mLocateNodeInfo;
    public ScrollNodeInfo mScrollNodeInfo;
    public CheckNodeInfo mCheckNodeInfo;
    public OperationNodeInfo mOperationNodeInfo;
    public boolean mNeedWaitWindow = true;
    public int mNeedWaitTime;
}
```
```json
    {
      "id": 1004,
      "describe": "打开获取应用活动信息权限",
      "need_wait_window": true,
      "locate_node": {
        "find_texts": [
          "AppName"
        ]
      },
      "scroll_node": {
        "class_name": "android.widget.ListView"
      },
      "check_node": {
        "class_name": "android.widget.CheckBox",
        "correct_status": true
      },
      "operation_node": {
        "behavior": "click"
      }
    }
```
