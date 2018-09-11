---
title: Block(三)--Block 的内存管理
date: 2018-08-06 01:01:18

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

## 先思考两个问题

**题1：**

![](/img/Block题1.png)

以下划线开头的 array 和以下划线开头的 strBlk 都是作为当前对象的两个成员变量，而 array 一般用 strong 关键字修饰，block 一般用 copy 修饰
在代码中首先对 array 进行创建，之后创建 strBlk ，在 block 的表达式当中，使用到了当前对象的 array 成员变量，之后再对这个 block 进行调用

思考：这段代码有什么问题？

答案：会产生循环引用，是自循环方式的循环引用

由于当前对象是通过 copy 属性关键字声明的 block，所以当前对象对于这个 block 是有一个强引用的，而 block 的表达式中又使用到了当前对象的 array 成员变量，在截获变量知识中，关于 block 中所使用到的对象类的局部变量或者说成员变量，会连同其属性关键字一块截获，而 array 属性关键字是通过 strong 来修饰的，所以在这个 block 中就有一个 strong 类型的指针指向原来的这个对象或者说当前对象，由此就产生了一个循环引用


![](/img/Block题1答案.png)

可以通过在当前栈上去声明或者说创建一个 _ _weak 所有权修饰符修饰的一个 weakArray 这么样的一个指针或者说变量来指向原对象的 array 成员变量，然后在这个 block 中使用创建的 weakArray ，由此就可以解除自循坏引用（是通过避免循环引用来解除的）


**题2：**

![](/img/Block题2.png)

在这个栈上面通过 _ _block 修饰符所修饰的变量来指向当前对象，同时当前对象的成员变量 block 在这里进行创建，block 的表达式当中有使用到 self 的一个 var 成员变量，在这个是使用的是 blockSelf 而不是直接使用当前对象，然后对其进行调用

答案：

* **在 MRC 下**，不会产生循环应用
* **在ARC下**，会产生循环引用，引起内存泄漏


![](/img/ARC下的循环引用.png)

比方说这个对象有一个成员变量 block，那么它持有 block，而这个 block 当中使用到了 _ _ block 修饰符所修饰的变量，所以 block 对象它持有了 _ _ block 变量，而 _ _block 变量它本身也是对原对象有一个强引用的持有的，因为 _ _ block 变量它的指向是原来的对象，这种循环引用叫大环引用

![](/img/ARC下的循环引用答案.png)

可以通过断环的方式解除

具体的改写方案：（ARC下的解决方案）

![](/img/ARC下的解决方案.png)

在 block 内部这个位置加上对 blockSelf 这个变量进行 nil 赋值操作就可以规避循环引用

当调用这个 block 之后就会断开刚才所说的这个环然后就可以都得到内存的释放和销毁

但是这种解决方案有一个弊端，就是说如果我们很长一段时间或者说永远都不会调用这个 block 的话，这个循环引用的环就会一直都存在

