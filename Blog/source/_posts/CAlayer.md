---
title: CAlayer
date: 2016-12-18 13:54:01
categories: iOS
description: 在平时开发中，我们经常使用到UILable、UIButton、UIImageView、UITextField等都是UIView的子类（可以去了解一下UIKit框架），UIView之所以能显示在屏幕上，完全是因为它内部的一个图层。
tags: [Objective-C]
---

主要从以下几个方面了解CAlayer：

1. CALayer和UIView的关系
2. CALayer的基本属性
3. position和anchorPoint的作用
4. CALayer和UIView的选择
3. CALayer所属框架
4. CALayer的隐式动画
4. 自定义图层


在平时开发中，我们经常使用到`UILable`、`UIButton`、`UIImageView`、`UITextField`等都是UIView的子类（可以去了解一下UIKit框架），UIView之所以能显示在屏幕上，完全是因为它内部的一个图层。


这里有[UIKit框架结构图](https://upload-images.jianshu.io/upload_images/1829339-9089f59e204212d2)，此图来自简书@不懂技术的爱迪生。

这个图层就是CALayer，它并不属于UIKit框架，后面会讲到。其实UIView本身不具备显示的功能，是它内部的层才有显示功能。

当UIView需要显示到屏幕上时，会调用drawRect:方法进行绘图，并且会将所有内容绘制在自己的图层上，绘图完毕后，系统会将图层拷贝到屏幕上，于是就完成了UIView的显示。

# CALayer和UIView的关系
* 在创建UIView对象时，UIView内部会自动创建一个图层(即CALayer对象)，通过UIView的layer属性可以访问这个层。

```
@property(nonatomic,readonly,retain) CALayer *layer; 

```

* CALayer的基本属性

``` bash
 宽度和高度
 @property CGRect bounds;
 
 位置(默认指中点，具体由anchorPoint决定)
 @property CGPoint position;
 
 锚点(x,y的范围都是0-1)，决定了position的含义
 @property CGPoint anchorPoint;
 
 背景颜色(CGColorRef类型)
 @property CGColorRef backgroundColor;
 
 形变属性
 @property CATransform3D transform;
 
 让图片显示固定区域(比如可以让一个图片只显示上半部分或者下半部分甚至更小)
 @property CGRect contentsRect;
 
 边框颜色(CGColorRef类型)
 @property CGColorRef borderColor;
 
 边框宽度
 @property CGFloat borderWidth;
 
 圆角半径
 @property CGFloat cornerRadius;
 
 内容(比如设置为图片CGImageRef)
 @property(retain) id contents;
```
* CALayer的阴影属性

```
阴影颜色
@property CGColorRef shadowColor;
 
阴影不透明(0.0 ~ 1.0)
@property float shadowOpacity;
 
阴影偏移位置
@property CGSize shadowOffset;
```

* CALayer的使用时的问题（存在的疑惑）

首先，CALayer是定义在QuartzCore框架中的(Core Animation)
CGImageRef、CGColorRef两种数据类型是定义在CoreGraphics框架中的，UIColor、UIImage是定义在UIKit框架中的
 
其次，QuartzCore框架和CoreGraphics框架是可以跨平台使用的，在iOS和Mac OS X上都能使用，但是UIKit只能在iOS中使用。
 
为了保证可移植性，QuartzCore不能使用UIImage、UIColor，只能使用CGImageRef、CGColorRef。

## CALayer的position和anchorPoint

 position和anchorPoint是CALayer非常重要的2个属性


```
@property CGPoint position;
```

* position:它是用来设置当前的layer在父控件当中的位置的,
所以它的坐标原点.以父控件的左上角为(0.0)点.

```
@property CGPoint anchorPoint;
```



* anchorPoint称为“定位点”、“锚点”
决定着CALayer身上的哪个点会在position属性所指的位置，以自己的左上角为原点(0, 0)，它的x、y取值范围都是0~1，默认值为（0.5, 0.5），意味着锚点在layer的中间.



# UIView和CALayer的选择


通过CALayer，就能做出跟UIView一样的界面效果
 
但是，对比CALayer，UIView多了一个事件处理的功能。也就是说，CALayer不能处理用户的触摸事件，而UIView可以，
所以，如果显示出来的东西需要跟用户进行交互的话，用UIView；如果不需要跟用户进行交互，用UIView或者CALayer都可以。
当然，CALayer的性能会高一些，因为它少了事件处理的功能，更加轻量级。


那么，从实质来讲，UIView仅仅是对CALayer的一层封装，实现了CALayer的delegate，提供了处理事件交互的具体功能，还有动画底层方法的高级API。可以说CALayer是UIView的内部实现细节。

# 隐式动画

先了解是什么根层和非根层.


* 根层:UIView内部自动关联着的那个layer我们称它是根层
* 非根层:自己手动创建的层,称为非根层

什么是隐式动画？

* 隐式动画就是当对非根层的部分属性(bounds、backgroundColor、position等)进行修改时, 它会自动的产生一些动画的效果.
我们称这个默认产生的动画为隐式动画.

关闭隐式动画效果


* 首先要了解动画底层是怎么做的.动画的底层是包装成一个事务来进行的.
* 什么是事务?
很多操作绑定在一起,当这些操作执行完毕后,才去执行下一个操作.

```
开启事务
[CATransaction begin];

设置事务没有动画
[CATransaction setDisableActions:NO];

设置隐式动画执行的时长
[CATransaction setAnimationDuration:2];

提交事务
[CATransaction commit];
```


# 自定义图层

## 方法一

### 方法描述：


设置CALayer的delegate，然后让delegate实现drawLayer:inContext:方法，当CALayer需要绘图时，会调用delegate的drawLayer:inContext:方法进行绘图。

```
//创建图层
CALayer *layer = [CALayer layer];
// 设置delegate  设置了CALayer的delegate，这里的self是指控制器

layer.delegate = self;

// 设置层的宽高
layer.bounds = CGRectMake(0, 0, 100, 100);

// 设置层的位置
layer.position = CGPointMake(100, 100);

// 开始绘制图层 需要调用setNeedsDisplay这个方法，才会通知delegate进行绘图
[layer setNeedsDisplay];

//将图层添加到view的根层上
[self.view.layer addSublayer:layer];

```

```
#pragma mark 画一个矩形框
- (void)drawLayer:(CALayer *)layer inContext:(CGContextRef)ctx {
    
    // 设置蓝色
    CGContextSetRGBStrokeColor(ctx, 0, 0, 1, 1);
    
    // 设置边框宽度
    CGContextSetLineWidth(ctx, 10);
    
    // 添加一个跟层一样大的矩形到路径中
    CGContextAddRect(ctx, layer.bounds);
    
    // 绘制路径
    CGContextStrokePath(ctx);
}

```
### 效果

![](/img/五角星.png)

## 方法二

### 方法描述：
* 创建一个CALayer的子类，然后覆盖drawInContext:方法，使用Quartz2D API进行绘图

```
//创建自定义图层类
XPCALayer *layer = [XPCALayer layer];

// 设置层的宽高
layer.bounds = CGRectMake(0, 0, 100, 100);
 
// 设置层的位置
layer.position = CGPointMake(100, 100);
    
// 设置图层颜色
layer.backgroundColor = [UIColor redColor].CGColor;
    
// 开始绘制图层  需要调用setNeedsDisplay这个方法，才会触发drawInContext:方法的调用，然后进行绘图
[layer setNeedsDisplay];
    
 // 将图层添加到view的根层上
[self.view.layer addSublayer:layer];
```

* 类内的实现，重写父类方法，绘制一个实心三角

```
- (void)drawInContext:(CGContextRef)ctx{
    
    // 设置为蓝色
    CGContextSetRGBFillColor(ctx, 0, 0, 1, 1);
    
    // 设置起点
    CGContextMoveToPoint(ctx, 50, 0);
    
    // 从(50, 0)连线到(0, 100)
    CGContextAddLineToPoint(ctx, 0, 100);
    
    // 从(0, 100)连线到(100, 100)
    CGContextAddLineToPoint(ctx, 100, 100/3);
    
    CGContextAddLineToPoint(ctx, 0, 100/3);
    
    CGContextAddLineToPoint(ctx, 100, 100);
    
    // 合并路径，连接起点和终点
    CGContextClosePath(ctx);
    
    // 绘制路径
    CGContextFillPath(ctx);
}

```
### 效果

![](/img/正方形.png)


## 总结

### 1.注意


* 无论采取哪种方法来自定义层，都必须调用CALayer的setNeedsDisplay方法才能正常绘图。
 
###  2.UIView的详细显示过程
 * 当UIView需要显示时，它内部的层会准备好一个CGContextRef(图形上下文)，然后调用delegate(这里就是UIView)的drawLayer:inContext:方法，并且传入已经准备好的CGContextRef对象。而UIView在drawLayer:inContext:方法中又会调用自己的drawRect:方法
 * 平时在drawRect:中通过UIGraphicsGetCurrentContext()获取的就是由层传入的CGContextRef对象，在drawRect:中完成的所有绘图都会填入层的CGContextRef中，然后被拷贝至屏幕
  
