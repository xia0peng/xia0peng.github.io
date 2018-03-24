---
title: iOS中的多线程
date: 2017-11-18 11:11:18

description: 在 iOS 中其实目前有 4 套多线程方案

categories: [原理]
tags: [Objective-C]
toc: true 
---

在 iOS 中其实目前有 4 套多线程方案，他们分别是：

1. Pthreads
2. NSThread
3. GCD
4. NSOperation & NSOperationQueue

## Pthreads     

**特点：**

* 一套通用的多线程API
* 适用于Unix\Linux\Windows等系统
* 跨平台\可移植
* 使用难度大

**使用语言：**基于 c语言 的框架

**使用频率：**几乎不用

**线程生命周期：**由程序员手动进行管理

## NSThread

**特点：**

* 使用更加面向对象
* 简单易用，可直接操作线程对象

**使用语言：**OC语言

**使用频率：**偶尔使用

**线程生命周期：**由程序员手动进行管理


这套方案是经过苹果封装后的，并且完全面向对象的。所以你可以直接操控线程对象，非常直观和方便，是轻量级最低的（优点）。但是，它的生命周期还是需要我们手动管理（缺点），所以这套方案也是偶尔用用，比如 [NSThread currentThread]，它可以获取当前线程类，你就可以知道当前线程的各种属性，用于调试十分方便

### 使用

* 先创建线程类，再启动

```
// 创建
NSThread *thread = [[NSThread alloc] initWithTarget:self selector:@selector(run:) object:nil];

// 启动
[thread start];

```

* 创建并自动启动

```
[NSThread detachNewThreadSelector:@selector(run:) toTarget:self withObject:nil];

```

* 使用 NSObject 的方法创建并自动启动

```
[self performSelectorInBackground:@selector(run:) withObject:nil];
```


## GCD

**特点:**

* 旨在替代NSThread等线程技术
* 充分利用设备的多核（自动）

**使用语言：**C语言

**使用频率：**经常使用

**线程生命周期：**自动管理


它是苹果为多核的并行运算提出的解决方案，所以会自动合理地利用更多的CPU内核（比如双核、四核），最重要的是它会自动管理线程的生命周期（创建线程、调度任务、销毁线程），完全不需要我们管理，我们只需要告诉干什么就行。同时它使用的也是 c语言，不过由于使用了 Block（Swift里叫做闭包），使得使用起来更加方便，而且灵活。所以基本上大家都使用 GCD 这套方案。

## NSOperation

**特点:**

* 基于GCD（底层是GCD）
* 比GCD多了一些更简单实用的功能
* 使用更加面向对象

**使用语言：**OC语言

**使用频率：**经常使用

**线程生命周期：**自动管理

NSOperation 是苹果公司对 GCD 的封装，完全面向对象，所以使用起来更好理解。 大家可以看到 NSOperation 和 NSOperationQueue 分别对应 GCD 的 任务 和 队列 。操作步骤也很好理解：

1. 将要执行的任务封装到一个 NSOperation 对象中。
2. 将此任务添加到一个 NSOperationQueue 对象中。

然后系统就会自动在执行任务。至于同步还是异步、串行还是并行请继续往下看：

### 添加任务

值得说明的是，NSOperation 只是一个抽象类，所以不能封装任务。但它有 2 个子类用于封装任务。分别是：NSInvocationOperation 和 NSBlockOperation 。创建一个 Operation 后，需要调用 start 方法来启动任务，它会 默认在当前队列同步执行。当然你也可以在中途取消一个任务，只需要调用其 cancel 方法即可。

* NSInvocationOperation: 需要传入一个方法名

```
//1.创建NSInvocationOperation对象
NSInvocationOperation *operation = [[NSInvocationOperation alloc] initWithTarget:self selector:@selector(run) object:nil];

//2.开始执行
[operation start];

```

* NSBlockOperation

```
//1.创建NSBlockOperation对象
NSBlockOperation *operation = [NSBlockOperation blockOperationWithBlock:^{
      NSLog(@"%@", [NSThread currentThread]);
  }];

//2.开始任务
[operation start];
  
```

之前说过这样的任务，默认会在当前线程执行。但是 NSBlockOperation 还有一个方法：addExecutionBlock: ，通过这个方法可以给 Operation 添加多个执行 Block。这样 Operation 中的任务 **会并发执行**，它会 **在主线程和其它的多个线程** 执行这些任务，注意下面的打印结果：

```
//1.创建NSBlockOperation对象
NSBlockOperation *operation = [NSBlockOperation blockOperationWithBlock:^{
    NSLog(@"%@", [NSThread currentThread]);
}];

//添加多个Block
//addExecutionBlock 方法必须在 start() 方法之前执行，否则就会报错。
for (NSInteger i = 0; i < 5; i++) {
    [operation addExecutionBlock:^{
        NSLog(@"第%ld次：%@", i, [NSThread currentThread]);
    }];
}

//2.开始任务
[operation start];

```

**打印：**

```
2017-11-18 16:30:29.787013+0800 model[75340:4510678] <NSThread: 0x600000262e00>{number = 1, name = main}

2017-11-18 16:30:29.787049+0800 model[75340:4510790] 第1次：<NSThread: 0x60400046f700>{number = 4, name = (null)}

2017-11-18 16:30:29.787050+0800 model[75340:4510788] 第0次：<NSThread: 0x60400046f6c0>{number = 3, name = (null)}

2017-11-18 16:30:29.787050+0800 model[75340:4510791] 第2次：<NSThread: 0x600000463100>{number = 5, name = (null)}

2017-11-18 16:30:29.787179+0800 model[75340:4510678] 第3次：<NSThread: 0x600000262e00>{number = 1, name = main}

2017-11-18 16:30:29.787181+0800 model[75340:4510790] 第4次：<NSThread: 0x60400046f700>{number = 4, name = (null)}

```

* 自定义Operation

除了上面的两种 Operation 以外，我们还可以自定义 Operation。自定义 Operation 需要继承 NSOperation 类，并实现其 main() 方法，因为在调用 start() 方法的时候，内部会调用 main() 方法完成相关逻辑。所以如果以上的两个类无法满足你的欲望的时候，你就需要自定义了。你想要实现什么功能都可以写在里面。除此之外，你还需要实现 cancel() 在内的各种方法。

### 创建队列

看过上面的内容就知道，我们可以调用一个 NSOperation 对象的 start() 方法来启动这个任务，但是这样做他们默认是 **同步执行** 的。就算是 addExecutionBlock 方法，也会在 **当前线程和其他线程 **中执行，也就是说还是会占用当前线程。这是就要用到队列 **NSOperationQueue** 了。而且，按类型来说的话一共有两种类型：主队列、其他队列。**只要添加到队列，会自动调用任务的 start() 方法**

* 主队列

每套多线程方案都会有一个主线程（当然啦，说的是iOS中，像 pthread 这种多系统的方案并没有，因为 UI线程 理论需要每种操作系统自己定制）。这是一个特殊的线程，必须串行。所以添加到主队列的任务都会一个接一个地排着队在主线程处理。

```
NSOperationQueue *queue = [NSOperationQueue mainQueue];
```


* 其他队列

因为主队列比较特殊，所以会单独有一个类方法来获得主队列。那么通过初始化产生的队列就是其他队列了，因为只有这两种队列，除了主队列，其他队列就不需要名字了。

注意：其他队列的任务会在其他线程并行执行。

```
//1.创建一个其他队列    
NSOperationQueue *queue = [[NSOperationQueue alloc] init];

//2.创建NSBlockOperation对象
NSBlockOperation *operation = [NSBlockOperation blockOperationWithBlock:^{
    NSLog(@"%@", [NSThread currentThread]);
}];

//3.添加多个Block
for (NSInteger i = 0; i < 5; i++) {
    [operation addExecutionBlock:^{
        NSLog(@"第%ld次：%@", i, [NSThread currentThread]);
    }];
}

//4.队列添加任务
[queue addOperation:operation];

```

打印

```
2017-11-18 17:49:25.806408+0800 model[75459:4552310] 第1次：<NSThread: 0x600000271300>{number = 6, name = (null)}

2017-11-18 17:49:25.806410+0800 model[75459:4552309] 第0次：<NSThread: 0x6000002712c0>{number = 5, name = (null)}

2017-11-18 17:49:25.806417+0800 model[75459:4552317] 第2次：<NSThread: 0x60400027bf00>{number = 4, name = (null)}

2017-11-18 17:49:25.806424+0800 model[75459:4552307] <NSThread: 0x600000271280>{number = 3, name = (null)}

2017-11-18 17:49:25.806668+0800 Runtime[75459:4552309] 第3次：<NSThread: 0x6000002712c0>{number = 5, name = (null)}

2017-11-18 17:49:25.806673+0800 model[75459:4552310] 第4次：<NSThread: 0x600000271300>{number = 6, name = (null)}

```

**问题：**将 NSOperationQueue 与 GCD的队列 相比较就会发现，这里没有串行队列，那如果我想要10个任务在其他线程串行的执行怎么办？

**答：**这就是苹果封装的妙处，你不用管串行、并行、同步、异步这些名词。NSOperationQueue 有一个参数 maxConcurrentOperationCount 最大并发数，用来设置最多可以让多少个任务同时执行。当你把它设置为 1 的时候，他不就是串行了嘛

NSOperationQueue 还有一个添加任务的方法，- (void)addOperationWithBlock:(void (^)(void))block; ，这是不是和 GCD 差不多？这样就可以添加一个任务到队列中了，十分方便。

NSOperation 有一个非常实用的功能，那就是添加依赖。比如有 3 个任务：A: 从服务器上下载一张图片，B：给这张图片加个水印，C：把图片返回给服务器。这时就可以用到依赖了:

```
//1.任务一：下载图片
NSBlockOperation *operation1 = [NSBlockOperation blockOperationWithBlock:^{
    NSLog(@"下载图片 - %@", [NSThread currentThread]);
    [NSThread sleepForTimeInterval:1.0];
}];

//2.任务二：打水印
NSBlockOperation *operation2 = [NSBlockOperation blockOperationWithBlock:^{
    NSLog(@"打水印   - %@", [NSThread currentThread]);
    [NSThread sleepForTimeInterval:1.0];
}];

//3.任务三：上传图片
NSBlockOperation *operation3 = [NSBlockOperation blockOperationWithBlock:^{
    NSLog(@"上传图片 - %@", [NSThread currentThread]);
    [NSThread sleepForTimeInterval:1.0];
}];

//4.设置依赖
[operation2 addDependency:operation1];      //任务二依赖任务一
[operation3 addDependency:operation2];      //任务三依赖任务二

//5.创建队列并加入任务
NSOperationQueue *queue = [[NSOperationQueue alloc] init];
[queue addOperations:@[operation3, operation2, operation1] waitUntilFinished:NO];

```

**打印**


```
2017-11-18 18:01:25.806424+0800 model[19392:4637517] 下载图片 - <NSThread: 0x7fc10ad4d970>{number = 2, name = (null)}

2017-11-18 18:01:25.806424+0800 model[19392:4637515] 打水印 - <NSThread: 0x7fc10af20ef0>{number = 3, name = (null)}

2017-11-18 18:01:25.806424+0800 model[19392:4637515] 上传图片 - <NSThread: 0x7fc10af20ef0>{number = 3, name = (null)}

```

* 注意：不能添加相互依赖，会死锁，比如 A依赖B，B依赖A。
* 可以使用 removeDependency 来解除依赖关系。
* 可以在不同的队列之间依赖，反正就是这个依赖是添加到任务身上的，和队列没关系。

### 其他方法

以上就是一些主要方法, 下面还有一些常用方法需要大家注意：

* NSOperation

```
BOOL executing; //判断任务是否正在执行

BOOL finished; //判断任务是否完成

void (^completionBlock)(void); //用来设置完成后需要执行的操作

- (void)cancel; //取消任务

- (void)waitUntilFinished; //阻塞当前线程直到此任务执行完毕

```

* NSOperationQueue

```
NSUInteger operationCount; //获取队列的任务数

- (void)cancelAllOperations; //取消队列中所有的任务

- (void)waitUntilAllOperationsAreFinished; //阻塞当前线程直到此队列中的所有任务执行完毕

[queue setSuspended:YES]; // 暂停queue

[queue setSuspended:NO]; // 继续queue

```

## 多线程的原理

同一时间，CPU只能处理1条线程，只有1条线程在工作（执行），多线程并发（同时）执行，其实是CPU快速地在多条线程之间调度（切换），如果CPU调度线程的时间足够快，就造成了多线程并发执行的假象。

* **问题：**如果线程非常非常多，会发生什么情况？

* **答：**CPU会在N多线程之间调度，CPU会累死，消耗大量的CPU资源，每条线程被调度执行的频次会降低（线程的执行效率降低）


多线程的优点
 
* 能适当提高程序的执行效率；
* 能适当提高资源利用率（CPU、内存利用率）

多线程的缺点

* 开启线程需要占用一定的内存空间（默认情况下，主线程占用1M，子线程占用512KB），如果开启大量的线程，会占用大量的内存空间，降低程序的性能
* 线程越多，CPU在调度线程上的开销就越大
* 程序设计更加复杂：比如线程之间的通信、多线程的数据共享

## 多线程的应用（个别案例）

### 线程同步

线程同步就是为了防止多个线程抢夺同一个资源造成的数据安全问题，所采取的一种措施。当然也有很多实现方法，请往下看：

* **互斥锁 ：**给需要同步的代码块加一个互斥锁，就可以保证每次只有一个线程访问此代码块。

```
@synchronized(self) {
  //需要执行的代码块
}

```

* **同步执行 ：**我们可以使用多线程的知识，把多个线程都要执行此段代码添加到同一个串行队列，这样就实现了线程同步的概念。这里可以使用 GCD 和 NSOperation 两种方案:

```
//GCD
//需要一个全局变量queue，要让所有线程的这个操作都加到一个queue中
dispatch_sync(queue, ^{
    NSInteger ticket = lastTicket;
    [NSThread sleepForTimeInterval:0.1];
    NSLog(@"%ld - %@",ticket, [NSThread currentThread]);
    ticket -= 1;
    lastTicket = ticket;
});


//NSOperation & NSOperationQueue
//重点：1. 全局的 NSOperationQueue, 所有的操作添加到同一个queue中
//       2. 设置 queue 的 maxConcurrentOperationCount 为 1
//       3. 如果后续操作需要Block中的结果，就需要调用每个操作的waitUntilFinished，阻塞当前线程，一直等到当前操作完成，才允许执行后面的。waitUntilFinished 要在添加到队列之后！

NSBlockOperation *operation = [NSBlockOperation blockOperationWithBlock:^{
    NSInteger ticket = lastTicket;
    [NSThread sleepForTimeInterval:1];
    NSLog(@"%ld - %@",ticket, [NSThread currentThread]);
    ticket -= 1;
    lastTicket = ticket;
}];

[queue addOperation:operation];

[operation waitUntilFinished];

//后续要做的事

```

### 延迟执行

延迟执行就是延时一段时间再执行某段代码。下面说一些常用方法。

* perform

```
// 3秒后自动调用self的run:方法，并且传递参数：@"abc"
[self performSelector:@selector(run:) withObject:@"abc" afterDelay:3];

```

* GCD

可以使用 GCD 中的 dispatch_after 方法，OC 和 Swift 都可以使用

```
// 创建队列
dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
// 设置延时，单位秒
double delay = 3; 

dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(delay * NSEC_PER_SEC)), queue, ^{
  // 3秒后需要执行的任务
});

```

* NSTimer

NSTimer 是iOS中的一个计时器类，除了延迟执行还有很多用法

```
[NSTimer scheduledTimerWithTimeInterval:3.0 target:self selector:@selector(run:) userInfo:@"abc" repeats:NO];

```

### 单例模式

```
@interface HbhNetWorkManager : NSObject <NSCopying>

@property (nonatomic, strong) AFHTTPSessionManager *manager;

/**
 *  单例
 *
 *  @return HbhNetWorkManager
 */
+(instancetype) shareInstance;

@end

static HbhNetWorkManager *shareInstance = nil;
+(instancetype) shareInstance{
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        shareInstance=[[HbhNetWorkManager alloc] init];
    });
    return shareInstance;
}
```

### 从其他线程回到主线程的方法

* NSThread

```
//Objective-C
[self performSelectorOnMainThread:@selector(run) withObject:nil waitUntilDone:NO];

//Swift
//swift 取消了 performSelector 方法。

```

* GCD

```
//Objective-C
dispatch_async(dispatch_get_main_queue(), ^{

});

//Swift
dispatch_async(dispatch_get_main_queue(), { () -> Void in

})

```

* NSOperationQueue

```
/Objective-C
[[NSOperationQueue mainQueue] addOperationWithBlock:^{

}];

//Swift
NSOperationQueue.mainQueue().addOperationWithBlock { () -> Void in

}

```

## 多线程的选择（更倾向于哪一种？）

倾向于GCD

因为GCD是用来解决多核编程问题的，会自动合理的利用更多的CPU内核，可以通过GCD和block轻松实现多线程编程，更加有效。但有时候最优的不是GCD，还有一种多线程技术NSOperationQueue，它能够将后台线程以队列方式依序执行，并提供更多操作入口，类似GCD，在NSOperationQueue中，可以随时取消已设定要准备执行的任务，而GCD没法停止已加入queue的block。

## 参考文献

[这里有一篇文章写的非常好，推荐。](https://www.jianshu.com/p/0b0d9b1f1f19)
