---
title: Block(二)--_ _block 修饰符
date: 2017-07-21 01:01:18

description: 带你走进 _ _block 修饰符、使用场景、机制原理

categories: Block
tags: [Objective-C]
---

*******
[Block(一) -- Block 本质和截获变量](https://xiaopengmonsters.github.io/2018/07/16/Block--Block%20%E6%9C%AC%E8%B4%A8%E5%92%8C%E6%88%AA%E8%8E%B7%E5%8F%98%E9%87%8F/)
[Block(二) -- _ _block 修饰符](https://xiaopengmonsters.github.io/2018/07/21/Block--_%20_block%20%E4%BF%AE%E9%A5%B0%E7%AC%A6/)
[Block(三) -- Block 的内存管理](https://xiaopengmonsters.github.io/2018/08/06/Block--Block%20%E7%9A%84%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86/)
[Block(四) -- Block 的循环引用](https://xiaopengmonsters.github.io/2018/06/05/Block--Block%20%E7%9A%84%E5%BE%AA%E7%8E%AF%E5%BC%95%E7%94%A8/)
******

### 什么场景下需要使用 _ _block 修饰符

![](/img/__block修饰符使用场景.png)

### 思考两个题先

![](/img/笔试题的坑1.png)

![](/img/笔试题的坑2.png)

### 对变量进行赋值时，_ _block 具体使用特点

![](/img/__block具体使用特点.png)

因为 block 的截获变量特性，对于全局变量和静态全局变量不涉及截获操作，那么就是对全局变量和静态全局变量直接使用的，所以对其值进行操作的时候是不需要 杠杠block 修饰的
对于静态局部变量是通过指针形式来使用操作对应的变量的，那实际上操作的就是 block 外部的变量，所以说这种场景下也是不需要 _ _block 修饰的

### _ _Block 笔试题

![](/img/__Block笔试题.png)

### _ _block 修饰符的机制和原理：

**_ _block 修饰的变量变成了对象**

![](/img/变量变成了对象.png)

加了 杠杠block 修饰符之后，在编译后，这个整型变量最终就变成了一个结构体
结构体中有 void* 类型的 isa 指针，所以可以收 杠杠block 修饰符修饰的变量最终变成了对象
之后有一个指向同类型的指针 _ _forwarding
最后有一个成员变量 ，就是在函数当中所定义的局部变量

比如说在 Block 定义后面对局部变量进行 4 的赋值操作，实际上这个变量已经变成了一个对象，是通过这个对象当中的 _ _forwarding 指针去找到对应的一个对象，然后对其成员变量进行值的赋值

在方法当中对局部变量的赋值就改写成了对于他的对象通过 _ _forwarding 指针来修改对应的变量值

**栈上的 _ _forwarding 指向它自身的**

栈上的 block 进行局部变量的赋值，实际上 杠杠forwarding 指针指向的仍是自己所以修改的变量就是栈上 _ _block 它内部的一个成员值

### 留一个思考问题

![](/img/杠杠forwarding用来干什么的.png)


思考是很有意思的事 ~ ~ ~

_ _forwarding 的内容详见[Block(三) -- Block 的内存管理](https://xiaopengmonsters.github.io/2018/08/06/Block--Block%20%E7%9A%84%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86/)