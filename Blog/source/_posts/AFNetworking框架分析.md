---
title: AFNetworking框架分析
date: 2017-3-16 13:23:28

description: 想必AFNetworking网络请求框架，每个iOS开发攻城狮都晓得，也会使用，但是其内部构造及原理对小白来说还是有些生疏，这篇文章我们就来探索AFNetworking的神秘。
categories: 原理
tags: [Objective-C]
toc: false 
---

想必AFNetworking网络请求框架，每个iOS开发攻城狮都晓得，也会使用，但是其内部构造及原理对小白来说还是有些生疏，这篇文章我们就来探索AFNetworking的神秘。

说明：本篇文章并非原创，文章底部有原文链接，之所以整理，是因为这是学习的过程，也是整理零散知识的过程，每一遍的来过都会有意外收获。

## AFN的结构

首先我们来看看AFN的结构：

![](/img/AFN结构图.png)

从上图可以看出，除了头文件和Support Files，AFNetworking是由NSURLSession、Reachability、Security、Serialization、UIKit五部分组成。

1. **NSURLSession**：网络通信模块（核心模块）对应AFNetworking中的 AFURLSessionManager和对HTTP协议进行特化处理的AFHTTPSessionManager，AFHTTPSessionManager是继承于AFURLSessionmanager的。
2. **Reachability**：网络状态监听模块
3. **Security**：网络通讯安全策略模块 
3. **Seriaalization**：网络通信信息序列化、反序列化模块
4. **UIKit**：对于IOSUIKit的扩展库

## 核心模块NSURLSession

### 1、NSURLSession由三个基本模块构成：

* NSURLSession
* NSURLSessionConfiguation
* NSURLSessionTask

NSURLSession相对于平时通信中的会话，但本身却不会进行网络数据传输，它会通过多个NSURLSessionTask去执行每次的网络请求

NSURLSession的行为取决于三个方面。包括

* NSURLSession的类型
* NSURLSessionTask的类型
* 在创建task时APP是否处于前端


### 2、NSURLSession有三种类型

* defaultSession（默认会话模式）：将cache和creditials储存于本地

* Ephemeral Session（瞬时会话模式）：对数据更加保密安全，并不会向本地储存任何数据，将cache和creditials储存在内存中，并和Session绑定，当Session销毁时，对应的数据也会被销毁。

* backgroundSession（后台会话模式）：可以时APP处于后台时继续数据传输，其行为与defaultSession类似，但是所有的数据传输均由一个非本APP的进程来管理。也有一些功能上的限制。

**在创建Session对象时通过NSURLSessionConfigration来配置，可设置Session的delegate，Session一但配置完成，就不能修改，除非创建一个新的Session对象。**

### 3、NSURLSessionTask包括三种Task类型

* NSURLSessionDataTask
* NSURLSessionDownLoadTask
* NSURLSessionUploadTask

所有的Task状态都是暂停的，需要用[Task resume]启动Task

### 4、NSURLSession有两种获取数据的方式

* 初始化session时指定delegate，在代理方法中返回数据，需要实现NSURLSession的两个代理方法
* 初始化Session时未指定delegate的，通过block回调返回数据。

### 5、NSURLSession对象的销毁，有两种销毁模式

* - (void)invalidateAndCancel 取消该Session中的所有Task，销毁所有delegate、block和Session自身，调用后Session不能再复用
* - (void)finishTasksAndInvalidate 会立即返回，但不会取消已启动的task，而是当这些task完成时，调用delegate

这里有个地方需要注意，即：NSURLSession对象对其delegate都是强引用的，只有当Session对象invalidate， 才会释放delegate，否则会出现memory leak。

**使用Session加速网络访问速度，使用同一个Session中的task访问数据，不用每次都实现三次握手，复用之前服务器和客户端之间的网络链接，从而加快访问速度。**

## 网络请求的过程

创建NSURLSessionConfig对象，用创建的config对象配置初始化NSURLSession，创建NSURLSessionTask对象并resume执行，用delegate或者block回调返回数据。

AFURLSessionManager封装了上述网络交互功能
AFURLSessionManager请求过程：

1. 初始化AFURLSessionManager
2. 获取AFURLSessionManager的Task对象
3. 启动Task

AFURLSessionManager会为每一个Task创建一个AFURLSessionmanagerTaskDelegate对象，manager会让其处理各个Task的具体事务，从而实现了manager对多个Task的管理

初始化好manager后，获取一个网络请求的Task，生成一个Task对象，并创建了一个AFURLSessionmanagerTaskDelegate并将其关联，设置Task的上传和下载delegate，通过KVO监听download进度和upload进度

### NSURLSessionDelegate的响应

因为AFURLSessionmanager所管理的AFURLSession的delegate指向其自身，因此所有的NSURLSessiondelegate的回调地址都是AFURLSessionmanager，而AFURLSessionmanager又会根据是否需要具体处理会将AF delegate所响应的delegate，传递到对应的AF delegate去。


### 参考文章

[AFNetworking实现原理理解](https://www.jianshu.com/p/02b25f6d1e1f)
