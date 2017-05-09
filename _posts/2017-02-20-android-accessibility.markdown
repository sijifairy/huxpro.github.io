---
layout:     post
title:      "Android无障碍功能介绍"
subtitle:   "Android Accessibility Service Introduction"
date:       2017-02-20 12:00:00
author:     "Lizhe"
header-mask: 0.2
catalog:    true
tags:
    - Android
    - Accessibility
---


> Android is for Everyone,  and Everyone has the access to Everything.

Google在API 4加入无障碍设计（Accessibility）的初衷是为了服务残疾人，或者无法正常和设备交互的人，比如正在开车的司机、带小孩的母亲、视力弱化的老人等。

Google API对Accessibility服务的描述为：
* Accessibility Service应该**只被**用来辅助残障用户使用Android设备和应用。
* 它运行在后台，能够在AccessibilityEvent发生时接到系统的异步回调。
* 这些事件代表着用户界面的状态变化，例如，焦点变更、按钮点击等。
* Accessibility Service也可以通过额外的配置，获取遍历界面上所有内容的能力。

## 一、Accessibility基础知识

### 1.1 声明

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
```xml
<service android:name=".MyAccessibilityService">
	<intent-filter>
		<action android:name="android.accessibilityservice.AccessibilityService" />
	</intent-filter>
	<meta-data android:name="android.accessibilityservice" android:resource="@xml/accessibilityservice" />
 </service>
```

xml/accessibilityservice.xml
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
* canRetrieveWindowContent 是否能获取窗口能容
* canRequestFilterKeyEvents 是否接收按键消息
* notificationTime 相邻两个AccessibilityEvent接收的最小时间间隔
* description 权限描述文字
* packageNames 接收哪些应用的AccessibilityEvent

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

## 二、单例化
> Accessibility权限的赋予是以Service为单元，而不是以Application为单元。

**Ex.** 一个应用里需要用Accessibility Service做两件事
* 清理内存ForceStop
* 监听系统按键

如果想当然地定义两个Accessibility Service，就会在Accessibility设置列表里显示两个对象。
![一个应用中如果有多个Accessibility Service时](http://upload-images.jianshu.io/upload_images/4075712-ba20e8275f22acd3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

为了解决这个问题，需要定义一个公共的CommonAccessibility Service继承自Accessibility Service，其他模块向这个公共Service里注册监听器，当公共Service收到onAccessibilityEvent时，通过监听器分发给各个模块。

## 三、业务过程模型

核心思想：将面向过程的业务转化成面向对象的逻辑<br/>
每一种业务都可以抽象为以下流程：

<div style="text-align: center;"><svg height="784.873046875" version="1.1" width="514.166015625" xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink" style="overflow: hidden; position: relative; left: -0.5px; top: -0.75px;" viewBox="0 0 514.166015625 784.873046875" preserveAspectRatio="xMidYMid meet"><desc style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);">Created with Raphaël 2.1.2</desc><defs style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);"><path stroke-linecap="round" d="M5,0 0,2.5 5,5z" id="raphael-marker-block" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);"></path><marker id="raphael-marker-endblock33-obj16" markerHeight="3" markerWidth="3" orient="auto" refX="1.5" refY="1.5" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);"><use xmlns:xlink="http://www.w3.org/1999/xlink" xlink:href="#raphael-marker-block" transform="rotate(180 1.5 1.5) scale(0.6,0.6)" stroke-width="1.6667" fill="#666666" stroke="none" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);"></use></marker><marker id="raphael-marker-endblock33-obj17" markerHeight="3" markerWidth="3" orient="auto" refX="1.5" refY="1.5" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);"><use xmlns:xlink="http://www.w3.org/1999/xlink" xlink:href="#raphael-marker-block" transform="rotate(180 1.5 1.5) scale(0.6,0.6)" stroke-width="1.6667" fill="#666666" stroke="none" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);"></use></marker><marker id="raphael-marker-endblock33-obj19" markerHeight="3" markerWidth="3" orient="auto" refX="1.5" refY="1.5" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);"><use xmlns:xlink="http://www.w3.org/1999/xlink" xlink:href="#raphael-marker-block" transform="rotate(180 1.5 1.5) scale(0.6,0.6)" stroke-width="1.6667" fill="#666666" stroke="none" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);"></use></marker><marker id="raphael-marker-endblock33-obj21" markerHeight="3" markerWidth="3" orient="auto" refX="1.5" refY="1.5" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);"><use xmlns:xlink="http://www.w3.org/1999/xlink" xlink:href="#raphael-marker-block" transform="rotate(180 1.5 1.5) scale(0.6,0.6)" stroke-width="1.6667" fill="#666666" stroke="none" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);"></use></marker><marker id="raphael-marker-endblock33-obj23" markerHeight="3" markerWidth="3" orient="auto" refX="1.5" refY="1.5" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);"><use xmlns:xlink="http://www.w3.org/1999/xlink" xlink:href="#raphael-marker-block" transform="rotate(180 1.5 1.5) scale(0.6,0.6)" stroke-width="1.6667" fill="#666666" stroke="none" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);"></use></marker><marker id="raphael-marker-endblock33-obj25" markerHeight="3" markerWidth="3" orient="auto" refX="1.5" refY="1.5" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);"><use xmlns:xlink="http://www.w3.org/1999/xlink" xlink:href="#raphael-marker-block" transform="rotate(180 1.5 1.5) scale(0.6,0.6)" stroke-width="1.6667" fill="#666666" stroke="none" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);"></use></marker><marker id="raphael-marker-endblock33-obj26" markerHeight="3" markerWidth="3" orient="auto" refX="1.5" refY="1.5" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);"><use xmlns:xlink="http://www.w3.org/1999/xlink" xlink:href="#raphael-marker-block" transform="rotate(180 1.5 1.5) scale(0.6,0.6)" stroke-width="1.6667" fill="#666666" stroke="none" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);"></use></marker><marker id="raphael-marker-endblock33-obj27" markerHeight="3" markerWidth="3" orient="auto" refX="1.5" refY="1.5" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);"><use xmlns:xlink="http://www.w3.org/1999/xlink" xlink:href="#raphael-marker-block" transform="rotate(180 1.5 1.5) scale(0.6,0.6)" stroke-width="1.6667" fill="#666666" stroke="none" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);"></use></marker><marker id="raphael-marker-endblock33-obj28" markerHeight="3" markerWidth="3" orient="auto" refX="1.5" refY="1.5" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);"><use xmlns:xlink="http://www.w3.org/1999/xlink" xlink:href="#raphael-marker-block" transform="rotate(180 1.5 1.5) scale(0.6,0.6)" stroke-width="1.6667" fill="#666666" stroke="none" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);"></use></marker></defs><rect x="0" y="0" width="49.5625" height="35.5" rx="20" ry="20" fill="#000000" stroke="#666666" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);" fill-opacity="0" stroke-width="2" class="flowchart" id="st" transform="matrix(1,0,0,1,104.375,27.2539)"></rect><text x="10" y="17.75" text-anchor="start" font-family="&quot;Arial&quot;" font-size="14px" stroke="none" fill="#3c3c3c" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0); text-anchor: start; font-family: Arial; font-size: 14px;" id="stt" class="flowchartt" transform="matrix(1,0,0,1,104.375,27.2539)" stroke-width="1"><tspan dy="4.75" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);"> Start</tspan></text><path fill="#000000" stroke="#666666" d="M41.00390625,20.501953125L0,41.00390625L82.0078125,82.0078125L164.015625,41.00390625L82.0078125,0L0,41.00390625" stroke-width="2" fill-opacity="0" id="judgeRom" class="flowchart" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);" transform="matrix(1,0,0,1,47.1484,116.7539)"></path><text x="46.00390625" y="41.00390625" text-anchor="start" font-family="&quot;Arial&quot;" font-size="14px" stroke="none" fill="#3c3c3c" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0); text-anchor: start; font-family: Arial; font-size: 14px;" id="judgeRomt" class="flowchartt" transform="matrix(1,0,0,1,47.1484,116.7539)" stroke-width="1"><tspan dy="4.75390625" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);"> Match Rom?</tspan></text><path fill="#000000" stroke="#666666" d="M38.091796875,19.0458984375L0,38.091796875L76.18359375,76.18359375L152.3671875,38.091796875L76.18359375,0L0,38.091796875" stroke-width="2" fill-opacity="0" id="judgeRule" class="flowchart" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);" transform="matrix(1,0,0,1,52.9727,255.6738)"></path><text x="43.091796875" y="38.091796875" text-anchor="start" font-family="&quot;Arial&quot;" font-size="14px" stroke="none" fill="#3c3c3c" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0); text-anchor: start; font-family: Arial; font-size: 14px;" id="judgeRulet" class="flowchartt" transform="matrix(1,0,0,1,52.9727,255.6738)" stroke-width="1"><tspan dy="4.763671875" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);"> Math Rule?</tspan></text><rect x="0" y="0" width="250.3125" height="35.5" rx="0" ry="0" fill="#000000" stroke="#666666" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);" fill-opacity="0" stroke-width="2" class="flowchart" id="readProcess" transform="matrix(1,0,0,1,4,409.1113)"></rect><text x="10" y="17.75" text-anchor="start" font-family="&quot;Arial&quot;" font-size="14px" stroke="none" fill="#3c3c3c" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0); text-anchor: start; font-family: Arial; font-size: 14px;" id="readProcesst" class="flowchartt" transform="matrix(1,0,0,1,4,409.1113)" stroke-width="1"><tspan dy="4.75" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);">读取Process对应的Intent和ActionList</tspan></text><rect x="0" y="0" width="83.015625" height="35.5" rx="0" ry="0" fill="#000000" stroke="#666666" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);" fill-opacity="0" stroke-width="2" class="flowchart" id="executeIntent" transform="matrix(1,0,0,1,87.6484,521.8652)"></rect><text x="10" y="17.75" text-anchor="start" font-family="&quot;Arial&quot;" font-size="14px" stroke="none" fill="#3c3c3c" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0); text-anchor: start; font-family: Arial; font-size: 14px;" id="executeIntentt" class="flowchartt" transform="matrix(1,0,0,1,87.6484,521.8652)" stroke-width="1"><tspan dy="4.75" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);">执行Intent</tspan></text><rect x="0" y="0" width="108.703125" height="35.5" rx="0" ry="0" fill="#000000" stroke="#666666" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);" fill-opacity="0" stroke-width="2" class="flowchart" id="executeActions" transform="matrix(1,0,0,1,74.8047,634.6191)"></rect><text x="10" y="17.75" text-anchor="start" font-family="&quot;Arial&quot;" font-size="14px" stroke="none" fill="#3c3c3c" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0); text-anchor: start; font-family: Arial; font-size: 14px;" id="executeActionst" class="flowchartt" transform="matrix(1,0,0,1,74.8047,634.6191)" stroke-width="1"><tspan dy="4.75" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);">执行ActionList</tspan></text><rect x="0" y="0" width="44.90625" height="35.5" rx="20" ry="20" fill="#000000" stroke="#666666" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);" fill-opacity="0" stroke-width="2" class="flowchart" id="end" transform="matrix(1,0,0,1,106.7031,747.373)"></rect><text x="10" y="17.75" text-anchor="start" font-family="&quot;Arial&quot;" font-size="14px" stroke="none" fill="#3c3c3c" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0); text-anchor: start; font-family: Arial; font-size: 14px;" id="endt" class="flowchartt" transform="matrix(1,0,0,1,106.7031,747.373)" stroke-width="1"><tspan dy="4.75" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);"> End</tspan><tspan dy="16.8" x="10" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);"></tspan></text><rect x="0" y="0" width="140.5625" height="35.5" rx="0" ry="0" fill="#000000" stroke="#666666" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);" fill-opacity="0" stroke-width="2" class="flowchart" id="defaultProcess" transform="matrix(1,0,0,1,314.2148,276.0156)"></rect><text x="10" y="17.75" text-anchor="start" font-family="&quot;Arial&quot;" font-size="14px" stroke="none" fill="#3c3c3c" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0); text-anchor: start; font-family: Arial; font-size: 14px;" id="defaultProcesst" class="flowchartt" transform="matrix(1,0,0,1,314.2148,276.0156)" stroke-width="1"><tspan dy="4.75" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);">使用默认的Process</tspan></text><path fill="none" stroke="#666666" d="M129.15625,62.75390625C129.15625,62.75390625,129.15625,102.40800619125366,129.15625,113.75434533460066" stroke-width="2" marker-end="url(#raphael-marker-endblock33-obj16)" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);"></path><path fill="none" stroke="#666666" d="M129.15625,198.76171875C129.15625,198.76171875,129.15625,240.93616630649194,129.15625,252.67417172992964" stroke-width="2" marker-end="url(#raphael-marker-endblock33-obj17)" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);"></path><text x="134.15625" y="208.76171875" text-anchor="start" font-family="&quot;Arial&quot;" font-size="14px" stroke="none" fill="#3c3c3c" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0); text-anchor: start; font-family: Arial; font-size: 14px;" stroke-width="1"><tspan dy="4.76171875" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);">yes</tspan></text><path fill="none" stroke="#666666" d="M211.1640625,157.7578125C211.1640625,157.7578125,384.49609375,157.7578125,384.49609375,157.7578125C384.49609375,157.7578125,384.49609375,254.62018838198856,384.49609375,273.01836426972955" stroke-width="2" marker-end="url(#raphael-marker-endblock33-obj19)" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);"></path><text x="216.1640625" y="147.7578125" text-anchor="start" font-family="&quot;Arial&quot;" font-size="14px" stroke="none" fill="#3c3c3c" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0); text-anchor: start; font-family: Arial; font-size: 14px;" stroke-width="1"><tspan dy="4.7578125" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);">no</tspan></text><path fill="none" stroke="#666666" d="M129.15625,331.857421875C129.15625,331.857421875,129.15625,391.86622379068285,129.15625,406.1065937765334" stroke-width="2" marker-end="url(#raphael-marker-endblock33-obj21)" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);"></path><text x="134.15625" y="341.857421875" text-anchor="start" font-family="&quot;Arial&quot;" font-size="14px" stroke="none" fill="#3c3c3c" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0); text-anchor: start; font-family: Arial; font-size: 14px;" stroke-width="1"><tspan dy="4.763671875" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);">yes</tspan></text><path fill="none" stroke="#666666" d="M205.33984375,293.765625C205.33984375,293.765625,293.7009215056896,293.765625,311.2178148498351,293.765625" stroke-width="2" marker-end="url(#raphael-marker-endblock33-obj23)" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);"></path><text x="210.33984375" y="283.765625" text-anchor="start" font-family="&quot;Arial&quot;" font-size="14px" stroke="none" fill="#3c3c3c" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0); text-anchor: start; font-family: Arial; font-size: 14px;" stroke-width="1"><tspan dy="4.75" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);">no</tspan></text><path fill="none" stroke="#666666" d="M129.15625,444.611328125C129.15625,444.611328125,129.15625,504.62013004068285,129.15625,518.8605000265334" stroke-width="2" marker-end="url(#raphael-marker-endblock33-obj25)" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);"></path><path fill="none" stroke="#666666" d="M129.15625,557.365234375C129.15625,557.365234375,129.15625,617.3740362906829,129.15625,631.6144062765334" stroke-width="2" marker-end="url(#raphael-marker-endblock33-obj26)" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);"></path><path fill="none" stroke="#666666" d="M129.15625,670.119140625C129.15625,670.119140625,129.15625,730.1279425406829,129.15625,744.3683125265334" stroke-width="2" marker-end="url(#raphael-marker-endblock33-obj27)" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);"></path><path fill="none" stroke="#666666" d="M384.49609375,311.515625C384.49609375,311.515625,384.49609375,384.111328125,384.49609375,384.111328125C384.49609375,384.111328125,129.15625,384.111328125,129.15625,384.111328125C129.15625,384.111328125,129.15625,399.48477268218994,129.15625,406.12057589925826" stroke-width="2" marker-end="url(#raphael-marker-endblock33-obj28)" style="-webkit-tap-highlight-color: rgba(0, 0, 0, 0);"></path></svg> </div>

#### 3.1 Rom——机型分类
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
#### 3.2 Rule——规则匹配
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
#### 3.3 Process——过程
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
#### 3.4 Intent——界面跳转
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
#### 3.5 Action——动作
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
#### 3.6 Json实体转换

## 四、实际应用

### 4.1 获取TopActivity

获取TopActivity（当前前台应用）的几种方式
1. Lollipop之前, 使用ActivityManage的getRunningTasks()方法
2. Lollipop及以后，使用AppUsageAccess方法
3. AccessibiltyService方式获取

#### 4.1.1 getRunningTasks()方法
```java
ActivityManager activityManager = (ActivityManager)context.getApplicationContext().getSystemService(Context.ACTIVITY_SERVICE);
ComponentName runningTopActivity = activityManager.getRunningTasks(1).get(0).topActivity;
```
需要声明权限
```xml
<uses-permission android:name="android.permission.GET_TASKS" />
```
这种方法不止能获取包名，还能获取Activity名。但是在Android 5.0以后，系统就不再对第三方应用提供这种方式来获取前台应用了，虽然调用这个方法还是能够返回结果，但是结果只包含你自己的Activity和Launcher了。

#### 4.1.2 App Usage Access方法
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
代码的功能是通过UsageStatsManager 来获取用户使用的程序的列表，然后按照最近使用时间排序，就得到了当前的前台应用，这种方式只能拿到包名，无法精确了Activity了。
使用这种方发之前，首先要引导用户开启App Usage Access权限。

需要声明权限
```xml
<uses-permission android:name="android.permission.PACKAGE_USAGE_STATS" />
```
libDeviceMonitor中获取TopActivity就是用的这种方式。

#### 4.1.3 AccessibilityService获取
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
**涉及到的AccessibilityService相关知识点：**
* AccessibilityEvent.TYPE_WINDOW_STATE_CHANGED
    - `TYPE_VIEW_CLICKED`  View点击事件
    - `TYPE_VIEW_LONG_CLICKED`  View长按事件
    - `TYPE_VIEW_TEXT_CHANGED` EditText内容变化事件
    - `TYPE_VIEW_SCROLLED` 界面滑动结束
    - `TYPE_WINDOW_STATE_CHANGED` 代表任何界面的跳转，如Activity、Dialog、PopupWindow等
* AccessibilityEvent.getClassName()
* AccessibilityEvent.getPackageName()

### 4.2 监听系统按键

其实只要实现了AccessibilityService的onKeyEvent方法就可以获得按键事件了。
先看代码：
```java
public class BigBangMonitorService extends AccessibilityService {

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
        KeyPressedTipViewController.getInstance().onKeyEvent(paramKeyEvent);
        return false;
    }

    @Override
    public void onAccessibilityEvent(AccessibilityEvent event) {
        //虚拟手机按键处理，优先级高于是否点击分词的判断
        if ((event.getEventType() == TYPE_VIEW_LONG_CLICKED) && ("com.android.systemui".equals(event.getPackageName())))
        {
            if (TextUtils.isEmpty(event.getContentDescription())){
                return;
            }
            //长按虚拟机触发的，需要转到按键处理去
            if (!TextUtils.isEmpty(back) && event.getContentDescription().equals(back)){
                KeyPressedTipViewController.getInstance().onKeyLongPress(KeyEvent.KEYCODE_BACK);
            }else if (!TextUtils.isEmpty(home) && event.getContentDescription().equals(home)){
                KeyPressedTipViewController.getInstance().onKeyLongPress(KeyEvent.KEYCODE_HOME);
            }else if (!TextUtils.isEmpty(recent) && event.getContentDescription().equals(recent)){
                KeyPressedTipViewController.getInstance().onKeyLongPress(KeyEvent.KEYCODE_APP_SWITCH);
            }
        }
    }
}
```
1. 在onCreate中需要设置flag，从flag的名字上就能看出是用于声明接收key事件的，其实等同于在xml中的accessibilityFlags中添加。
2. 跟onAccessibilityEvent中的点击事件一样，这里收到的也是异步的回调，对这些paramKeyEvent进行的修改不会影响到应用中接收事件。
3. 你可能注意到了onAccessibilityEvent中做的处理，这是为了兼容具有NavigationBar（虚拟导航栏）的手机。因为在这些手机上，点击返回、Home、多任务三个按钮的时候，是作为普通的点击事件来处理的，也就是onAccessibilityEvent中的回调。至于代码中我只处理了TYPE_VIEW_LONG_CLICKED，则是因为单击这些键触发菜单显然是不合适的，所以没有做单击的触发方式。
4. KeyPressedTipViewController中的处理是跟业务逻辑相关的处理，可以不用关心，反正已经得到按键事件了，做爱做的事就行了。

### 4.3 获取其他App中的文字
```java
private CharSequence mWindowClassName;
private String mCurrentPackage;
private int mCurrentType;
private Map<String,Integer> selections;//保存每个应用的包名对应的触发方式
private boolean onlyText = true;
public  int double_click_interval = 1000;
@Override
public void onAccessibilityEvent(AccessibilityEvent event) {
    int type=event.getEventType();
    switch (type){
        case AccessibilityEvent.TYPE_WINDOW_STATE_CHANGED:
            mWindowClassName = event.getClassName();
            mCurrentPackage = event.getPackageName()==null?"":event.getPackageName().toString();
            Integer selectType=selections.get(mCurrentPackage);
            mCurrentType = selectType==null?TYPE_VIEW_NONE:(selectType+1);                
            break;
        case TYPE_VIEW_CLICKED:
        case TYPE_VIEW_LONG_CLICKED:
            getText(event);
            break;

}


private synchronized void getText(AccessibilityEvent event){        
    int type=getClickType(event);
    CharSequence className = event.getClassName();
    if (mWindowClassName==null){
        return;
    }
    if (mWindowClassName.toString().startsWith("com.forfan.bigbang")){
        //自己的应用不监控
        return;
    }
    if (mCurrentPackage.equals(event.getPackageName())){        
        if (type!=mCurrentType){
            //点击方式不匹配，直接返回
            return;
        }
    }else {
        //包名不匹配，直接返回
        return;
    }
    if (className==null || className.equals("android.widget.EditText")){
        //输入框不监控
        return;
    }
    if (onlyText){
        //onlyText方式下，只获取TextView的内容
        if (className==null || !className.equals("android.widget.TextView")){
            if (!hasShowTipToast){
                ToastUtil.show(R.string.toast_tip_content);
                hasShowTipToast=true;
            }
            return;
        }
    }
    AccessibilityNodeInfo info=event.getSource();
    if(info==null){
        return;
    }
    CharSequence txt=info.getText();
    if (TextUtils.isEmpty(txt) && !onlyText){
        //非onlyText方式下获取文字更多，但是可能并不是想要的文字
        //比如系统短信页面需要这样才能获取到内容。
        List<CharSequence> txts=event.getText();
        if (txts!=null) {
            StringBuilder sb=new StringBuilder();
            for (CharSequence t : txts) {
                sb.append(t);
            }
            txt=sb.toString();
        }
    }
    if (!TextUtils.isEmpty(txt)) {
        if (txt.length()<=2 ){
            //对于太短的词进行屏蔽，因为这些词往往是“发送”等功能按钮，其实应该根据不同的activity进行区分
            if (!hasShowTooShortToast) {
                ToastUtil.show(R.string.too_short_to_split);
                hasShowTooShortToast = true;
            }
            return;
        }
        //打开分词功能
        Intent intent=new Intent(this, BigBangActivity.class);
        intent.addFlags(intent.FLAG_ACTIVITY_NEW_TASK);
        intent.putExtra(BigBangActivity.TO_SPLIT_STR,txt.toString());
        startActivity(intent);
    }
}


private Method getSourceNodeIdMethod;
private long mLastSourceNodeId;
private long mLastClickTime;

private long getSourceNodeId(AccessibilityEvent event)  {
    //用于获取点击的View的id，用于检测双击操作
    if (getSourceNodeIdMethod==null) {
        Class<AccessibilityEvent> eventClass = AccessibilityEvent.class;
        try {
            getSourceNodeIdMethod = eventClass.getMethod("getSourceNodeId");
        } catch (NoSuchMethodException e) {
            e.printStackTrace();
        }
    }
    if (getSourceNodeIdMethod!=null) {
        try {
            return (long) getSourceNodeIdMethod.invoke(event);
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (InvocationTargetException e) {
            e.printStackTrace();
        }
    }
    return -1;
}

private int getClickType(AccessibilityEvent event){
    int type = event.getEventType();
    long time = event.getEventTime();
    long id=getSourceNodeId(event);
    if (type!=TYPE_VIEW_CLICKED){
        mLastClickTime=time;
        mLastSourceNodeId=-1;
        return type;
    }
    if (id==-1){
        mLastClickTime=time;
        mLastSourceNodeId=-1;
        return type;
    }
    if (type==TYPE_VIEW_CLICKED && time - mLastClickTime<= double_click_interval && id==mLastSourceNodeId){
        mLastClickTime=-1;
        mLastSourceNodeId=-1;
        return TYPE_VIEW_DOUBLD_CLICKED;
    }else {
        mLastClickTime=time;
        mLastSourceNodeId=id;
        return type;
    }
}
```

相关知识点：
* AccessibilityEvent.getSource()
* AccessibilityNodeInfo.getText()

### 4.4 实现全局复制

### 4.5 清理内存（ForceStop)
安全清理类应用清理内存的机制可能大多数Android开发者并不清楚，但是即便是普通用户也知道一个app可以在设置->应用管理里面强制停止（ForceStop）。没错，安全类应用的深度清理就是把Accessibility Service给予的模拟点击功能和FoceStop结合了起来。

![深度内存清理](http://upload-images.jianshu.io/upload_images/4075712-0ec630504378ec77.gif?imageMogr2/auto-orient/strip)

上面动画清晰的显示了内存清理的工作机制：

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
- 最后调起Settings的AppInfo，把前面调起的界面杀掉
- 存在Settings界面还在的可能，这时候发出GLOBAL\_ACTION\_BACK
**涉及到的AccessibilityService相关知识点**
我们可以向AccessibilityNodeInfo发送Action，常用Action如下：
+ `ACTION_CLICK` 点击动作
+ `ACTION_LONG_CLICK` 长按动作
+ `ACTION_SCROLL_UP` 向上滑动
+ `ACTION_SCROLL_DOWN` 向下滑动
+ `ACTION_SET_TEXT` 设置文字内容
```java
scrollNode.performAction(AccessibilityNodeInfo.ACTION_SCROLL_FORWARD)
```

除此以外，还可以向AccessibilityService发出GlobalAction：
+ `GLOBAL_ACTION_BACK` Back键
+ `GLOBAL_ACTION_HOME` Home键
+ `GLOBAL_ACTION_RECENTS` Recent键
```java
mAccessibilityService.performGlobalAction(GLOBAL_ACTION_BACK)
```

### 4.6 自动权限获取
工具类应用一般需要引导用户获取各种权限，比如Notification Listening、5.0以上的App Usage Access、设备管理器权限、悬浮窗权限等等。而用户经常是没有耐心去开启这么多权限的，所以我们可以在拿到Accessibility权限以后，静默开启这些权限。

![权限静默开启](http://upload-images.jianshu.io/upload_images/4075712-093a8c1313f91e9b.gif?imageMogr2/auto-orient/strip)

上图中就自动开启了Notification Listening、App Usage Access、设备管理器权限。

50%透明红色的覆盖可以替换成不透明的动画，这样就可以在不干扰用户视觉的前提下，帮助用户开启需要的权限。

### 4.7 微信检查被删好友

### 4.8 DevTools

### 4.9 ~~抢红包插件~~

### 4.10 ~~拦截和获取短信验证码~~

### 4.11 ~~窃取各种隐私、聊天记录等~~

## 五、 开发符合Accessibility规范的应用

当App用户数或流行度增长到某个级别之后，需要考虑App的Accessibility性能，来保证App在特殊情况下对用户友好。
大厂们都对这个方向比较重视，因为能标榜自己产品的品味、人性化等。
Facebook还做了Accessibility Debug工具。

浏览App的导航方式：
* Touch Screen，最普遍
* External Hareware（DPad，trackball, keyboard, etc.），其他肢体残缺人士的辅助设施
* TalkBack (an accessibility service published by Google)，盲人

#### 5.1 Focus Navigation
Focus Navigation是Android平台内置的一种导航方式。
忽视它会导致：
* 某些按钮不能被激活交互
* 文本的阅读顺序混乱

#### 5.2 问题举例
```xml
<LinearLayout
        xmlns:android="http://schemas.android.com/apk/res/android"
        android:orientation="vertical"
        android:layout_width="match_parent"
        android:layout_height="wrap_content">
        <TextView
              android:text="Heading One"
              android:layout_width="match_parent"
              android:layout_height="wrap_content" />
        <Button
              android:text="Action"
              android:layout_width="wrap_content"
              android:layout_height="wrap_content" />
        <TextView
              android:text="Heading Two"
              android:layout_width="match_parent"
              android:layout_height="wrap_content" />
</LinearLayout>
```
**Focusable的view是哪些？**
A. the TextViews and the Button
B. the LinearLayout and the Button

Screen Reader的发声顺序为Heading One，Heading Two，Action。

#### 5.3 如何判断一个View是否是Focusable的

![流程图](/img/in-post/post-android-accessibility/accessibility_focus.png)

#### 5.4 Stetho
[通过Stetho调试应用的Accessibility性能](https://code.facebook.com/posts/391276077927020/android-accessibility-debugging-with-stetho/)
