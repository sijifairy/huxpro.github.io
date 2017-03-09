---
layout:     post
title:      "Android源码中的那些彩蛋"
subtitle:   "Easter Eggs in AOSP"
date:       2017-03-06 00:00:00
author:     "Lizhe"
header-mask: 0.2
catalog:    true
tags:
    - Android
---


最近看android源码的时候遇到了一些很好玩的代码，总结出来搞个乐子。

## 1. Let's party like it's 1995!

```java
View createViewFromTag(View parent, String name, Context context, AttributeSet attrs, boolean ignoreThemeAttr) {

        private static final String TAG_1995 = "blink";
[...]

        if (name.equals(TAG_1995)) {
            // Let's party like it's 1995!
            return new BlinkLayout(context, attrs);
        }

[...]
```

LayoutInflater的createViewFromTag方法是用来解析单个xml节点的，但我再里面很惊奇地发现一个常量叫TAG_1995。它的这个命名很奇葩，而且和它相关的这个BlinkLayout从来没听说过。所以去AOSP里找到这行的初始提交。

提交记录在[link](https://android.googlesource.com/platform/frameworks/base/+/9c1223a71397b565f38015c07cae57a5015a6500%5E!/core/java/android/view/LayoutInflater.java)

提交的comment如下：
>Improve LayoutInflater's compliance.
>
>There are standards, we should do our best to implement them properly.

并不能理解这个comment的意思，提升了LayoutInflater的柔顺度？在LayoutInlater定义了一个BlinkLayout能提升柔顺度？如果一定要搞清楚它的含义，大概只能去问Romain Guy <romainguy@google.com> 这位author了。

## 