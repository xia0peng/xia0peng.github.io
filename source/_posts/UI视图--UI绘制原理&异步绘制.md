---
title: UI视图--UI绘制原理&异步绘制
date: 2018-04-13 15:22:09
description: UI绘制原理&异步绘制
categories: UI视图
tags: [Objective-C]
---
本文以删除view上的所有子视图为例，重点讲的是NSSet和NSArray的makeObjectsPerformSelector方法和enumerator方法
## removeFromSuperview方法

首先来看看常用的removeFromSuperview方法，下面是苹果官方定义：

* Unlinks the receiver from its superview and its window,
and removes it from the responder chain.


* 译： 把接收者（当前view）从它的父视图移除，并删除它的响应链。 


调用removeFromSuperview方法会将当前视图从其父视图移除。（注意：只是将自己从俯视图移除，以前总是误以为将自己所有自视图从俯视图移除）所以用for...in...的方法，取到每一个subview，让他们执行removeFromSuperView就可以达到效果

```
for (UIView *view in [self.view subviews]) {

        [view removeFromSuperview];
    }
```



 注意：
 1. 永远不要在你的view的drawRect方法中调用removeFromSuperview；
 2. removeFromSuperview的实质并不是将这个视图从内存中移除,而是将一个视图从他的父视图上删除。计算机删除的本质是，标记删除，当你删除一个东西的时候，系统只是将这块内存做了一个标记，表示目前无人使用，但是之前视图的内存地址存在。所以如果想让视图不存在，需要在移除之后置为nil。

## makeObjectsPerformSelector

```
- (void)makeObjectsPerformSelector:(SEL)aSelector;  
- (void)makeObjectsPerformSelector:(SEL)aSelector withObject:(id)argument; 
```
介绍：让数组中的每个元素 都调用 aSelector  并把 withObject 后边的 argument 对象做为参数传给方法aSelector

一行搞定删除子视图

```
[self.view.sublayers makeObjectsPerformSelector:@selector(removeFromSuperview)];
```

带参数方法的使用：如果一个数组arry中存储了一组有hidden属性的对象（假设为view），需要将数组里所有对象的hide全部赋值为真，就可以这么写：

```
[arry makeObjectsPerformSelector:@selector(setHidden:) withObject:@YES];  
```
这么写就相当于arry数组里面的每一个对象都调用了setHidden方法，并且参数为YES，不用再遍历，一行代码搞定，是不是很方便。

但是若想设置为NO的话，则无效（亲测）。

```
[arry makeObjectsPerformSelector:@selector(setHidden:) withObject:@NO];  
```
这是因为YES和NO都为BOOL类型，设置为YES时，传递的为非0的指针，所以会设置 view.hidden = YES，但若设置为NO时，传递的仍为非0的指针，所以执行的结果仍是 view.hidden = YES。具体可看[这里](https://www.cnblogs.com/Apologize/p/5383652.html)。

但是可以用nil达到参数为NO的效果

```
[arry makeObjectsPerformSelector:@selector(setHidden:) withObject:nil];  
```
## enumerator ##

```
- (void)enumerateObjectsUsingBlock:block;
```
这个方法也是遍历数组，block里面的参数包括obj（运行的对象）、idx（下标）、stop（是否继续遍历的标志），*stop可以控制遍历何时停止，在需要停止时令*stop = YES即可（不要忘记前面的**），应该说，这个能满足基本所有的遍历需求了，有下标，有运行的对象，还有是否继续遍历的标志。

```
NSArray *xpArray = @[@"A", @"B", @"C", @"D", @"E"];
    
    [xpArray enumerateObjectsUsingBlock:^(id  _Nonnull obj, NSUInteger idx, BOOL * _Nonnull stop) {
        
        NSLog(@"%@", obj);
        
        if ([obj isEqualToString:@"C"]) {
            
            *stop = YES;
        }
    }];
```
不过反向遍历呢？苹果提供了另外一个方法：

```
[xpArray enumerateObjectsWithOptions:NSEnumerationReverse usingBlock:^(id obj, NSUInteger idx, BOOLBOOL *stop) {  
    NSLog(@"idx=%d, id=%@", idx, obj);  
}]; 
```
这个enumerateObjectsWithOptions:usingBlock:方法比前面那个方法多了一个枚举类型的参数NSEnumerationReverse，这个参数指定了遍历的顺序。

注意：这里要补充一点，这个方法是可以修改块签名，当我们已经明确集合中的元素类型时，可以把默认的签名id类型修改成已知类型，比如常见的NSString，这样既可以节省系统资源开销，也可以防止误向对象发送不存在的方法是引起的崩溃。
