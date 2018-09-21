---
title: NStimer准确吗
date: 2017-12-14 13:15:48

description: NSTimer 不是一个基于真实时间的机制。NSTimer被激发需要满足三个条件。

categories: [问题记录]
tags: [Objective-C]
toc: false 
---

### 基本使用

```
 /*   
  * TimeInterval:设定执行时间    
  * target:目标    
  * @selector:方法(也就是目标(target)的行为(selector))  
  * userInfo:用于向selector方法中传参数,  一般是self
  * repeats:是否重复
  */
  
NSTimer * timer = [NSTimer scheduledTimerWithTimeInterval:.3 target:self selector:@selector(changeColor:) userInfo:view repeats:YES];    
[timer fire];//开始执行

//计时器执行的方法,sender 就是对应的计时器(那个计时器调的我)
- (void)changeColor:(NSTimer *)sender
{    
//sender计时器对象,通过.userinfo属性就能拿到当初传来的参数(id类型),
对于此题上面穿的是一个view对象,所以直接用UIview类型接收   
 UIView * vie = sender.userInfo;   
 //修改传入视图的背景色   
 vie.backgroundColor = [UIColor colorWithRed:arc4random() % 256 / 255.0 green:arc4random() % 256 / 255.0 blue:arc4random() % 256 / 255.0 alpha:1.0];
}

```

NSTimer 不是一个基于真实时间的机制。NSTimer被激发需要满足三个条件：

1. NSTimer被添加到特定mode的runloop中
2. 该mode型的runloop正在运行
3. 到达激发时间。

所以当上面1、2条出现问题时NSTimer就会受到影响，所以NSTimer是不准确的。

### 不准的原因

1. NSTimer加在main runloop中，模式是NSDefaultRunLoopMode，main负责所有主线程事件，例如UI界面的操作，复杂的运算，这样在同一个runloop中timer就会产生阻塞。
2. 模式的改变。主线程的 RunLoop 里有两个预置的 Mode：kCFRunLoopDefaultMode 和 UITrackingRunLoopMode。

* 这两个 Mode 都已经被标记为”Common”属性。DefaultMode 是 App 平时所处的状态，TrackingRunLoopMode 是追踪 ScrollView 滑动时的状态。当你创建一个 Timer 并加到 DefaultMode 时，Timer 会得到重复回调，但此时滑动一个TableView时，RunLoop 会将 mode 切换为 TrackingRunLoopMode，这时 Timer 就不会被回调，并且也不会影响到滑动操作。

### 解决NStimer不准的问题

1、在子线程中进行NSTimer的操作，再在主线程中修改UI界面显示操作结果

```
- (void)timerMethod2 {
    
    NSThread *thread = [[NSThread alloc] initWithTarget:self selector:@selector(newThread) object:nil];
    
    [thread start];
}

- (void)newThread{
    
    @autoreleasepool
    
    {
        
        [NSTimer scheduledTimerWithTimeInterval:1.0 target:self selector:@selector(showTime) userInfo:nil repeats:YES];
        
        [[NSRunLoop currentRunLoop] run];
    }
}

```

2、仍然在主线程中进行NSTimer操作，但是将NSTimer实例加到main runloop的特定mode（模式）中，避免被复杂运算操作或者UI界面刷新所干扰

```
[[NSRunLoop currentRunLoop] addTimer:self.timer forMode:NSRunLoopCommonModes];
```

[NSRunLoop currentRunLoop] 获取的就是 main runloop，使用NSRunLoopCommonModes模式，将NSTimer加入其中。


**注意：**

* 一开始的时候系统就为我们将主线程的main runloop隐式的启动了。

* 在创建线程的时候，可以主动获取当前线程的runloop。每个子线程对应一个runloop
