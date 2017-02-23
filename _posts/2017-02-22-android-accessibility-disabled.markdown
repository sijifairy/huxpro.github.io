---
layout:     post
title:      "为何Accessibility权限总被无故关闭"
subtitle:   "Why Accessibility Service Keeps Getting Disabled"
date:       2017-02-21 12:00:00
author:     "Lizhe"
header-mask: 0.2
catalog:    true
tags:
    - Android
---


一直以来，我就注意到给app开启Accessibility权限以后会莫名其妙消失。直到昨天，某PM同学发现forceStop之后Accessibility权限一定会关闭的规律。

为何不是我发现这个规律呢？可能是因为懒。

既然发现了规律，那么一切都有迹可循。

## 一、原理

#### 1.1 ActivityManagerService
在上一篇[Magic Locker进程保活总结]()中，我们知道forceStop之后会调用ActivityManagerService的forceStopPackageLocked方法，方法里发出了Intent.ACTION_PACKAGE_RESTARTED广播。具体源码见上一篇文章。

#### 1.2 PackageMonitor
PackageMonitor是一个辅助类，用来监听应用的安装、卸载、更新等。其中就监听了Intent.ACTION_PACKAGE_RESTARTED。

它内部注册了一个BroadcastReceiver，onReceive如下
```java
@Override
public void onReceive(Context context, Intent intent) {
    // 省略N行 
    
    String action = intent.getAction();
    if (Intent.ACTION_PACKAGE_ADDED.equals(action)) {
        // 省略N行
    }
    // 省略N行 
    else if (Intent.ACTION_PACKAGE_RESTARTED.equals(action)) {
        mDisappearingPackages = new String[] {getPackageName(intent)};
        mChangeType = PACKAGE_TEMPORARY_CHANGE;
        onHandleForceStop(intent, mDisappearingPackages,
                intent.getIntExtra(Intent.EXTRA_UID, 0), true);
    }
    // 省略N行 
}
```
即PackageMonitor在收到Intent.ACTION_PACKAGE_RESTARTED后，会调用自己的onHandleForceStop方法。

#### 1.3 AccessibilityManagerService

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
        @Override
        public void onSomePackagesChanged() {
            synchronized (mLock) {
                populateAccessibilityServiceListLocked();
                manageServicesLocked();
            }
        }
        
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
                
                ……

            }
        }
        
        ……

    };
    // package changes
    monitor.register(context, true);
    // boot completed
    IntentFilter bootFiler = new IntentFilter(Intent.ACTION_BOOT_COMPLETED);
    mContext.registerReceiver(monitor, bootFiler);
}
```
在onHandleForceStop的回调用，把forceStop对应的app从mEnabledServices中移除，即将Accessibility权限关闭。

## 二、我们能做什么？

看到上面的分析，我们可以推测，之所以Accessibility权限莫名其妙消失，因为app被forceStop了。

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