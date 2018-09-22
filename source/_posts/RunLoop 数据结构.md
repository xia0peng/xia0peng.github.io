---
title: RunLoop(二)--RunLoop 数据结构
date: 2018-08-13 01:01:18

description: 数据结构才能了解到 RunLoop 的实质

categories: RunLoop
tags: [Objective-C]
---

*******
[RunLoop(一)--RunLoop 本质和事件循环机制](https://xiaopengmonsters.github.io/2018/08/10/RunLoop%20%E6%9C%AC%E8%B4%A8%E5%92%8C%E4%BA%8B%E4%BB%B6%E5%BE%AA%E7%8E%AF%E6%9C%BA%E5%88%B6)
[RunLoop(二)--RunLoop 数据结构](https://xiaopengmonsters.github.io/2018/08/13/RunLoop%20%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84/)
[RunLoop(三)--RunLoop 与 NSTimer 相关问题](https://xiaopengmonsters.github.io/2018/08/18/RunLoop%20%E4%B8%8E%20NSTimer%20%E7%9B%B8%E5%85%B3%E9%97%AE%E9%A2%98/)
[RunLoop(四)--RunLoop 与多线程相关问题](https://xiaopengmonsters.github.io/2018/08/18/RunLoop%20%E4%B8%8E%E5%A4%9A%E7%BA%BF%E7%A8%8B%E7%9B%B8%E5%85%B3%E9%97%AE%E9%A2%98/)
[RunLoop[转]](https://xiaopengmonsters.github.io/2017/04/20/RunLoop/)
******

在 OC 中实际提供了两个 RunLoop 的
一个是 NSRunLoop 
一个是 CFRunLoop
NSRunLoop 是对 CFRunLoop 的封装，提供了一些面向对象的 API
NSRunLoop 是位于 Foundation 当中的，CFRunLoop 位于 CoreFoundation 当中的

RunLoop 的数据结构主要有三个
* CFRunLoop
* CFRunLoopMode
* Source/Timer/Observer

### CFRunLoop

![](/img/CFRunLoop.png)

```
struct __CFRunLoop {
    CFMutableSetRef _commonModes;     // Set
    CFMutableSetRef _commonModeItems; // Set
    CFRunLoopModeRef _currentMode;    // Current Runloop Mode
    CFMutableSetRef _modes;           // Set
    ...
};
```

### CFRunLoopMode

![](/img/CFRunLoopMode.png)

```
struct __CFRunLoopMode {
    CFStringRef _name;            // Mode Name, 例如 @"kCFRunLoopDefaultMode"
    CFMutableSetRef _sources0;    // Set
    CFMutableSetRef _sources1;    // Set
    CFMutableArrayRef _observers; // Array
    CFMutableArrayRef _timers;    // Array
    ...
};

```

### CFRunLoopSource

![](/img/CFRunLoopSource.png)

在 CF 框架当中官方名称叫 CFRunLoopSource ，有两种 source0 和 source1
唤醒线程就是从内核态切换到用户态

### CFRunLoopTimer

![](/img/CFRunLoopTimer.png)

和平时所使用的 NSTimer 是具备免费桥转换的

### CFRunLoopObserver

观测时间点

* KCFRunLoopEntry
* KCFRunLoopBeforeTimers
* KCFRunLoopBeforeSources
* KCFRunLoopBeforeWaiting
* KCFRunLoopAfterWaiting
* KCFRunLoopExit

可以通过注册一些 Observer 来实现对 RunLoop 的一些相关时间点的监测或者观察

问：我们可以监听 RunLoop 哪些时间点？
* RunLoop 的入口事迹，当 RunLoop 准备启动的时候系统会给我们一个回调通知，这个通知掉 CFRunLoopEntry
* 代表的含义：通知观察者 RunLoop 将要对 Timer 一些相关事件进行处理了
* 代表将要处理一些 Source 事件
* 通知对应观察者，当前 RunLoop 将要进入休眠状态，这个通知或者说观测点是非常重要的一个观测点，在 RunLoop 发送这个通知的时候，即将要发生用户态到内核态的切换
* 这也是一个重要的观测点，这个通知发出的时机恰好是从内核态切换到用户态之后的不久之间
* 代表 RunLoop 退出的通知

### 各个数据之间的关系

![](/img/各个数据之间的关系.png)

### RunLoop 的 Mode

![](/img/RunLoop的Mode.png)

一个 RunLoop 可以对应多个 mode，每个 mode 当中又可以有多个 Sources1，Timers，Observers
当 RunLoop 运行在某一个 mode 上的时候，比如说运行在 mode1 上面，这个时候如果 mode2 当中某一个 timer 事件或者 Observe 事件回调了，这个时候是没有办法接收对应 mode2 当中所回调过来的事件，这就 RunLoop 有多个 mode 的原因，实际上起到的就是一个屏蔽的效果，当运行到 mode1 上时，只能接收处理 mode1 当中的 Sources1，Timers，Observers

思考：一个 Timer 要想同时加入到两个 mode 里面，需要怎么做？如果说这个 Timer 既想要它在 mode1 上可以正常运行，在每一个事件回调中做正常的处理，在 mode2 上也需要做相应的处理和事件的回调接收

系统提供了添加到两个 mode 的机制的，这样可以保证当 RunLoop 发生 mode 切换的时候也可以让对应的 Timer 等事件正常处理接收

**这实际上就涉及到了 CommonMode（CommonMode 的特殊性）：**

![](/img/CommonMode的特殊性.png)

CommonMode 本身是有它的特殊性的，CommonMode 并不是一个实际存在的模式，在 OC 当中经常会通过 NSRunLoopCommonModes 字符串常量来表达 CommonMode
CommonMode 本身和 defaultMode 是有区分的，调用一个线程运行在 CommonMode 上面和运行在具体的 defaultMode 上是有区别的


这里有个概念叫 “CommonModes”：一个 Mode 可以将自己标记为”Common”属性（通过将其 ModeName 添加到 RunLoop 的 “commonModes” 中）。每当 RunLoop 的内容发生变化时，RunLoop 都会自动将 _commonModeItems 里的 Source/Observer/Timer 同步到具有 “Common” 标记的所有Mode里。
应用场景举例：主线程的 RunLoop 里有两个预置的 Mode：kCFRunLoopDefaultMode 和 UITrackingRunLoopMode。这两个 Mode 都已经被标记为”Common”属性。DefaultMode 是 App 平时所处的状态，TrackingRunLoopMode 是追踪 ScrollView 滑动时的状态。当你创建一个 Timer 并加到 DefaultMode 时，Timer 会得到重复回调，但此时滑动一个TableView时，RunLoop 会将 mode 切换为 TrackingRunLoopMode，这时 Timer 就不会被回调，并且也不会影响到滑动操作。
有时你需要一个 Timer，在两个 Mode 中都能得到回调，一种办法就是将这个 Timer 分别加入这两个 Mode。还有一种方式，就是将 Timer 加入到顶层的 RunLoop 的 “commonModeItems” 中。”commonModeItems” 被 RunLoop 自动更新到所有具有”Common”属性的 Mode 里去。