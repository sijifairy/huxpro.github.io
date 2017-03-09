---
layout:     post
title:      "Android流畅度优化"
subtitle:   "Android Performance Optimization"
date:       2017-01-11 12:00:00
author:     "Lizhe"
header-mask: 0.2
catalog:    true
tags:
    - Android
    - 性能
---


# 性能优化注意事项


## 所有人相关

1. 避免大面积的动画。e.g.文件夹的打开和关闭、AppDrawer动画、QuickSettings动画、HideApps等，都是从点扩展到整个屏幕，从整个屏幕收缩成一个点。

   * 跨度大，稍微一卡就很容易看出来。
   * 动画数量多，包含alpha，scaleX，scaleY，translation等等，对硬件性能要求很高。竞品看过之后发现，大多数Launcher只有文件夹这种强相关的才使用展开动画，一般相关动画就直接alpha值、蒙黑加小距离translation就可以了。要兼顾低性能手机的表现。

2. 过渡动画可以隐藏掉不需要的细节。比如从A过渡到B，可以在动画开始时瞬间隐藏A的前景，只保留背景。这样动画时的绘制消耗几乎减小一倍。像Apus的文件夹打开这个大面积动画，点击的瞬间桌面上的所有图标都消失了，而且并没有影响动画效果。这种不损耗效果的性能提升要多考虑。

3. 可以分步骤显示内容。比如从A过渡到B，动画时只显示B的背景或者B的占位，过渡动画结束后再使用新的动画把B的内容展示出来，即分步骤展示，减轻动画的瞬时压力。比如Hola搜索动画和Apus的综合页。

4. 动画间要考虑冲突。比如Apus点击Boost开始动画的时候，风铃会消失，避免多个动画的同时进行。反观我们的launcher，天气动画、风铃动画、Boost动画可以同时进行，低配手机上肯定比较卡。

5. 动画时避免不必要的逻辑。业务逻辑给动画让步。

## DEV相关

### 工具

1. GPU Overdraw

    <img src="6.png" style="width:40%" />
    <img src="2.png" style="width:40%" />
    <br/>

   overdraw颜色和绘制次数的关系。
2. 查看Overdraw的形成
   > Tracer for OpenGL 工具

3. Show GPU profile as Bar，直观显示卡顿与否。

    <img src="11.png" style="width:40%" />
    <img src="14.png" style="width:40%" />
    <br/>

    <img src="12.png" style="width:40%" />
    <img src="13.png" style="width:40%" />
    <br/>
4. BlockCanary

   不能用太差的手机。
5. StrictMode，检查IO
6. Systrace/Method trace
7. View Hierarchy/Layout Inspector
8. Lint

### 知识点

1. 选择尽量简单的绘制方案。
   * 壁纸绘制，使用ColorFilter

     ```java
     mWallpaperPaint.setColorFilter(new LightingColorFilter(0xff262626, Color.TRANSPARENT));
     ```

   * 分段背景绘制, 新闻页

     <img src="4.png" style="width:40%"></img>

   * onDraw中使用clipRect
2. 写完功能多想想，显示B之后要隐藏A。
3. 针对复杂的Layout，使用RelativeLayout减少ViewTree深度。
4. 针对include情况，使用Merge标签减少ViewTree深度。
5. 无意义代码要放在动画开始之前。例如Flurry
6. 耗时操作要注意不能阻塞主线程。例如数据库操作、耗时Utils
7. SharedPreference分文件，无跨进程的需求一定注意。
8. 慎用TextView的ellipsize：marquee，View的fadingEdge
9. 不要使用LinearLayout的嵌套Weight布局。measure指数爆炸
10. Activity的layout如果根节点必须定义background属性，则将theme的windowBackground指定为@null
11. 重写onDraw一定不能包含耗时操作以及临时变量。例如在ondraw、getview中应减少对象申请，尽量重用。更多是一些逻辑上的东西，例如循环中不断申请局部变量等
12. xml里面View节点不能包含冗余属性。
13. 使用TextView的CompoundDrawable代替TextView+ImageView。
14. 尽量不要在Launcher上堆砌过多的View，改用Activity来实现。
	
	<img src="15.png" style="width:40%"></img>
15. 慎用Alpha
16. 用TextView/EditText的时候，如果这个TextView会运行时setText或setHint，那么它的layout\_width最好是match\_parent或固定宽度。防止requestLayout
17. LinearLayout里面有多个TextView时，注意设置baselineAligned属性为false。防止requestLayout
18. 技术选型要考虑过度绘制。
19. RecyclerView和ListView的选择。

### OverDesign

1. 复杂的Layout层级，重叠的View，复杂的逻辑
2. 开发人员无节制的View堆砌，究其根本无非是产品无节制的需求设计。
3. 由俭入奢易，由奢入俭难

