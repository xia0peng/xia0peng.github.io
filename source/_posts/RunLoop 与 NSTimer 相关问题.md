---
title: RunLoop(三)--RunLoop 与 NSTimer 相关问题
date: 2017-12-20 01:01:18

description: 滑动 TableView 的时候我们的定时器还会生效吗？

categories: RunLoop
tags: [Objective-C]
---

*******
[RunLoop(一)--RunLoop 本质和事件循环机制](https://xiaopengmonsters.github.io/2018/08/10/RunLoop%20%E6%9C%AC%E8%B4%A8%E5%92%8C%E4%BA%8B%E4%BB%B6%E5%BE%AA%E7%8E%AF%E6%9C%BA%E5%88%B6)
[RunLoop(二)--RunLoop 数据结构](https://xiaopengmonsters.github.io/2018/08/13/RunLoop%20%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84/)
[RunLoop(三)--RunLoop 与 NSTimer 相关问题](https://xiaopengmonsters.github.io/2018/08/18/RunLoop%20%E4%B8%8E%20NSTimer%20%E7%9B%B8%E5%85%B3%E9%97%AE%E9%A2%98/)
[RunLoop(四)--RunLoop 与多线程相关问题](https://xiaopengmonsters.github.io/2018/08/18/RunLoop%20%E4%B8%8E%E5%A4%9A%E7%BA%BF%E7%A8%8B%E7%9B%B8%E5%85%B3%E9%97%AE%E9%A2%98/)
[RunLoop[转]](https://xiaopengmonsters.github.io/2017/04/20/RunLoop/)
[NStimer准确吗](https://xiaopengmonsters.github.io/2017/12/14/NStimer%E5%87%86%E7%A1%AE%E5%90%97/)
******


以前写过一遍关于 [NSTimer 准确性的文章](https://xiaopengmonsters.github.io/2017/12/14/NStimer%E5%87%86%E7%A1%AE%E5%90%97/)，可以看看哈

滑动 TableView 的时候我们的定时器还会生效吗？

![](/img/定时器还会生效吗.png)

我们的这个线程或者说当前 TableView 他正常情况下在 default 模式下，而当对 TableView 进行滑动的时候会发生 mode 的切换，会切换到 UITrackingRunLoopMode 上面

前面讲过当把一个 Timer 或者 Sources，Observe 添加到某一个 mode 上之后，如果当前 RunLoop 是运行在另一个 mode 上，对应的这些 Timer 或者 Sources，Observe 是没有办法进行后续的处理和回调的，mode 有多个，解决的问题是 Timer，Sources 以及 Observe 的隔离，这种隔离也产生了一个问题

就是：我们创建的 Timer 默认情况下是添加到当前 RunLoop 的 default 模式下，这个时候如果切换到 TrackingRunLoopMode 上面，定时器就不会在生效了

![](/img/CFRunLoopAddTimer.png)

解决方案：可以通过 CFRunLoopAddTimer 函数把 NSTimer 添加到当前 RunLoop 的 commonMode 当中，前面讲过 commonMode 不是实际的 mode，它只是把一些 mode 打上一个 common 的标记，然后可以把某一个事件源比如说 timer 同步到多个 mode 当中

**CFRunLoopAddTimer 内部实现：**

![](/img/CFRunLoopAddTimer内部实现1.png)
![](/img/CFRunLoopAddTimer内部实现2.png)

有三个参数第一个是 RunLoop，第二个是 timer，第三个是 mode 的名称
内部有一个判断：
如果当前的 mode 是 commonMode 的话，首先会提取 RunLoop 的 commonModes ，然后会判断 RunLoop 对应的  commonModeItems 是否为空，如果为空的话会重新创建集合，然后把 timer 添加到 commonModeItems 集合中，然后把 RunLoop 和 timer 封装成一个 context ，之后对这个集合当中的每一个元素都调用 CFRunLoopAddItemToCommonModes 函数

**CFRunLoopAddItemToCommonModes 的实现**

![](/img/CFRunLoopAddItemToCommonModes的实现.png)

在这个函数内部首先会取 当前 mode 的名称，然后再把对应的 RunLoop 和 item 取出来，接下来会根据当前 item 类型的判断来觉得调用 CFRunLoopAddSource 还是 CFRunLoopAddObserve 以及 CFRunLoopAddTimer

循坏调用？实际上不是的
因为这个时候传进来的这个 modeName 已经从 commonMode 变成了被打上 commonMode 标记的具体的实际的一个 mode 

然后在 CFRunLoopAddTimer 当中如果传进来的是一个具体的 mode ，那么在下面的那一段代码逻辑中就会把 timer 最终添加到对应的 RunLoopMode 下面的 timer 数组当中，这样的话就可以实现通过把多个 mode 当中都添加同一个 timer

![](/img/CFRunLoopAddItemToCommonModes的实现2.png)

因为调用 CFSetApplyFunction 这个函数 就是对于这个集合当中每一个元素都调用 CFRunLoopAddItemToCommonModes 函数，就可以实现把多个 timer 同步到多个 mode 下面 ，所以说可以通过 add timer 到一个 commonMode 上，可以实现把一个 timer 添加到多个 mode 上

把一个 timer 添加到多个 mode 上面之后，UITrackingRunLoopMode 实际上也是被打上 common 标记的，就把这个 timer 同步到了 TrackingRunLoopMode 上面，这个时候列表继续滑动的时候，timer 也是能正常响应到时间的时间回调的