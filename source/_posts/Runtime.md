---
title: Runtime和消息转发
date: 2017-02-14 11:20:28

description: 之前有学习过Runtime相关知识，但是都是比较零散的积累，有了自己的博客，那就是时候系统的整理学习Runtime的相关知识了。

categories: [原理,Runtime]
tags: [Objective-C,Runtime]
---

# 什么是Runtime

* Runtime简称运行时。OC就是运行时机制，其中最主要的是`消息机制`。
* 对于C语言，函数的调用在编译的时候会决定调用哪个函数。
* 对于OC的函数，属于动态调用过程，在编译的时候并不能决定真正调用哪个函数，只有在真正运行的时候才会根据函数的名称找到对应的函数来调用。
* Runtime基本上是用C汇编写的，这个库使得C语言有了面向对象的能力。

## 消息机制（方法调用的本质）
* 方法调用的本质，就是让对象发送消息。
* OC中所有方法的调用都会转化为objc_msgSend来实现。

### SEL
其定义如下

```
typedef struct objc_selector *SEL；     
```

SEL又叫选择器，类成员方法的指针，但不同于C语言中的函数指针，函数指针直接保存了方法的地址，但SEL只是方法编号。

### IMP

IMP实际上是一个函数指针，指向方法实现的地址。
其定义如下:

```
id (*IMP)(id, SEL,...)
```

第一个参数：是指向self的指针(如果是实例方法，则是类实例的内存地址；如果是类方法，则是指向元类的指针)

第二个参数：是方法选择器(selector)

接下来的参数：方法的参数列表。

### Method

Method方法的结构体，其中保存了方法的名字，实现和类型描述字符串，则定义如下：

```
typedef struct objc_method *Method
    struct objc_method{
        SEL method_name      OBJC2_UNAVAILABLE; // 方法名
        char *method_types   OBJC2_UNAVAILABLE;
        IMP method_imp       OBJC2_UNAVAILABLE; // 方法实现
    }
```
可以看到该结构体中包含一个SEL和IMP，实际上相当于在SEL和IMP之间作了一个映射。有了SEL，我们便可以找到对应的IMP，从而调用方法的实现代码。

**原理：对象根据方法编号SEL去Dispath table表（映射表）寻找到对应的IMP，IMP就是一个函数指针，然后执行这个方法**

![](/img/消息机制.png)


下面演示了这样一个消息的基本框架：
当消息发送给一个对象时首先从运行时系统缓存使用过的方法中寻找。
如果找到，执行该方法,如未找到继续执行下面的步骤

objc_msgSend通过对象的isa指针获取到类的结构体，然后在方法分发表里面查找方法的selector。
如果没有找到selector，objc_msgSend结构体中的指向父类的指针找到其父类，并在父类的分发表里面查找方法的selector。
依此，会一直沿着类的继承体系到达NSObject类。一旦定位到selector，函数会就获取到了实现的入口点，并传入相应的参数来执行方法的具体实现,并将该方法添加进入缓存中如果最后没有定位到selector，则会走消息转发流程。 

## 消息转发


### 1、动态方法解析


如果在上述索引未能成功，则首先会转入动态方法解析，该过程允许我们使用`class_addMethod`函数动态提供相对于的方法实现

```
/** 如果调用的是实例方法，则会调用该方法 */
+ (BOOL)resolveInstanceMethod:(SEL)sel;

/** 如果调用的是类方法，则会调用该方法 */
+ (BOOL)resolveClassMethod:(SEL)sel;

```

下面举一个例子：

1. 创建了一个Animal类的对象an，然后调用an的eat方法，注意，这个eat方法是没有实现的
2. 进入Animal类的.m文件，我们实现了resolveInstanceMethod这个方法为我的Animal类动态增加了一个eat方法的实现。（[这里](https://xiaopengmonsters.github.io/2017/02/21/Runtime%E4%B9%8B%E5%8A%A8%E6%80%81%E6%B7%BB%E5%8A%A0%E6%96%B9%E6%B3%95/)会详细讲解Runtime动态添加方法）

当外部调用[an eat]时，由于我们没有实现eat对应的方法，那么系统会调用resolveInstanceMethod让你去做一些其他操作。（也可以不做操作，如果不操作会进入下一步消息转发，在这个例子中，我为Animal方法动态增加了实现。）

```
void eat(id self, SEL _cmd, NSString *string){
    NSLog(@"add C IMP %@", string);
}

+ (BOOL)resolveInstanceMethod:(SEL)sel{
    if (sel == @selector(eat)) {

        class_addMethod(self, sel, (IMP) eat, "v@:");

        return YES;
    }
    return [super resolveInstanceMethod:sel];
}
```

### 2、消息转发

* 如果不对上面的`resolveInstanceMethod`方法做任何处理，直接调用父类方法，那么，系统会来到了`forwardingTargetForSelector`方法


```
/** 这个方法返回你需要转发消息的对象 */
- (id)forwardingTargetForSelector:(SEL)aSelector;

```


接着上面的例子：

1. 不对`resolveInstanceMethod`进行处理，直接调用父类方法。
2. 新建了一个熊猫类Panda，并且实现了Panda的eat方法。

``` 
+ (BOOL)resolveInstanceMethod:(SEL)sel{
    
    return [super resolveInstanceMethod:sel];
}

- (id)forwardingTargetForSelector:(SEL)aSelector;{
    return [[Panda alloc]init];
}
```


当程序来到`forwardingTargetForSelector`方法时，我们现在返回一个Panda类的实例对象，继续运行，程序就来到了Panda类的eat方法，这样，就实现了消息转发。


* 如果`forwardingTargetForSelector`方法也不实现的话，那么程序会进入下面两个函数



```
/** 这个方法是用来生成方法签名的，这个签名就是给forwardInvocation中的参数NSInvocation调用的 */
- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector;

/** 在这个方法中用你要转发的那个对象调用这个对应的签名 */
- (void)forwardInvocation:(NSInvocation *)anInvocation;
```


unrecognized selector sent to instance这个错误的原因，就是因为`methodSignatureForSelector`这个方法中，由于没有找到eat对应的实现方法，所以返回了一个空的方法签名，最终导致程序报错崩溃。

所以我们需要做的是自己新建方法签名，再在`forwardInvocation`中用你要转发的那个对象调用这个对应的签名，这样也实现了消息转发。


```
- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector{
    
    NSString *sel = NSStringFromSelector(aSelector);
    
    if ([sel isEqualToString:@"eat"]) {
        
        /** 为你的转发发放手动生成签名 */
        return [NSMethodSignature signatureWithObjCTypes:"v@:"];
    }

    return [super methodSignatureForSelector:aSelector];
}

- (void)forwardInvocation:(NSInvocation *)anInvocation{
    
    SEL selector = [anInvocation selector];
    
    /** 新建需要转发消息的对象 */
    Panda *panda = [[Panda alloc]init];
   
    if ([panda respondsToSelector:selector]) {
        
        /** 唤醒这个方法 */
        [anInvocation invokeWithTarget:panda];
    }
}

```


### Runtime文章链接

[Runtime之动态添加属性](https://xiaopengmonsters.github.io/2017/02/20/Runtime%E4%B9%8B%E5%8A%A8%E6%80%81%E6%B7%BB%E5%8A%A0%E5%B1%9E%E6%80%A7/)

[Runtime之动态添加方法](https://xiaopengmonsters.github.io/2017/02/21/Runtime%E4%B9%8B%E5%8A%A8%E6%80%81%E6%B7%BB%E5%8A%A0%E6%96%B9%E6%B3%95/)
