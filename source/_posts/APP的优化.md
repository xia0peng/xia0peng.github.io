---
title: APP的优化
date: 2018-1-8 15:21:18

description: 一款APP开发出来以后，并不能单单关注的它的功能，重中之重是其性能，只有性能优越，用户体验才能提高，这样才能留住用户，让用户觉得我们是在用心最开发✧(≖ ◡ ≖✿)。

categories: [性能优化]
tags: [Objective-C]
toc: fales 
---

***
[UITableView的优化](https://xiaopengmonsters.github.io/2017/12/26/UITableView%E7%9A%84%E4%BC%98%E5%8C%96/)
[UITableView的优化进阶篇](https://xiaopengmonsters.github.io/2018/02/16/UITableView%E7%9A%84%E4%BC%98%E5%8C%96%E8%BF%9B%E9%98%B6%E7%AF%87/)
[APP的优化](https://xiaopengmonsters.github.io/2018/01/08/APP%E7%9A%84%E4%BC%98%E5%8C%96/)
***

一款APP开发出来以后，并不能单单关注的它的功能，重中之重是其性能，只有性能优越，用户体验才能提高，这样才能留住用户，让用户觉得我们是在用心最开发✧(≖ ◡ ≖✿)。

# 1、启动优化

[iOS App 启动性能优化](https://mp.weixin.qq.com/s/Kf3EbDIUuf0aWVT-UCEmbA)这篇文章很详细的介绍了启动的性能优化，其中还介绍了一些与启动相关的知识，值得一看。

在这里只对启动优化做一个概括，大概分为以下几种：

1. 移除不需要用到的动态库
2. 移除不需要用到的类
3. 合并功能类似的类和扩展（Category）
4. 压缩资源图片
5. 优化applicationWillFinishLaunching
6. 优化rootViewController加载

总结：

* 利用DYLD_PRINT_STATISTICS分析main()函数之前的耗时

   1. 重新梳理架构，减少动态库、ObjC类的数目，减少Category的数目
   2. 定期扫描不再使用的动态库、类、函数，例如每两个迭代一次
   3. 用dispatchonce()代替所有的__attribute__((constructor))函数、C++静态对象初始化、ObjC的+load 
   4. 在设计师可接受的范围内压缩图片的大小，会有意外收获


* 利用锚点分析applicationWillFinishLaunching的耗时

   1. 将不需要马上在applicationWillFinishLaunching执行的代码延后执行
   2. rootViewController的加载，适当将某一级的childViewController或subviews延后加载
   3. 如果你的App可能会被后台拉起并冷启动，可考虑不加载rootViewController

   
* 不应放过的一些小细节

   1. 异步操作并不影响指标，但有可能影响交互体验，例如大量网络请求导致数据拥堵
   2. 有时候一些交互上的优化比技术手段效果更明显，视觉上的快决不是冰冷的数据可以解释的，好好和你们的设计师谈谈动画


计算代码执行耗时（用Instruments也可以）：

```
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    
    CFAbsoluteTime start = CFAbsoluteTimeGetCurrent();
    
    // do something
    
    CFAbsoluteTime end = CFAbsoluteTimeGetCurrent();
    
    NSLog(@"%fms",(end -start)*1000);
     
}

```

# 2、UITableView的优化

UITableView是APP最重要的控件之一，几乎所有APP都会用到UITableView，因此它的优化也为APP优化重中之重，是优化APP必不可少的以部门。前面写过一篇专门讲解[UITableView优化](https://xiaopengmonsters.github.io/2017/12/26/UITableView%E7%9A%84%E4%BC%98%E5%8C%96/)的文章，在这里就不具体描述。

# 3、未完待续。。。
