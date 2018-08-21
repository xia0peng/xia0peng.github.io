---
title: UI视图--UI绘制原理&异步绘制
date: 2018-04-13 15:22:09
description: UI绘制原理&异步绘制
categories: UI视图
tags: [Objective-C]
---


## UI绘制原理的过程

![](img/UI绘制原理的过程.png)

当调用 [UIView setNeedsDisplay] 方法时，系统会立即调用它所对应 layer 的同名方法 setNeedsDisplay，之后相当于是 layer 上打了一个脏标记，在当前 runloop 即将要结束的时候，会调用 [CALayer display] 方法，然后进入到视图真正的绘制过程当中

所以当调用 [UIView setNeedsDisplay] 时并没有立刻发生对应视图的绘制工作，视图的绘制时机，是在当前runloop即将结束的时候才会开始

[CALayer display] 方法内部实现中，首先会判断 layer 的 delegate 是否响应 dispLayer：，如果不响应，就会进入到系统的绘制流程中，如果响应，就为我们提供了异步绘制的入口

## 系统的绘制流程

![](img/系统的绘制流程.png)

在 CALayer 内部会创建一个 backing store （CGContextRef）,然后 layer 会判断它是否有代理，如果没有代理的话，会调用 [CALayer drawInContext:] ，如果有代理，会调用 [layer.delegate drawLayer:inContext:] 然后做当前视图的绘制工作，这部分是发生在系统内部的，然后在一个合适的时机给予我们一个回调方法 就是 [UIView drawRect:]， [UIView drawRect:] 的默认实现是什么都不做，给我们开这个口子就允许我们在系统绘制的基础之上做一些其他的相关的绘制工作，最后不论是哪个分支，都是由 CALyer 上传对应的 backing store （可以理解为位图）到GPU

## 异步绘制

[layer.delegate drawLayer:inContext:] 方法实现就可以进入到异步绘制的流程

* 代理负责生产对应的 bitmap

* 设置该 bitmap 作为 layer.contents 属性的值 

#### 异步绘制的机制和流程

![](img/异步绘制的机制和流程.png)

在调用 setNeedsDisplay 方法之后，在当前 runloop 快要结束的时候，由系统调用视图所对应 CALayer 的 display 方法，然后如果代理实现了 displayLayer：函数时，会调用代理的 displayLayer：函数方法，然后会通过子线程的切换，在子线程中做位图的绘制 ，此时主线程可以做别的事


在全局并发队列子线程中：

1. 第一步通过 CGBitmapContextCreat() 函数来创建一个位图的上下文
2. 然后通过CoreGraphic 的相关API做当前UI控件的绘制工作 
3. 之后，再通过 CGBitmapContextCreatImage() 函数来根据当前所绘制的上下文，生成一张 CGImage 图片
4. 然后回到主队列中提交位图，设置给 CALayer 的 contents 属性 ，这样就完成了一个UI控件的异步绘制