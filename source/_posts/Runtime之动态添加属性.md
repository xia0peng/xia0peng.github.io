---
title: Runtime之动态添加属性(四)
date: 2017-02-20 19:17:18

description: 为什么要将Runtime的关联属性单独拿出来写一篇文章呢？ 因为单独来讲一个小知识点更加简洁，容易掌握，也是一种知识详细的梳理过程。篇幅越短读者就不容易疲劳，阅读更有效果。

categories: Runtime
tags: [Objective-C,Runtime]
toc: false 
---

***
[Runtime--Runtime的数据结构(一)](https://xiaopengmonsters.github.io/2018/05/03/Runtime--Runtime%E7%9A%84%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84/)
[Runtime--类对象与元类对象(二)](https://xiaopengmonsters.github.io/2018/05/13/Runtime--%E7%B1%BB%E5%AF%B9%E8%B1%A1%E4%B8%8E%E5%85%83%E7%B1%BB%E5%AF%B9%E8%B1%A1/)
[Runtime和消息转发(三)](https://xiaopengmonsters.github.io/2017/02/14/Runtime/)
[Runtime之动态添加属性(四)](https://xiaopengmonsters.github.io/2017/02/20/Runtime%E4%B9%8B%E5%8A%A8%E6%80%81%E6%B7%BB%E5%8A%A0%E5%B1%9E%E6%80%A7/)
[Runtime之动态添加方法(五)](https://xiaopengmonsters.github.io/2017/02/21/Runtime%E4%B9%8B%E5%8A%A8%E6%80%81%E6%B7%BB%E5%8A%A0%E6%96%B9%E6%B3%95/)
***

为什么要将Runtime的关联属性单独拿出来写一篇文章呢？ 因为单独来讲一个小知识点更加简洁，容易掌握，也是一种知识详细的梳理过程。篇幅越短读者就不容易疲劳，阅读更有效果。

动态添加方法在后面也会单独写一篇博文。

## 给分类添加属性

* 原理：给一个类声明属性，其实本质就是给这个类添加关联，并不是直接把这个值的内存空间添加到类存空间。


对象关联允许开发者对已经存在的类在 Category 中添加自定义的属性：


```
OBJC_EXPORT void objc_setAssociatedObject(id object, const void *key, id value, objc_AssociationPolicy policy) __OSX_AVAILABLE_STARTING(__MAC_10_6, __IPHONE_3_1);
```

参数1：object 是源对象

参数2：value 是被关联的对象

参数3：key 是关联的键，objc_getAssociatedObject 方法通过不同的 key 即可取出对应的被关联对象

参数4：policy 是一个枚举值，表示关联对象的行为，从命名就能看出各个枚举值的含义

要取出被关联的对象使用 objc_getAssociatedObject 方法即可，要删除一个被关联的对象，使用 objc_setAssociatedObject 方法将对应的 key 设置成 nil 即可.

```
/** 这个也可以取消关联 */
void objc_removeAssociatedObjects(id object)
```

### 应用情景

* 给NSObject类添加一个name属性
* 给UIButton或UIView添加一个单击事件回调属性
* 给控件（UILable，UIButton，UIView等）添加一个角标显示的信息的属性，以及信息的颜色，字体大小等属性

下面我们以给 UIButton 添加一个监听单击事件的 block 属性为例：

创建一个UIButton的分类

在.h文件：

```
#import <UIKit/UIKit.h>

typedef void(^clickBlock)(void);

@interface UIButton (block)

/*
 * 在分类中声明一个属性时,只会生成setter和getter方法的声明,并不能生成setter和getter方法的
 * 实现以及带下划线的成员变量.
 * 所以, 在分类中有两种方式声明一个属性
 */

/** 第一种写法 */
@property (nonatomic,copy) clickBlock click;

/** 第二种写法 */
//@property clickBlock click;

@end
```
在.m文件：

```
#import "UIButton+block.h"
#import <objc/runtime.h>

/** 定义关联的key */
static const void *clickKey = "click";

@implementation UIButton (block)

//Category中的属性，只会生成setter和getter方法，不会生成成员变量

-(void)setClick:(clickBlock)click{
    
    /* 产生关联,让某个对象(name)与当前对象的属性(name)产生关联
     参数1: id object :表示给哪个对象添加关联
     参数2: const void *key : 表示: id类型的key值(以后用这个key来获取属性) 属性名
     参数3: id value : 属性值
     参数4: 策略, 是个枚举(点进去,解释很详细)
     
     单词Associated 关联
     */
    objc_setAssociatedObject(self, clickKey, click, OBJC_ASSOCIATION_COPY_NONATOMIC);
    
    [self removeTarget:self action:@selector(buttonClick) forControlEvents:UIControlEventTouchUpInside];
    
    if (click) {
        [self addTarget:self action:@selector(buttonClick) forControlEvents:UIControlEventTouchUpInside];
    }
}

- (clickBlock)click{
    return objc_getAssociatedObject(self, clickKey);
}

- (void)buttonClick{
    if (self.click) {
        self.click();
    }
}

@end

```

这样，我们就成功的给UIButton类添加了一个监听单击事件的block属性

使用：

```
UIButton *button = [[UIButton alloc] init];
    
button.frame = self.view.bounds;
    
[self.view addSubview:button];
    
button.click = ^{
    
    NSLog(@"点击了button");
    
};
```

## 疑难杂症
为什么我们会用runtime方法来给系统的类动态添加属性? 直接在分类的.m文件中定义一个全局的`clickBlockXP _clickXp;`也可以达到相同的效果，为什么不能那样做呢

就像这样：

```
#import "UIButton+block.h"

typedef void(^clickBlockXP)(void);

/** 定义的全局block */
clickBlockXP _clickXp;

@implementation UIButton (block)

-(void)setClick:(clickBlock)click{
    
    _clickXp = click;
    
    [self removeTarget:self action:@selector(buttonClick) forControlEvents:UIControlEventTouchUpInside];
    
    if (click) {
        [self addTarget:self action:@selector(buttonClick) forControlEvents:UIControlEventTouchUpInside];
    }
}

- (clickBlock)click{
    return _clickXp;
}

- (void)buttonClick{
    if (self.click) {
        self.click();
    }
}
```

**不能这样做的原因**：属性保存的地址不同，如果使用疑难杂症中所述的那样，虽然可以达到效果，但是当button销毁了，button.click并不会随着它的销毁而销毁，这样就不是关联关系了，所以这时候就需要使用到runtime，那么就需要将某个属性保存到它的对象里，给哪个对象添加属性，就将之保存到谁里面。属性和对象共存亡。



### 文章链接

[Runtime和消息转发](https://xiaopengmonsters.github.io/2017/02/14/Runtime/)

[Runtime之动态添加方法](https://xiaopengmonsters.github.io/2017/02/21/Runtime%E4%B9%8B%E5%8A%A8%E6%80%81%E6%B7%BB%E5%8A%A0%E6%96%B9%E6%B3%95/)

[Runtime全方位装逼指南](http://www.cocoachina.com/ios/20160523/16386.html)

[runtime简单使用之动态添加属性](https://www.jianshu.com/p/e52c17db0aa9)