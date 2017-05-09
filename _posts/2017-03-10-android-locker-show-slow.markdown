---
layout:     post
title:      "Android自定义锁屏显示速度的优化"
subtitle:   "Optimization of Charing Screen's Display Speed"
date:       2017-03-10 00:00:00
author:     "Lizhe"
header-mask: 0.2
catalog:    true
tags:
    - Android
    - 锁屏
---

## 1. 问题提出
在做公司充电锁屏feature的时候发现锁屏Activity出现的太慢，探讨一下原因。

充电锁屏Activity出现分两步
* 注册并接收系统SCREEN_ON和SCREEN_OFF广播
* 接收到广播后，使用startActivity将充电锁屏启动

代码如下
```java
final IntentFilter screenFilter = new IntentFilter();
screenFilter.addAction(Intent.ACTION_SCREEN_OFF);
screenFilter.addAction(Intent.ACTION_SCREEN_ON);

HSApplication.getContext().registerReceiver(new BroadcastReceiver() {
    @Override
    public void onReceive(Context context, Intent intent) {
        if (intent.getAction().equals(Intent.ACTION_SCREEN_OFF)) {
            startChargingScreenActivity();
        } else if (intent.getAction().equals(Intent.ACTION_SCREEN_ON)) {
            startChargingScreenActivity();
        }
    }
}, screenFilter);

public static void startChargingScreenActivity() {
    Intent intent = new Intent(HSApplication.getContext(), ChargingScreenActivity.class);
    intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK
            | Intent.FLAG_ACTIVITY_CLEAR_TOP | Intent.FLAG_ACTIVITY_NO_ANIMATION);
    HSApplication.getContext().startActivity(intent);
}
```

效果如下，在Launcher里灭屏并立即亮屏，可以看到亮屏后不能立刻显示锁屏界面。
<video width="40%" src="../../../../img/in-post/post-android-locker-show-slow/before.mp4"  controls="controls" autoplay="autoplay"></video>

## 2. 问题分析
想要优化这个显示速度，只能从两方面下手，即
* 优化系统广播SCREEN_ON和SCREEN_OFF的接收速度
* 优化Activity的显示速度。

### 2.1 加快Broadcast接收速度

### 2.2 加快Activity显示速度
