---
title: UITableView的优化
date: 2017-12-26 17:23:33

description: UITableView是iOS中最常用的控件之一，毫不夸张的说基本每个app的都会用到UITableView，简单的使用可能不会有什么问题，但是利用UITableView进行复杂的布局（比如朋友圈的实现），就可能会出现一些性能问题，例如：滑动时卡顿，cell加载太慢等。

categories: [性能]
tags: [Objective-C]
toc: false 
---

UITableView是iOS中最常用的控件之一，毫不夸张的说基本每个app的都会用到UITableView，简单的使用可能不会有什么问题，但是利用UITableView进行复杂的布局（比如朋友圈的实现），就可能会出现一些性能问题，例如：滑动时卡顿，cell加载太慢等。

那么优化就势在必得。。。

## 在优化之前 

### UITableView的cell重用机制

UITableView最核心的思想就是UITableViewCell的重用机制。重用机制实现了数据和显示的分离,并不为每个数据创建一个UITableViewCell。

简单的理解就是：UITableView只会创建一屏幕（或一屏幕多一点）的UITableViewCell，其他都是从中取出来重用的。每当Cell滑出屏幕时，就会放入到一个集合（或数组）中（这里就相当于一个重用池），当要显示某一位置的Cell时，会先去集合（或数组）中取，如果有，就直接拿来显示；如果没有，才会创建。这样做的好处可想而知，极大的减少了内存的开销。

这种机制下系统默认有一个可变数组`NSMutableArray *visiableCells`，用来保存当前显示的cell。一个可变字典`NSMutableDictnery *reusableTableCells`，用来保存可重复利用的cell。（之所以用字典是因为可重用的cell有不止一种样式,我们需要根据它的reuseIdentifier,也就是所谓的重用标示符来查找是否有可重用的该样式的cell）。


### UITableView的代理方法

 必须实现的两个数据源回调方法:

```
/** 返回每个分区的行数 */
- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section;

/** 返回每一行的cell */
- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath;

```

调用最频繁的两个回调方法：

```
/** 返回cell的高度 */
- (CGFloat)tableView:(UITableView *)tableView heightForRowAtIndexPath:(NSIndexPath *)indexPath;

/** 返回每一行的cell */
- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath;

```
UITableView是继承自UIScrollView的，需要先确定它的contentSize及每个Cell的位置，然后才会把重用的Cell放置到对应的位置。所以事实上，UITableView的回调顺序是先多次调用tableView:heightForRowAtIndexPath:以确定contentSize及Cell的位置，然后才会调用tableView:cellForRowAtIndexPath:，从而来显示在当前屏幕的Cell。

因此，优化UITableView的首要任务就是优化上面这两个回调方法。

## 优化

### 1. Cell重用

**数据源优化方案：**首先创建一个静态变量reuseID（代理方法返回Cell会调用很多次，防止重复创建，static保证只会被创建一次，提高性能），然后，从缓存池中取相应identifier的Cell并更新数据，如果没有，才开始alloc新的Cell，并用identifier标识Cell。每个Cell都会注册一个identifier（重用标识符）放入缓存池，当需要调用的时候就直接从缓存池里找对应的id，当不需要时就放入缓存池等待调用。（移出屏幕的Cell才会放入缓存池中，并不会被release）所以在数据源方法中做出如下优化：

```
//调用次数太多，static 保证只创建一次reuseID，提高性能
static NSString *reuseID = “reuseCellID”;
```
```
//缓存池中取已经创建的cell
UITableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:reuseID];
```


**缓存池的实现：**当Cell要alloc时，UITableView会在堆中开辟一段内存以供Cell缓存之用。Cell的重用通过identifier标识不同类型的Cell，由此可以推断出，缓存池外层可能是一个可变字典，通过key来取出内部的Cell，而缓存池为存储不同高度、不同类型（包含图片、Label等）的Cell，可以推断出缓存池的字典内部可能是一个可变数组，用来存放不同类型的Cell，缓存池中只会保存已经被移出屏幕的不同类型的Cell。

### 2. 提前计算并缓存Cell的高度

调用最频繁的两个方法，上面已经优化了数据源方法，这里就是优化tableView:heightForRowAtIndexPath:方法了。

* UITableView在不设UITableViewCell的预估行高的情况下，会优先调用tableView:heightForRowAtIndexPath:方法，获取每个Cell的即将显示的高度，从而确定UITableView的布局，实际就是要获取contentSize（UITableView继承自UIScrollView,只有获取滚动区域，才能实现滚动）,然后才调用tableView:cellForRowAtIndexPath,获取每个Cell，进行赋值。

* 滚动时，每当cell进入凭虚都会计算高度，提前估算高度告诉编译器，编译器知道高度后，紧接着就会创建cell，这时再调用高度的具体计算方法，这样可以方式浪费时间去计算显示以外的cell

**方案：**创建一个frame模型，在刚获取到数据的时候，提前计算每个Cell的高度，之后每次返回缓存高度，可以避免在滑动时实时计算高度。

**示例：**如下面这个是我以前开发的朋友圈，先将获取到的数据转化为模型，然后将模型数组转换为frame模型数据，提前计算好所有的frame（包括cell高度和cell所有子控件frame）

```
/**
 *  根据纳信模型数组 转成 纳信frame模型数据
 *
 *  @param statuses 纳信模型数组
 *
 */
- (NSArray *)statusFramesWithStatuses:(NSArray *)statuses
{
    NSMutableArray *frames = [NSMutableArray array];
    for (NXStatusModel *status in statuses) {
        NXStatusFrame *frame = [[NXStatusFrame alloc] init];
        // 传递纳信模型数据，计算所有子控件的frame
        frame.isMask = NO;
        frame.status = status;
        [frames addObject:frame];
    }
    return frames;
}
```

### 3. 避免cell的重新布局

cell的布局填充等操作 比较耗时，一般创建时就布局好，
如可以将cell单独放到一个自定义类，初始化时就布局好。

* 尽可能的将 相同内容的抽取到一种样式Cell中，前面已经提到了Cell的重用机制，这样就能保证UITbaleView要显示多少内容，真正创建出的Cell可能只比屏幕显示的Cell多一点。虽然Cell的’体积’可能会大点，但是因为Cell的数量不会很多，完全可以接受的。

* 只定义一种Cell，那该如何显示不同类型的内容呢？答案就是，把所有不同类型的view都定义好，放在cell里面，通过hidden显示、隐藏，来显示不同类型的内容。毕竟，在用户快速滑动中，只是单纯的显示、隐藏subview比实时创建要快得多。

### 4. 滑动时，按需加载

开发的过程中，自定义Cell的种类千奇百怪，但Cell本来就是用来显示数据的，不说100%带有图片，也差不多，这个时候就要考虑，下滑的过程中可能会有点卡顿，尤其网络不好的时候，异步加载图片是个程序员都会想到，但是如果给每个循环对象都加上异步加载，开启的线程太多，一样会卡顿，我记得好像线程条数一般3-5条，最多也就6条吧。这个时候利用UIScrollViewDelegate两个代理方法就能很好地解决这个问题。

```
- (void)scrollViewDidEndDragging:(UIScrollView *)scrollView willDecelerate:(BOOL)decelerate
- (void)scrollViewDidEndDecelerating:(UIScrollView *)scrollView
```

思想就是识别UITableView禁止或者减速滑动结束的时候，进行异步加载图片，快滑动过程中，只加载目标范围内的Cell，这样按需加载，极大的提高流畅度。而SDWebImage可以实现异步加载，与这条性能配合就完美了，尤其是大量图片展示的时候。而且也不用担心图片缓存会造成内存警告的问题。

```
//获取可见部分的Cell
NSArray *visiblePaths = [self.tableView indexPathsForVisibleRows];
        for (NSIndexPath *indexPath in visiblePaths)
        {
        //获取的dataSource里面的对象，并且判断加载完成的不需要再次异步加载
             <code>
        }
```

记得在记得在“tableView:cellForRowAtIndexPath:”方法中加入判断：

```
// tableView 停止滑动的时候异步加载图片
- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath

         if (self.tableView.dragging == NO && self.tableView.decelerating == NO)
            {
               //开始异步加载图片
                <code>
            }
```
    
### 5. 渲染

不要使用ClearColor，无背景色，透明度也不要设置为0，渲染耗时比较长

**1、**减少subviews的个数和层级
 
 子控件的层级越深，渲染到屏幕上所需要的计算量就越大；如多用drawRect绘制元素，替代用view显示

**2、**少用subviews的透明图层

 对于不透明的View，设置opaque为YES，这样在绘制该View时，就不需要考虑被View覆盖的其他内容（尽量设置Cell的view为opaque，避免GPU对Cell下面的内容也进行绘制）

**3、**避免CALayer特效（shadowPath）

给Cell中View加阴影会引起性能问题，如下面代码会导致滚动时有明显的卡顿：

```
view.layer.shadowColor = color.CGColor;
view.layer.shadowOffset = offset;
view.layer.shadowOpacity = 1;
view.layer.shadowRadius = radius;
```

### 6. 使用局部更新
如果只是更新某组的话，使用reloadSection进行局部更新


## 总结

[这里](http://www.cocoachina.com/ios/20150602/11968.html)有位大佬，可以去看看哦

 UITableView的优化主要从三个方面入手:

* 提前计算并缓存好高度（布局），因为heightForRowAtIndexPath:是调用最频繁的方法；
* 滑动时按需加载，防止卡顿，这个我也认为是很有必要做的性能优化，配合SDWebImage
* 异步绘制，遇到复杂界面，遇到性能瓶颈时，可能就是突破口（如题，遇到复杂的界面，可以从这入手）

除了上面最主要的三个方面外，还有很多熟知的优化点：

* 正确使用reuseIdentifier来重用Cells
* 尽量使所有的view opaque，包括Cell自身
* 尽量少用或不用透明图层
* 如果Cell内现实的内容来自web，使用异步加载，缓存请求结果
* 减少subviews的数量
* 在heightForRowAtIndexPath:中尽量不使用cellForRowAtIndexPath:，如果你需要用到它，只用一次然后缓存结果
* 尽量少用addView给Cell动态添加View，可以初始化时就添加，然后通过hide来控制是否显示

### 文章链接

[优化UITableViewCell高度计算的那些事](http://blog.sunnyxx.com/2015/05/17/cell-height-calculation/)

[UITableView优化技巧](http://www.cocoachina.com/ios/20150602/11968.html)

[UITableView性能优化](http://blog.csdn.net/u011452278/article/details/60961350)


