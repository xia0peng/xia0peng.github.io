---
title: Runtime之动态添加方法
date: 2017-2-21 18:13:03
description: 动态添加的方法的作用就是去处理未实现的实例方法或者是类方法，它的调用时刻:只要我们调用了一个不存在的方法时，它就会动态方法解析，接下来就会进入消息转发流程，这此过程中我们可以拦截然后动态的添加方法，防止程序崩溃。
categories: [原理,Runtime]
tags: [Objective-C,Runtime]
toc: false 
---

* 动态添加的方法的作用就是去处理未实现的实例方法或者是类方法，它的调用时刻: 只要我们调用了一个不存在的方法时，它就会动态方法解析，接下来就会进入消息转发流程，这此过程中我们可以拦截然后动态的添加方法，防止程序崩溃。

* 如果一个类方法非常多，加载类到内存的时候也比较耗费资源，需要给每个方法生成映射表，可以使用动态给某个类添加方法解决。

* 有没有使用performSelector，其实主要想问你有没有动态添加过方法。使用performSelector可以调用一个没有实现的方法，但是会报错。

## 动态添加方法

以一个示例讲解：

1、 创建一个熊猫Panda类，Panda类并没有实现eat方法，可以使用performSelector调用一个没有实现的方法，但是会报错。

```
Panda *pan = [[Panda alloc] init];
    
// 默认Panda，没有实现eat方法，不能直接调用，可以通过performSelector调用，但是会报错。
// 动态添加方法就不会报错
    
/** 无参 */
//[pan performSelector:@selector(eat)];
    
/** 有参 */
[pan performSelector:@selector(eat:) withObject:@521];
    
```

2、在Panda类中添加方法（以有参为例）

```
#import "Panda.h"
#import <objc/runtime.h>

@implementation Panda

// 默认方法都有两个隐式参数，
// 定义添加的方法
void eat(id self, SEL sel, NSNumber *meter)
{
    NSLog(@"\n%@\n%@\n%@",self,NSStringFromSelector(sel),meter);
}

// 当一个对象调用未实现的方法，会调用这个方法处理,并且会把对应的方法列表传过来.
// 刚好可以用来判断，未实现的方法是不是我们想要动态添加的方法
+ (BOOL)resolveInstanceMethod:(SEL)sel
{
    
    if (sel == NSSelectorFromString(@"eat:")) {
        // 动态添加eat方法
        
        // 第一个参数：给哪个类添加方法
        // 第二个参数：添加方法的方法编号
        // 第三个参数：添加方法的函数实现（函数地址）
        // 第四个参数：函数的类型，(返回值+参数类型) v:void @:对象->self :表示SEL->_cmd
        class_addMethod(self, sel, (IMP) eat, "v@:");
        
        return YES;
    }
    
    return [super resolveInstanceMethod:sel];
}

@end

```

### 使用场景

一个类方法非常多，一次性加载到内存，比较耗费资源，为什么动态添加方法? OC都是懒加载，有些方法可能很久不会调用。

比如电商，视频，社交等一些软件会有有收费项目或者会员机制，那么只有在开通会员的时候才会拥有特定功能，然而存在相当一部门用户是没有使用收费功能，或者是没有开通开通会员的，我们就在这些用户使用时不加载这些方法（这个方法的类是要加载的），后面利用Runtime动态的添加这些方法，以达到性能最大化。

### 文章链接

[Runtime和消息转发](https://xiaopengmonsters.github.io/2017/02/14/Runtime/)

[Runtime之动态添加属性](https://xiaopengmonsters.github.io/2017/02/20/Runtime%E4%B9%8B%E5%8A%A8%E6%80%81%E6%B7%BB%E5%8A%A0%E5%B1%9E%E6%80%A7/)

