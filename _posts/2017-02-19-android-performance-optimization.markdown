---
layout:     post
title:      "Android流畅度优化之动画设计原则"
subtitle:   "Android Performance Optimization —— Animation Design Principles"
date:       2017-02-19 12:00:00
author:     "Lizhe"
header-mask: 0.2
catalog:    true
tags:
    - Android
    - 动画
    - 性能
---


<p>当Android产品从代码上优化到一定程度以后，想要进一步提升流畅度就不止是DEV的事情了。对于低端手机而言，当同一时间的显示压力大于机器的计算、绘制瓶颈时，单纯的技术手段已经不能解决卡顿的问题，这时需要从产品流程的前端考虑。</p>

经过调研、对比和思考，总结出了几点被PM和设计师们都接受的动画设计原则，如下所述。

## 时间线拆解
把同一时间进行的多个动画分散开，降低瞬时压力，即以时间换空间。
比如从A过渡到B，动画时只显示B的背景或者B的占位，过渡动画结束后再使用新的动画把B的内容展示出来。

<div>
    <img src="/img/in-post/post-android-performance-optimization/folder_add_together.gif" style="width:40%;float:left;" />
    <img src="/img/in-post/post-android-performance-optimization/folder_add_seperated.gif" style="width:40%;float:left;margin-left: 20px" />
</div>
<div style="clear: both;"></div>

## 隐藏不需要的细节
有些动画删掉一些细节不会影响交互效果，但对的性能的压力降低很多。要在效果和性能间找到平衡点。
从A过渡到B，可以在动画开始的瞬间隐藏A的前景，只保留背景。几乎看不出差别，绘制压力降低一倍。
<div>
    <img src="/img/in-post/post-android-performance-optimization/folder_open_overdraw.gif" style="width:40%;float:left;" />
    <img src="/img/in-post/post-android-performance-optimization/folder_open_no_overdraw.gif" style="width:40%;float:left;margin-left: 20px" />
</div>
<div style="clear: both;"></div>

右图在打开文件夹的瞬间隐去了桌面图标，可以看到红色的Overdraw闪烁消失了，动画更加流畅。

## 慎用大面积动画
避免大面积的动画。e.g.文件夹的打开和关闭、AppDrawer动画、QuickSettings动画、HideApps等，都是从点扩展到整个屏幕，从整个屏幕收缩成一个点。
   * 跨度大，稍微一卡就很容易看出来。
   * 动画数量多，包含alpha，scaleX，scaleY，translation等等，对硬件性能要求很高。竞品看过之后发现，大多数Launcher只有文件夹这种强相关的才使用展开动画，一般相关动画就直接alpha值、蒙黑加小距离translation就可以了。要兼顾低性能手机的表现。

## 同一界面上考虑动画冲突
动画间要考虑冲突。比如Apus点击Boost开始动画的时候，风铃会消失，避免多个动画的同时进行。反观我们的launcher，天气动画、风铃动画、Boost动画可以同时进行，低配手机上肯定比较卡。

## 动画分级
