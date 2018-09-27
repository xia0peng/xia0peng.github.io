---
title: Runtime--类对象与元类对象(二)
date: 2017-08-16 01:01:18

description: 深入理解 对象、类对象、元类对象

categories: Runtime
tags: [Objective-C]
---


***
[Runtime--Runtime的数据结构(一)](https://xiaopengmonsters.github.io/2018/05/03/Runtime--Runtime%E7%9A%84%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84/)
[Runtime--类对象与元类对象(二)](https://xiaopengmonsters.github.io/2018/05/13/Runtime--%E7%B1%BB%E5%AF%B9%E8%B1%A1%E4%B8%8E%E5%85%83%E7%B1%BB%E5%AF%B9%E8%B1%A1/)
[Runtime和消息转发(三)](https://xiaopengmonsters.github.io/2017/02/14/Runtime/)
[Runtime之动态添加属性(四)](https://xiaopengmonsters.github.io/2017/02/20/Runtime%E4%B9%8B%E5%8A%A8%E6%80%81%E6%B7%BB%E5%8A%A0%E5%B1%9E%E6%80%A7/)
[Runtime之动态添加方法(五)](https://xiaopengmonsters.github.io/2017/02/21/Runtime%E4%B9%8B%E5%8A%A8%E6%80%81%E6%B7%BB%E5%8A%A0%E6%96%B9%E6%B3%95/)
***


## 对象、类对象、元类对象

* **类对象**存储实例方法列表等信息

* **元类对象**存储类方法列表等信息

#### 通过一副经典图来了解一下

![](/img/类对象与元类对象.png)

**实例和类对象之间的关系**

* 首先看到的是一个根类（Root class）它的类对象，根类的父类指向的是 nil，对应的就是 OC 中 NSObject 这个类，它实际上是没有父类的，父类是nil

* 对于这个类对象，他有子类（Superclass）以及子类的子类（Subclass），当给定一个实例（instance）的时候，由于这个实例是 id 类型的，也就是 objc_object 数据结构，它当中有一个 isa 成员变量，所指向的就是这个实例所对应的类对象，实例通过 isa 指针找到自己的类对象

**类对象和元类对象之间有什么区别和联系**

* 最右侧的这一部分分别为，根类的元类对象，父类的元类对象和子类的元类对象，他们三个之间同样也是父子关系或者说是继承关系，对于类对象由于它是 objc_class 数据结构，继承与 objc_object，所以他们也分别有 isa 指针，那么对应的根类的类对象它的 isa 指针就指向根元类对象，父类的类对象当中的 isa 指针指向父类的元类对象，子类的 isa 指针指向的是子类的元类对应

阐述问题时不仅要说出，实例对象可以通过 isa 指针找到它的类对象，类对象当中存在方法列表，类对象通过 isa 找到它的元类对象，从而可以访问一些关于类方法列表等信息，还要说出类对象和元类对象都是 objc_class 数据结构的，这个数据结构由于继承了 objc_object，所以他们才有 isa 指针，进而才能实现刚才说描述的联系

**元类对象的 isa 指针指向**

* 类对象和元类对象都是 objc_class 数据结构，objc_class由于继承了 objc_object，所以元类对象也有 isa 指针

* 对于任何一个元类对象，他的 isa 指针都指向根元类对象，包括根元类对象子身

* 根元类对象 superclass 指针指向的是根类对象，这个指向就决定了重要的性质

**如果调用的一个类方法没有实现，但是有同名的实例方法实现，这个时候会不会发生崩溃，会不会产生实际的调用（性质来了）**

* 就是由于根元类对象 superclass 指针指向了根类对象，当在元类对象当中去找类方法列表没有查找到的情况下，就会顺着这个指针去实例方法列表当中查找，如果有同名方法就会执行同名方法的实例方法


**消息传递的过程（现在理解是不是很清晰~~~）**

比如说调用了一个实例方法 A

* 首先系统会根据当前实例的 isa 指针找到它的类对象
* 然后在类对象中遍历方法列表查找同名的方法实现
* 如果没有查找到的话，就会顺次以 superclass 指针的指向去查找父类类对象的方法列表，然后再顺次查找根类对象的方法列表
* 如果没有找到就会进入消息转发流程

如果调用了一个类方法 A

* 首先通过类对象的 isa 指针找到元类对象，然后顺次（从当前元类对象到根元类对象）遍历方法列表，然后再到根类对象，再到 nil
* 所以在遍历类方法的过程当中和遍历实例方法的过程当中有一个区别，区别就在根元类对象 superclass 指针的指向


## 思考


```
// .h

#import "Mobile.h"

@interface Phone : Mobile

@end

// .m

@implementation Phone

- (id)init
{
    self = [super init];
    if (self) {
        NSLog(@"%@", NSStringFromClass([self class]));
        NSLog(@"%@", NSStringFromClass([super class]));
    }
    return self;
}

```

会输出什么？

先思考一会儿。。。

解析：

[self class] 消息的接收者是当前对象，从当前类对象的方法列表开始查找，class 这个方法实际上是在 NSObject 当中才有实现的，super 调用它的 class ，实际的接收着仍然是当前对象，也就是仍然是 self，只不过调用 [super class] 的时候是从当前对象的父类对象（跨越了类对象）当中开始查找 class 的方法实现，不论从哪开始查找，最终都是找到 NSObject 当中

代码角度：

[super class]会被转换为下面代码

```
struct objc_super objcSuper = {
    self,
    class_getSuperclass([self class]),
};
id (*sendSuper)(struct objc_super*, SEL) = (void)objc_msgSendSuper;
sendSuper(&objcSuper, @selector(class));

```

super的调用会被转换为objc_msgSendSuper的调用，并传入一个objc_super类型的结构体。结构体有两个参数，第一个就是接受消息的对象，第二个是[super class]对应的父类。

```
struct objc_super {
    __unsafe_unretained _Nonnull id receiver;
    __unsafe_unretained _Nonnull Class super_class;
};

```

由此可知，虽然调用的是[super class]，但是接受消息的对象还是self。然后来到父类Father的class方法中，输出self对应的类 Phone。

两个都是输出 Phone 