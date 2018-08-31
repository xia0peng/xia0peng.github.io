---
title: UI视图--离屏渲染
date: 2018-04-16 13:11:19
description: 离屏渲染
categories: UI视图
tags: [Objective-C]
---
***
[UI视图--图像显示原理与卡顿&掉帧](https://xiaopengmonsters.github.io/2018/04/05/UI%E8%A7%86%E5%9B%BE--%E5%9B%BE%E5%83%8F%E6%98%BE%E7%A4%BA%E5%8E%9F%E7%90%86%E4%B8%8E%E5%8D%A1%E9%A1%BF&%E6%8E%89%E5%B8%A7/)
[UI视图--UI绘制原理&异步绘制](https://xiaopengmonsters.github.io/2018/04/13/UI%E8%A7%86%E5%9B%BE--UI%E7%BB%98%E5%88%B6%E5%8E%9F%E7%90%86&%E5%BC%82%E6%AD%A5%E7%BB%98%E5%88%B6/)
[UI视图--离屏渲染](https://xiaopengmonsters.github.io/2018/04/16/UI%E8%A7%86%E5%9B%BE--%E7%A6%BB%E5%B1%8F%E6%B8%B2%E6%9F%93/)
***

#### 离屏渲染

* On-Screen Rendering ：意为当前屏幕渲染，指的是 GPU 的渲染操作是在当期用于显示的屏幕缓冲区进行的

* Off-Screen Rendering ：意为离屏渲染，指的是 GPU 在当前屏幕缓冲区以外新开辟一个缓存区进行渲染操作

#### 何时触发

* 圆角（当和 makeToBounds 一起使用时）

* 图层蒙版

* 阴影

* 光栅化

* 光栅化（Rasterization）是把顶点数据转换为片元的过程，具有将图转化为一个个栅格组成的图象的作用，特点是每个元素对应帧缓冲区中的一像素。（应用：较为广泛的应用于深度学习卷积神经网络的结构中）

#### 为何要避免离屏渲染

* 创建新的渲染缓冲区

* 上下文切换

离屏渲染是发生在 GPU 层面上的，由于离屏渲染使 GPU 层面上面触发了 openGL 的多通道渲染管线，产生了额外的开销

在触发离屏渲染的时候，会增加 GPU 的工作量，GPU 的工作量的增加很有可能导致 CPU 和 GPU 工作的总耗时超出了 16.7 毫秒，那么可能就会导致UI的卡顿和掉帧

离屏渲染会创建新的渲染缓冲区，导致内存上的开销，有多通道渲染管线，最终要把多通道的渲染结果进行合成，所有会有上下文的切换，就有 GPU 的额外开销，那么可能就会导致 UI 的卡顿和掉帧
