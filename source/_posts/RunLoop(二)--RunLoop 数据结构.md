---
title: RunLoop(二)--RunLoop 数据结构
date: 2018-08-13 01:01:18

description: RunLoop 数据结构

categories: RunLoop
tags: [Objective-C]
---

*******
[RunLoop(一)--RunLoop 本质和事件循环机制](https://xiaopengmonsters.github.io/2018/07/16/Block--Block%20%E6%9C%AC%E8%B4%A8%E5%92%8C%E6%88%AA%E8%8E%B7%E5%8F%98%E9%87%8F/)
[RunLoop(二)--RunLoop 数据结构](https://xiaopengmonsters.github.io/2018/07/16/Block--Block%20%E6%9C%AC%E8%B4%A8%E5%92%8C%E6%88%AA%E8%8E%B7%E5%8F%98%E9%87%8F/)
[RunLoop(三)--RunLoop 与 NSTimer 相关问题](https://xiaopengmonsters.github.io/2018/07/16/Block--Block%20%E6%9C%AC%E8%B4%A8%E5%92%8C%E6%88%AA%E8%8E%B7%E5%8F%98%E9%87%8F/)
[RunLoop(四)--RunLoop 与多线程相关问题](https://xiaopengmonsters.github.io/2018/07/16/Block--Block%20%E6%9C%AC%E8%B4%A8%E5%92%8C%E6%88%AA%E8%8E%B7%E5%8F%98%E9%87%8F/)
[RunLoop[转]](https://xiaopengmonsters.github.io/2017/04/20/RunLoop/)
******

### Block 的类型

impl.isa 就是标识当前 block 是哪种类型的

![](/img/Block的类型.png)

