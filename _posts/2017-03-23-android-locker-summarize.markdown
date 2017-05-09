---
layout:     post
title:      "Android锁屏的选型方式——Activity or Window"
subtitle:   "How to Make a Locker——Use Activity or Window"
date:       2017-03-23 00:00:00
author:     "Lizhe"
header-mask: 0.2
catalog:    true
---

锁屏根据产品类型可分为
* 独立锁屏应用，如CM Locker、Magic Locker等。
  这种锁屏的价值在于向用户提供个性化、安全性服务。
* 应用内锁屏Feature，如各Launcher、各清理软件、各工具软件内带有的锁屏功能。
  这种锁屏仅仅是一种流量变现的手段，利用用户的麻木或愚昧，抢占高频锁屏场景。通常带有大幅原生广告甚至全屏广告，并不向用户提供个性化、安全性服务。

根据技术方案也可分为两大类
* 锁屏主体为View，直接向Window上添加，使用后台常驻Service的Context对象。
  独立锁屏应用一般采用这种方式。
* 锁屏主题为Activity，有自己的Context和生命周期。
  应用内锁屏一般采用这种方式。

Window方式的优缺点：
* 优点
  + 按Home键可以不消失。
  + 使用type为TYPE_ERROR的悬浮窗可以屏蔽StatusBar和NavigationBar。
  + 通常使用常驻并提升了进程优先级的Service管理，稳定性较好。
* 缺点
  + 前期学习成本较高，需要对Window的管理有较多理解。
  + 风险较大，需要考虑来电、闹钟、Toast、关机Dialog等的处理，处理不好容易出事故。
  + 需要自己管理对象的生命周期，自行处理常驻or回收。

相对应地，Activity方式的优缺点：
* 优点
  + 学习成本较低，较为传统。
  + 风险较低，毕竟只是个Activity。
  + 方便管理对象生命周期，方便LeakCanary检测内存泄漏。
* 缺点
  + 按Home键必然消失，逃不过Home键的管理。
  + 只能做到半屏蔽StatusBar和NavigationBar。
  + 依赖于Application的存活。
  + 存在一些坑，比如过场动画、机型碎片问题，难以解决，体验一般。

结论：
* 推荐使用Window方式，能做到极致的用户体验。
* Window方式学习成本较高，维护成本较低；Activity方式反之。

关于使用两种方式实现简单的锁屏应用，参见我的github:
