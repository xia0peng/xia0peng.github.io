---
title: RunLoop(四)--RunLoop 与多线程相关问题
date: 2018-08-18 08:01:18

description: 怎样实现一个常驻线程？

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

### RunLoop 与 多线程

* 线程是和 RunLoop 一一对应的

* 自己创建的线程默认是没有开启 RunLoop 的

### 怎样实现一个常驻线程

* 为当前线程开启一个 RunLoop 

* 向该 RunLoop 中添加一个 Port/Source 等维持 RunLoop 的事件循环

* 启动该 RunLoop


实现一个常驻线程的基本步骤：
1. 为当前线程开启一个 RunLoop ：NSRunLoop *runLoop = [NSRunLoop currentRunLoop];，因为获取当前 RunLoop 这个方法本身会查找如果当前线程没有 RunLoop 的话，会在系统的内部创建
2. 如果线程没有资源或者事件源要处理的话，默认情况下是不能维持事件循环的就会直接退出了，所以需要给他添加一个 Port/Source 来维持他的时间循环机制
3. 然后再调用 RunLoop 的 run 方法就可以实现一个常驻线程


代码实现：

![](/img/常驻线程代码实现1.png)
![](/img/常驻线程代码实现2.png)

**滑动与图片刷新：**
当 tableview 的 cell 上有需要从网络获取的图片的时候，滚动 tableView，异步线程会去加载图片，加载完成后主线程就会设置 cell 的图片，但是会造成卡顿。可以让设置图片的任务在 CFRunLoopDefaultMode 下进行，当滚动 tableView 的时候，RunLoop 是在 UITrackingRunLoopMode 下进行，不去设置图片，而是当停止的时候，再去设置图片。

```
- (void)viewDidLoad {
  [super viewDidLoad];
  // 只在NSDefaultRunLoopMode下执行(刷新图片)
  [self.myImageView performSelector:@selector(setImage:) withObject:[UIImage imageNamed:@""] afterDelay:ti inModes:@[NSDefaultRunLoopMode]];    
}
```