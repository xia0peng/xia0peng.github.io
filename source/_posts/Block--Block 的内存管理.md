---
title: Block(三)--Block 的内存管理
date: 2017-08-29 01:01:18

description: Block 的内存管理

categories: Block
tags: [Objective-C]
---

*******
[Block(一) -- Block 本质和截获变量](https://xiaopengmonsters.github.io/2018/07/16/Block--Block%20%E6%9C%AC%E8%B4%A8%E5%92%8C%E6%88%AA%E8%8E%B7%E5%8F%98%E9%87%8F/)
[Block(二) -- _ _block 修饰符](https://xiaopengmonsters.github.io/2018/07/21/Block--_%20_block%20%E4%BF%AE%E9%A5%B0%E7%AC%A6/)
[Block(三) -- Block 的内存管理](https://xiaopengmonsters.github.io/2018/08/06/Block--Block%20%E7%9A%84%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86/)
[Block(四) -- Block 的循环引用](https://xiaopengmonsters.github.io/2018/06/05/Block--Block%20%E7%9A%84%E5%BE%AA%E7%8E%AF%E5%BC%95%E7%94%A8/)
******

### Block 的类型

impl.isa 就是标识当前 block 是哪种类型的

![](/img/Block的类型.png)

### 不同 Block 的内存分配

![](/img/不同Block的内存分配.png)

### Block 的 copy 操作


![](/img/Block的copy操作.png)

比如声明一个对象的成员变量是一个 block，而在栈上面去创建这个 block 同时赋值给成员变量 block，如果成员变量 block 没有使用 copy 关键字的话，比如使用的是 assigin，那么当具体通过成员变量去访问对应 block 的话，可能就由于栈所对应的函数退出之后，在内存当中就销毁掉了，这个时候再继续访问会因为内存崩溃

### 栈上 Block 的销毁

![](/img/栈上Block的销毁.png)

比如说在栈上有一个 杠杠block 变量，同时也有一个 Block，在变量的作用域结束之后或者说栈上的函数退出之后，那么关于栈上的 杠杠block 变量以及 Block 都会随之而销魂
由于 杠杠block 修饰的变量已经变成了对象，所以在这里也提到了关于 _ _block 变量在变量作用域结束之后一个销魂的逻辑

### 栈上 Block 的 copy

![](/img/栈上Block的copy.png)

比方说在栈上有一个 Block，Block 当中它使用到了一个 杠杠block 变量（可能是我们对这个变量赋值了，所以声明为 杠杠block 类型），当对栈上的 Block 进行 copy 操作之后，会在堆上产生一个和原来栈上面的一模一样的对应的 Block 和对应的 _ _block 变量，但是他们是分占两块内存空间，左侧是在栈上，右侧是在堆上，那么随着变量作用域的结束，栈上的 Block 会随之而销毁


思考：当对栈上的 Block 进行 copy 操作之后，在 MRC 环境下，是否会引起内存泄露？
答案：是肯定的，假如进行了 copy 操作之后同时堆上的 Block 没有额外其他成员变量指向他，那么就和 alloc 出一个对象没有调用对应的 release 的效果是一样的，产生内存泄露


### 栈上 _ _block 变量的 copy

![](/img/栈上杠杠block变量的copy.png)

比如说在栈上有一个  **_ _block** 变量，在这个  **_ _block** 变量中有一个  **_ _forwarding** 指针，在上一节讲过，在栈上 **_ _forwarding** 指针是指向它自身的
在对 **_ _block** 变量进行 copy 操作之后，会在堆上面产生一个 **_ _block** 变量和 栈上的是完全一致的，但是是两块内存空间

此时还有额外的变化在里面，可以回答 **_ _forwarding** 这个指针所存在的意义：
比方说对栈上的 **_ _block** 进行 copy 操作之后，栈上的  **_ _forwarding** 指针实际上指向的是堆上面的  **_ _block** 变量，而堆上面的  **_ _forwarding** 指针指向是其自身

所以说在对 multiplier 这个整型值进行值的改变的时候使用的其实转换出来的都是同一行代码，而对不论是栈上还是堆上的 Block 操作都是生效的，来具体解释一下：
比如说现在在栈上面去对某一个  **_ _block** 变量进行修改，这个时候如果对栈上的  **_ _block** 变量已经做过 copy 操作之后，实际上修改的不是栈上的  **_ _block** 变量对应的值，而是通过栈上  **_ _block** 变量里面的  **_ _forwarding** 指针找到堆上面的  **_ _block** 变量，然后对堆上面对应的变量 （multiplier ）值进行复制
那么同样的如果说 **_ _block** 变量由于被成员变量的一个 Block 所持有的话，当我们在另个一个方法或者说其他的地方去调用这个对应的 **_ _block** 变量的修改的情况下，那实际上是通过自身的 **_ _forwarding** 指向来修改的

经过编译器编译，无论这行代码出现在栈上的调用还是出现在堆上的调用，实际上最终所修改的值都是对堆上的 **_ _block** 变量进行修改的，说的这个大前提是栈上的 **_ _block** 变量经过了 copy 在堆上面产生了一致的 **_ _block** 变量
如果说对栈上的 **_ _block** 变量没有做 copy 操作，那么这行代码所修改的就是栈上的 **_ _block** 变量

### _ _forwarding 总结

![](/img/forwarding总结.png)

在左侧有一个代码段，声明了一个 **_ _block** 修饰的整型变量，初始化值是10
然后定义了一个 **_blk** 这么一个当前对象的成员变量形式的 Block，然后在栈上进行创建
之后在修改 multiplier 的值为 6
调用一个执行 Block 的一个同一个实例的另一个方法

讲解：在栈上创建了这个局部变量，通过 **_ _block** 修饰符修饰之后，这个局部变量就变成了一个对象，所以说这一步 multiplier = 6 的赋值实际上不是对变量进行赋值，而是通过 multiplier 这个对象的 **_ _forwording** 指针对其成员变量 multiplier 进行赋值
然后 **_blk** 这个变量实际上是某一对象的成员变量，当对它进行赋值操作的时候，会对它进行一个 copy，这个 block 就会跑到堆上面去或者说在堆上面有一个副本，然后再进行Block的执行

如果说这个 **_blk** 没进行 copy 操作，那么修改的就是栈上 **_ _block** 变量，如果对这个 **_blk** 进行了 copy 操作，这个行代码 multiplier = 6 所代表的含义就是通过栈上的 multiplier 的 **_ _forwording** 指针找到堆上面所对应的 multiplier 的copy 副本，然后对其副本进行值的修改改成6

右边调用了堆上的 block ，入参是 4，在调用的时候，在 Block 的执行体中所使用到的 multiplier **_ _block** 变量实际上使用的是堆上面的 **_block** 变量，在这里经过copy之后是对堆上面变量的修改，所以结构是4*6=24

### _ _forwarding 存在的意义

* **不论在任何内存位置，都可以顺利的访问同一个 _ _blcok 变量**

如果没有对 **_ _block** 变量进行 copy 的话，那么实际上操作的就是栈上的 **_ _block** 变量
如果发生了 copy 之后，无论是在栈上还是在堆上，对 **_ _block** 的修改或者说赋值操作都是操作的堆上面的 **_ _block** 变量，同时栈上关于 **_ _block** 变量的使用也都是使用的堆上的 **_ _block** 变量