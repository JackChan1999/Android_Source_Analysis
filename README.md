## Android 开源项目源码解析

[android-open-project-analysis](https://github.com/android-cn/android-open-project-analysis)

这是一个协作项目，最终多数开源库原理解析会在这里分享出来

## Android源码设计模式分析项目

[android_design_patterns_analysis](https://github.com/simple-android-framework-exchange/android_design_patterns_analysis)

该项目通过分析Android系统中的设计模式来提升大家对设计模式的理解，从源码的角度来剖析既增加了对Android系统本身的了解，也从优秀的设计中领悟模式的实际运用以及它适用的场景，避免在实际开发中的生搬硬套。

## Android知名开源库简单实现以及设计分析

[simple-android-opensource-framework](https://github.com/simple-android-framework-exchange/simple-android-opensource-framework)

该项目通过分析并实现Android平台知名开源框架的简单版本来提升自我，并达到深入理解各大开源库的核心原理的目的。稳定、强大的开源库一般都较为复杂，比如Universal-ImageLoader，因此简版开源库不需要完全按照原版来实现，只需要把核心架构、原理实现，并且做到可运用到实际项目中即可。在实现开源库简版的同时，作者需要写一系列文章来剖析它的实现原理以及为什么要这么设计，在提升自我的同时将框架的设计与实现、领悟分享给他人，希望大家在提升自我的同时对行业做出一些贡献。

## Android SDK 源码解析

[AndroidSdkSourceAnalysis](https://github.com/LittleFriendsGroup/AndroidSdkSourceAnalysis)

android sdk 源码解析——旨在帮助Android开发者更好的学习Android！我们只是一群普通的程序员，但是，我们热爱分享，想热热闹闹的玩点有意义的事！如果你也想陪我们一起愉快的玩耍，欢迎加入我们！Issues认领分析文章！

- [PDF、Mobi、ePub下载](https://www.gitbook.com/book/alleniverson/android-source-analysis/details)
- [Android源码分析.pdf](http://download.csdn.net/detail/axi295309066/9788564)
- [在线阅读](https://www.gitbook.com/book/alleniverson/android-source-analysis/content)
- [GitHub托管](https://github.com/JackChen1999/Android_Source_Analysis)


## 目录

* [前言](README.md)

* [Android源码分析 公共技术点](common\README.md)
  * [公共技术点之Java反射](common/公共技术点之Java反射.md)
  * [公共技术点之Java注解](common/公共技术点之Java注解.md)
  * [公共技术点之Java动态代理](common/公共技术点之Java动态代理.md)
  * [公共技术点之View绘制流程](common/公共技术点之View绘制流程.md)
  * [公共技术点之View事件传递](common/公共技术点之View事件传递.md)
  * [公共技术点之Android动画基础](common/公共技术点之Android动画基础.md)

* [Android源码分析 第1期](chapter1/README.md)
  * [AsyncTask源码分析](chapter1/AsyncTask源码分析.md)
  * [你真的了解AsyncTask](chapter1/你真的了解AsyncTask.md)
  * [Binder源码分析](chapter1/Binder源码分析.md)
  * [BottomSheets源码解析](chapter1/BottomSheets源码解析.md)
  * [CompoundButton源码分析](chapter1/CompoundButton源码分析.md)
  * [CoordinatorLayout源码分析](chapter1/CoordinatorLayout源码分析.md)
  * [FloatingActionButton源码解析](chapter1/FloatingActionButton源码解析.md)
  * [LruCache源码解析](chapter1/LruCache源码解析.md)
  * [Scroller源码解析](chapter1/Scroller源码解析.md)
  * [SearchView源码解析](chapter1/SearchView源码解析.md)
  * [SwipeRefreshLayout](chapter1/SwipeRefreshLayout.md)
  * [TabLayout源码解析](chapter1/TabLayout源码解析.md)
  * [TextView源码解析](chapter1/TextView源码解析.md)
  * [ViewDragHelper源码解析](chapter1/ViewDragHelper源码解析.md)

* [Android源码分析 第2期](chapter2\README.md)
  * [Bundle源码解析](chapter2/Bundle源码解析.md)
  * [Handler源码解析](chapter2/Handler源码解析.md)
  * [LayoutInflater源码解析](chapter2/LayoutInflater源码解析.md)
  * [LocalBroadcastManager源码解析](chapter2/LocalBroadcastManager源码解析.md)
  * [MediaPlayer源码分析](chapter2/MediaPlayer源码分析.md)
  * [NavigationView源码解析](chapter2/NavigationView源码解析.md)
  * [NestedScrolling事件机制源码解析](chapter2/NestedScrolling事件机制源码解析.md)
  * [NestedScrollView源码解析](chapter2/NestedScrollView源码解析.md)
  * [ScrollView源码解析](chapter2/ScrollView源码解析.md)
  * [Service源码解析](chapter2/Service源码解析.md)
  * [SharePreferences源码解析](chapter2/SharePreferences源码解析.md)
  * [SQLiteOpenHelper源码解析](chapter2/SQLiteOpenHelper源码解析.md)
  * [TextInputLayout源码解析](chapter2/TextInputLayout源码解析.md)

* [Android源码分析 第3期](chapter3\README.md)
  * [Android Lock Pattern 源码解析](chapter3/Android%20Lock%20Pattern%20源码解析.md)
  * [Android Universal Image Loader 源码分析](chapter3/Android%20Universal%20Image%20Loader%20源码分析.md)
  * [android-Ultra-Pull-To-Refresh 源码解析](chapter3/android-Ultra-Pull-To-Refresh%20源码解析.md)
  * [BaseAdapterHelper 源码分析](chapter3/BaseAdapterHelper%20源码分析.md)
  * [CalendarListView 源码解析](chapter3/CalendarListView%20源码解析.md)
  * [CircularFloatingActionMenu](chapter3/CircularFloatingActionMenu.md)
  * [Cling 源码解析](chapter3/Cling%20源码解析.md)
  * [Dagger 源码解析](chapter3/Dagger%20源码解析.md)
  * [DiscreteSeekBar 源码解析](chapter3/DiscreteSeekBar%20源码解析.md)
  * [DynamicLoadApk 源码解析](chapter3/DynamicLoadApk%20源码解析.md)
  * [EventBus 源码解析](chapter3/EventBus%20源码解析.md)
  * [HoloGraphLibrary 源码解析](chapter3/HoloGraphLibrary%20源码解析.md)
  * [PagerSlidingTabStrip 源码解析](chapter3/PagerSlidingTabStrip%20源码解析.md)
  * [PhotoView 源码解析](chapter3/PhotoView%20源码解析.md)
  * [Side Menu.Android 源码解析](chapter3/Side%20Menu.Android%20源码解析.md)
  * [SlidingMenu 源码解析](chapter3/SlidingMenu%20源码解析.md)
  * [ViewPagerindicator 源码解析](chapter3/ViewPagerindicator%20源码解析.md)
  * [Volley源码解析](chapter3/Volley源码解析.md)
  * [xUtils 源码解析](chapter3/xUtils%20源码解析.md)

* [Android源码设计模式分析 第4期](design_patterns/android_design_patterns_analysis.md)
  * [面向对象六大原则](design_patterns/oop-principles/oop-principles.md)
  * [Android设计模式源码解析之单例模式](design_patterns/singleton/readme.md)
  * [Android设计模式源码解析之适配器模式](design_patterns/adapter/readme.md)
  * [Android设计模式源码解析之桥接模式](design_patterns/bridge/readme.md)
  * [Android设计模式源码解析之Builder模式](design_patterns/builder/readme.md)
  * [Android设计模式源码解析之责任链模式](design_patterns/chain-of-responsibility/readme.md)
  * [Android设计模式源码解析之命令模式](design_patterns/command/readme.md)
  * [Android设计模式源码解析之外观模式](design_patterns/facade/readme.md)
  * [Android设计模式源码解析之迭代器模式](design_patterns/iterator/readme.md)
  * [Android设计模式源码解析之原型模式](design_patterns/prototype/readme.md)
  * [Android设计模式源码解析之代理模式](design_patterns/proxy/README.md)
  * [Android设计模式源码解析之策略模式](design_patterns/strategy/README.md)
  * [Android设计模式源码解析之模板方法模式](design_patterns/template-method/readme.md)

## 关注我

- Email：<815712739@qq.com>
- CSDN博客：[Allen Iverson](http://blog.csdn.net/axi295309066)
- 新浪微博：[AndroidDeveloper](http://weibo.com/u/1848214604?topnav=1&wvr=6&topsug=1&is_all=1)
- GitHub：[JackChan1999](https://github.com/JackChan1999)
- GitBook：[alleniverson](https://www.gitbook.com/@alleniverson)
- 个人博客：[JackChan](https://jackchan1999.github.io/)

|                   微信打赏                   |                  支付宝打赏                   |
| :--------------------------------------: | :--------------------------------------: |
| <img src="assets/weixin.png" width="300" /> | <img src="assets/支付宝.jpg" width="300" /> |