---
layout: post
title: Android 开发进阶之清除内存泄漏
categories: [开发，Android]
---

# Android 内存管理机制

Android 的内存管理机制可以简单概括为：系统没有为内存提供交换区，它使用 [paging](http://en.wikipedia.org/wiki/Paging) 与 [memory-mapping(mmapping)](http://en.wikipedia.org/wiki/Memory-mapped_files) 来管理内存。

对开发来说，上面的管理机制意味着：
1. 彻底释放内存资源的**_唯一方法**_是释放对象的引用，使对象可以被 GC(garbage collector) 回收。
2. 有一种例外情况：没有任何修改的文件，比如代码本身，映射进内存后，如果系统需要使用这部分内存，会将这部分内存页移出。
# 什么是内存泄漏

上面第 2 点在开发应用时，并没有实际意义。因此在开发应用时，正确使用内存先要保证释放掉不需要的内存资源。如果对象不需要了，但是由于没有释放对它的引用， GC 无法回收相应的内存资源，这部分内存就无法被利用了。这种情况就是所谓的“内存泄漏”。

内存泄漏是资源泄漏的一种，是由于没有正确管理内存分配而造成内存不再使用却没有得到释放。

![Memory Leaks](http://upload-images.jianshu.io/upload_images/4498-8a1c2d2d32b791b1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

内存泄漏就是对内存资源的浪费，内存通常是珍稀资源。所以，内存泄漏的影响很坏！

如果应用存在内存泄漏，对用户来说，应用会越用越慢，并且会出现闪退；对开发者而言，会收到很多应用不稳定的评价，大量内存溢出（ OOM ）的错误日志，紧接着就是产品，测试，领导甚至老板的围攻。

![苦逼的“程序员”](http://upload-images.jianshu.io/upload_images/4498-338c976bbd027a17.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
# 如何清除内存泄漏
## 排查泄漏
### 症状

前面提到，如果存在内存泄漏，并且每次泄漏的内存很多，则应用在使用过程中会时不时出现闪退的现象。如果查看日志数据，会看到`OutOfMemoryError`类型的错误：

![OutOfMemoryError](http://upload-images.jianshu.io/upload_images/4498-7c8b08af5a545b58.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如果每次溢出的内存不多，则应用偶尔会出现闪退的现象，甚至平常不会出现闪退现象。但统计系统也会存在一些 `OutOfMemoryError` 类型的错误。

这里需要提醒的是：`OutOfMemoryError` 错误打印的栈信息中出错的位置很有可能不是问题的原因。因为由于泄漏导致内存不够时，任何位置都可能引起 `OutOfMemoryError` 错误。所以不要过分关注引起 `OutOfMemoryError` 的位置。
### 确诊
#### 思路

试着找到导致泄漏的操作路径，拼命重复这个操作路径！

这里需要提醒的是：
1. 不是任意一台设备都可以复现所有泄漏，使用同款设备尝试。
2. 要记录测试数据供后续分析：测试前记录下应用所占用的内存大小 `m0` ，重复多次后再记录下应用所占用的内存大小 `m1` 以及重复次数 `n` ；出现 OOM 时，或者将要出现 OOM 时抓取应用的 heap dump 数据（ .hprof 文件）。
3. 如果每次泄漏的内存很少，重复次数 `n` 就需要很大，此时可以借助 `monkey` 测试脚本来完成。

如果上面的 `m1` 明显大于 `m0` 或者直接出现 OOM 错误，则应用一定存在内存泄漏。
## 定位泄漏

确认存在内存泄漏后，接下来就要定位哪些对象被泄漏了。目前比较好用的是 [Memory Analyzer (MAT)](http://eclipse.org/mat/) 这个工具。MAT 是一个 `Java heap analyzer` ，用来查找内存泄漏与优化内存。
### 相关概念
#### Heap Dump

在一个时间点，给一个 Java 进程的内存使用情况拍个照，就是一份 Heap Dump 数据。通常 heap dump 包含了快照触发时， Java 虚拟机堆 java 对象和类的相关信息，如：
- All Objects
  Class, fields, primitive values and references
- All Classes
  Classloader, name, super class, static fields
- GC Roots
  Objects defined to be reachable by the JVM
- Thread Stacks and Local Variables
  The call-stacks of threads at the moment of the snapshot, and per-frame information about local objects

需要注意的是：heap dump 数据并不包含对象分配信息，所以无法从中获知谁创建了对象，在哪里创建的对象。
#### Shallow vs. Retained Heap

**Shallow heap** 是一个对象实际占用的内存大小。
**Retained set of X** 指的是这样的对象集合： X 对象被 GC 回收后，所有能被回收的对象集合。
**Retained heap of X** 指的是 retained set 中所有对象 shallow heap 的总和。

换一种说法： shallow heap 是一个对象在堆中占用的大小，retained heap 是对象被 GC 回收后，能释放的堆大小。
#### Dominator Tree

**dominator tree** 是 MAT 提供的一种对象图。将对象的引用关系图转成 dominator tree 可以使我们容易看清堆中内存的分布以及相关依赖。

下面是一些定义：
- 对象 x **dominates** 对象 y 则在对象图中每一条从起点（或者根节点）到对象 y 的路径必须经过对象 x 。
- 对象 y 的 **immediate dominator** x 是距离 y 最近的那个 dominator 。
- **dominator tree** 基于对象图构建。在 dominator tree 中，每一个对象都是其子对象的 immediate dominator 。因此，对象与对象之间的依赖关系很容易被识别。

dominator tree 有以下几点重要特征：
- 对象 x 的子树中的对象集合就是 x 的 retained set 。
- 如果对象 x 是 对象 y 的 immediate dominator ，则 x 的 immediate dominator 也 dominates y ，以此类推。
- The edges in the dominator tree do not directly correspond to object references from the object graph.

根据上面的概念，下图左边的 object graph 可以转换为右边的 dominator tree ：

![object graph to dominator tree](http://upload-images.jianshu.io/upload_images/4498-fc521d8105f9d4dd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
#### Garbage Collection Roots

GC root 是 heap 外面那个可以访问的对象。下面是可能的 GC root ：
- System Class
  Class loaded by bootstrap/system class loader. For example, everything from the rt.jar like java.util.* .
- JNI Local
  Local variable in native code, such as user defined JNI code or JVM internal code.
- JNI Global
  Global variable in native code, such as user defined JNI code or JVM internal code.
- Thread Block
  Object referred to from a currently active thread block.
- Thread
  A started, but not stopped, thread.
- Busy Monitor
  Everything that has called wait() or notify() or that is synchronized. For example, by calling synchronized(Object) or by entering a synchronized method. Static method means class, non-static method means object.
- Java Local
  Local variable. For example, input parameters or locally created objects of methods that are still in the stack of a thread.
- Native Stack
  In or out parameters in native code, such as user defined JNI code or JVM internal code. This is often the case as many methods have native parts and the objects handled as method parameters become GC roots. For example, parameters used for file/network I/O methods or reflection.
- Finalizable
  An object which is in a queue awaiting its finalizer to be run.
- Unfinalized
  An object which has a finalize method, but has not been finalized and is not yet on the finalizer queue.
- Unreachable
  An object which is unreachable from any other root, but has been marked as a root by MAT to retain objects which otherwise would not be included in the analysis.
- Java Stack Frame
  A Java stack frame, holding local variables. Only generated when the dump is parsed with the preference set to treat Java stack frames as objects.
- Unknown
  An object of unknown root type. Some dumps, such as IBM Portable Heap Dump files, do not have root information. For these dumps the MAT parser marks objects which are have no inbound references or are unreachable from any other root as roots of this type. This ensures that MAT retains all the objects in the dump.
### 寻找被泄漏对象（病灶）

MAT 的功能很多很强大，用来分析内存泄漏的话，主要使用 `Dominator Tree` 与 `Histogram` 这两个功能。
#### MAT 相关功能简介

使用 MAT 打开前面拿到的 hprof 文件：

![MAT_Overview.png](http://upload-images.jianshu.io/upload_images/4498-7ea9d3cb247bb19f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

首先看到的是 `Overview` 页面。
里面 `Details` 部分显示了堆的一些基本信息，以及 `Unreachable Objects Histogram` 入口，其中列出了堆中所有 **Unreferenced** 对象。

在 `Actions` 部分，有`Histogram` 和 `Dominator Tree` 的入口，前者更关注堆中对象的个数，后者更关注堆中对象的类型。其中列出的对象都是 **Referenced** 对象。被泄漏的对象一定是从里面找。

寻找被泄漏对象，可以从两个方向下手：
#### 方式一、从对象个数入手

如果前面的重复次数 `n` 已知的话，可以先从对象个数入手。重复一次泄漏路径，就会泄漏一次对象，所以重复 `n` 次，泄漏的对象个数应该为 `n` 个。

打开 Histogram ：
![Histogram](http://upload-images.jianshu.io/upload_images/4498-43d5755f1fddc786.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

`Histogram` 页面是一张表，表里的每一行是一个 java 类。第一列是类名，第二列是该类实例的个数，第三列是该类所有实例的 `shallow heap` ，第四列是该类所有实例的 `retained heap` 。

表的第一行可以输入相应字段的条件过滤要显示的结果，如排查应用层的泄漏，可以通过提供类名的关键词过滤，使之只显示相关类的信息。

前面重复次数 `n` 为 9 。排查对象个数为 9 附近的类，
![Objects leaked shown in histogram](http://upload-images.jianshu.io/upload_images/4498-ae5ba92c50e97917.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

不难发现 `HomeTabActivity` 这个类依然在 heap 中（App 此时已经不在前台，且已经强制 GC）。因此，可以确认 `HomeTabActivity` 对象被泄漏了。
#### 方式二、从对象类型开始

如果重复次数 `n` 不确定，则可以从 `Dominator Tree` 开始查。通过 `Dominator Tree` ，我们可以很方便的看到有哪些无法被 GC 回收的内存块儿，以及对应内存块儿的 GC root 。因此，我们可以通过排查并确认内存块儿以及相应 GC root 是否合理来判断此内存块儿中的对象是否是被泄漏的对象。

打开 `Dominator Tree` ：

![Dominator Tree](http://upload-images.jianshu.io/upload_images/4498-a214a05af9f05d81.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

`Dominator Tree` 页面也是一张表，表里的每一行是一个对象，第一列显示了该对象的类名以及内存地址等，第二列显示了该对象的 `shallow heap` ，第三列显示了该对象的 `retained heap` ，第四列显示了该对象的占比。

与 `Histogram` 类似，可以通过过滤缩小排查范围，基于前面的分析，这次我们用更小的范围排查。

需要注意，在 `Class Name` 这一列中，靠左边一排图标中，有些图标左下角有小圆点，有些没有。带小圆点的对象就是前面提到的 GC root 。最右边的字段，如： `System Class` 是 GC root 的类型。GC root 本身不会是泄漏的对象。

只需要排查不是 GC root 的那些对象。不难发现 heap 中存在 9 个 `HomeTabActivity` 类型的对象，与当时应用已经不在前台的事实有出入，所以，这 9 个对象不应该存在，是被泄漏的，同时与之前重复次数 `n` 一致。

![MAT_dt_leaked.png](http://upload-images.jianshu.io/upload_images/4498-56a6b1af92411451.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 修复泄漏（治病）

找到被泄漏的对象后，接着要算出从该对象到 GC roots 的最短强引用路径，找到本不该存在的路径，对照相应源码，修复掉错误的代码逻辑，也就剔除了这个内存泄漏。
### 找病根

在 MAT 中如何看到一个对象到 GC root 的最短强引用路径呢？
- 在 `Histogram` 中查看

![Show shortest paths to GC roots exclude weak references](http://upload-images.jianshu.io/upload_images/4498-6b8306d1195935f8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在被泄漏类上面，点击右键菜单中的 `Merge Shartest Paths to GC Roots` --> `exclude weak references` 。就会看到这个 java 类中所有无法被释放的对象的 GC roots ，点开每条路径，可以看到引用关系。

![Paths to GC roots with detail infomation](http://upload-images.jianshu.io/upload_images/4498-053a07de7b9488ec.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

从上图，可以看到被泄漏的 `HomeTabActivity` 对象都是同一个 GC root 。
- 在 `Dominator Tree` 中查看

在被泄漏对象上面，通过右键菜单，选择 `Path to GC Roots` --> `exclude weak references` 可以看到该对象到 GC root 的一条路径。

![MAT_dominator_tree_root.png](http://upload-images.jianshu.io/upload_images/4498-08b9170faa811eec.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

与通过 `Dominator Tree` 找到的路径一致。

![Paths to GC roots with detail infomation](http://upload-images.jianshu.io/upload_images/4498-6e6b529bef3e0110.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

前面说这条路径是不应该存在的，但是是什么原因导致其出现呢？接下来我们分析泄漏原因。

![MAT_dt_gc_root.png](http://upload-images.jianshu.io/upload_images/4498-85f8cf9d7e9bff7d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们看到位于地址 `0x4068aca8` 的一个 `HomeTabActivity` 对象被位于地址 `0x409cbbc8` 的一个 `Toast` 对象通过成员变量 `mContext` 引用。接着，`mContext` 又被 `Toast` 中的内部类 `TN` 对象所引用，这个对象又是一个 `Native Stack` 类型的 GC root 。

根据上面的引用路径，结合应用相关源码：

`HomeTabActivity` 源代码（部分）；

```
/* HomeTabActivity.java */
public class HomeTabActivity extentds ... {
  ...
  @Override
  public void onBackPressed() {
    ...
    if ((currentTime - touchTime) >= waitTime) {
      Toast.makeText(this, "再按一次退出应用", Toast.LENGTH_SHORT).show();
    } else {
      ...
    }
    ...
  }
  ...
}
```

发现在连按两次返回键退出应用的功能代码中，将 `HomeTabActivity` 对象的引用传入 `Toast.makeText()` 。

因此泄漏的原因是：
`HomeTabActivity` 对象被生命周期更长的 `Toast$TN` 对象所引用，导致其实际生命周期超出了所预期的生命周期。

其实，站在 coder 的角度，内存泄漏本质就是**该死不死**，不论是什么具体形式导致了这种局面。
### 处方

本例中的泄漏是由于使用了不恰当的 `Context` 对象所致。

Android 中存在 **Application Context** 与 **Activity Context** 两种具体的 `Context` 实例。前者的生命周期与应用进程的生命周期一样，比后者长。

因此，在使用 Toast 时，应该使用 **Application Context** ，就不会出现**该死不死**的对象，也就不存在内存泄漏。修复代码如下：

```
public class HomeTabActivity extentds ... {
  ...
  @Override
  public void onBackPressed() {
    ...
    if ((currentTime - touchTime) >= waitTime) {
      Toast.makeText(getApplicationContext(), "再按一次退出应用", Toast.LENGTH_SHORT).show();
    } else {
      ...
    }
    ...
  }
  ...
}
```
# 附录：
## Android 常见内存泄漏形式
- Activity 泄漏 - 内部类

![内部类](http://upload-images.jianshu.io/upload_images/4498-49a86f195a4a943d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- Activity 泄漏 - 容器对象泄漏

![容器对象泄漏](http://upload-images.jianshu.io/upload_images/4498-349cfd4b9654849f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- Activity 泄漏 - Static, Singleton

![Static, Singleton](http://upload-images.jianshu.io/upload_images/4498-959a9e9fad44f481.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- 谨慎选择合适的 Context

![合适的 Context](http://upload-images.jianshu.io/upload_images/4498-6b5002b829fe5b40.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- 注意有生命周期对象的注销

![有生命周期对象的注销](http://upload-images.jianshu.io/upload_images/4498-4b2fffd90bab8eb6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- 注意大胖子（Bitmap, WebView, Cursor）的及时回收

![Bitmap](http://upload-images.jianshu.io/upload_images/4498-bb1f522c758a54c4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![WebView](http://upload-images.jianshu.io/upload_images/4498-f52c5526dbeee268.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![Cursor](http://upload-images.jianshu.io/upload_images/4498-31dfe16c930c743f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 参考文献
- [Managing Your App's Memory](http://developer.android.com/training/articles/memory.html)
- [Android性能优化之内存篇](http://hukai.me/android-performance-memory/)
- [Eclipse Memory Analyzer - help](http://help.eclipse.org/mars/index.jsp?topic=/org.eclipse.mat.ui.help/welcome.html)

