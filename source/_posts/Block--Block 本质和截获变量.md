---
title: Block(一)--Block 本质和截获变量
date: 2017-07-16 01:01:18

description: 更深的理解 Block

categories: Block
tags: [Objective-C]
---

*******
[Block(一) -- Block 本质和截获变量](https://xiaopengmonsters.github.io/2018/07/16/Block--Block%20%E6%9C%AC%E8%B4%A8%E5%92%8C%E6%88%AA%E8%8E%B7%E5%8F%98%E9%87%8F/)
[Block(二) -- _ _block 修饰符](https://xiaopengmonsters.github.io/2018/07/21/Block--_%20_block%20%E4%BF%AE%E9%A5%B0%E7%AC%A6/)
[Block(三) -- Block 的内存管理](https://xiaopengmonsters.github.io/2018/08/06/Block--Block%20%E7%9A%84%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86/)
[Block(四) -- Block 的循环引用](https://xiaopengmonsters.github.io/2018/06/05/Block--Block%20%E7%9A%84%E5%BE%AA%E7%8E%AF%E5%BC%95%E7%94%A8/)
******

## Block 本质

![](/img/Block本质.png)

Block 实际上就是一个对象（结构体中有 isa 指针），**这个对象封装了函数，以及函数执行的上下文**

Block 在编译器编译之后会转换成：

![](/img/Block编译1.png)

![](/img/Block编译2.png)

Block 在编译器编译之后，是一个高亮的结构体，第一参数是无类型的函数指针，第二参数是 Block 相关描述的结构，后面的参数是 Block 要使用的，比如这里是传进来 Block 的局部变量

在这个高亮的结构体中有一个 impl 的结构体，在这个结构体中4个成员变量
第一个是 void 类型的 isa 指针，所以可以理解 block 是一个对象
还有一个关键的对象 FuncPtr，是无类型的函数指针，在 Block 的花括号中所定义的执行体，最终就会产生这样一个函数，然后 Block 同过函数指针来指向对应的函数实现

### 什么是 Block 调用？

* Block 调用即是**函数的调用**

首先对 block 进行强制类型转换，转后转后取它当中的成员变量 FuncPtr ，FuncPtr 是位于 Block 的结构体当中，通过函数指针取到对应的函数执行体，然后把对应的参数传进去

## Block 截获变量

### 思考一道题先

![](/img/滴滴笔试真题Block.png)

阅读一下下这个题：
首先定义了一个 int 类型的局部变量值为 6
之后定义了一个入参和返回值均为 int 类型的 Block，它的表达式是把入参和局不变的乘积作为返回值
之后又把局部变量的值修改为 4
然后在控制台打印输入参数为 2 的 Block 调用结果

**没错 是12，不小心被你看见了，那为什么是12呢，思考一下**

关于截获变量就涉及被截获变量的类型，针对不同类型的变量，Block 对其截获的特点也是不一样的

思考：关于 Block 的截获特性你是否有了解？  


### 特性

局部变量（基本数据类型、对象类型）
静态局部变量
全局变量
静态全局变量

![](/img/Block截获变量特性.png)

为什么是12：因为这是一个基本数据类型变量，在定义的时候就以值的方式传递到了 Block 所对应的结构体当中，而具体的函数调用是使用 Block 对应结构体当中的变量，所以输出的值是 12

**理解以指针形式截获静态变量：**

![](/img/指针形式截获静态变量.png)

因为在 Block 结构体当中是传递进或者说保存了静态局部变量的指针，那在具体调用的时候，通过这个指针来找到对应静态局部变量的值，静态局部变量被修改成4了，所以结果是8


**思考：如果在 Block 当中想要对局部变量进行值的修改，怎么做 **


用 **_ _block** 实现 
详见[Block(二) -- _ _block 修饰符](https://xiaopengmonsters.github.io/2018/07/21/Block--_%20_block%20%E4%BF%AE%E9%A5%B0%E7%AC%A6/)