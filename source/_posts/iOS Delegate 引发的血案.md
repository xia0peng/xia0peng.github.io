---
title: iOS Delegate 引发的血案
date: 2018-03-18 17:23:33

description: 今天在项目重构过程中遇到一个很神奇的 bug，和同组的小伙伴一起研究了好久终于排查出问题所在，当然大家应该已经猜到了这个问题是因为 Delegate 的使用所导致的。接下来我先简单的将这个问题描述一下

categories: 问题记录
tags: [Objective-C]
toc: false 
---

今天在项目重构过程中遇到一个很神奇的 bug，和同组的小伙伴一起研究了好久终于排查出问题所在，当然大家应该已经猜到了这个问题是因为 Delegate 的使用所导致的。接下来我先简单的将这个问题描述一下

## 问题描述 

![](/img/iOSDelegate引发的血案.png)

* 矩形框1里面的每个标签对应一个 tableView, 该 tableView 都是由同一个类实例化所得。
* 矩形框2是一个音频播放器，初始化是在 tableView 内的，UI在 tableViewCell 内实现的一个按钮。
* 点击矩形框2内的按钮的时候，tableViewCell 通过代理通知 tableView 执行播放器播放操作并把相关数据字典dic传递给tableView。
* tableView 接收到代理后执行播放器的播放代理，同时会用到cell传递回来的数据

操作描述

* 进入到上图所示界面，默认选中的是全部标签，直接点击其余任意一个标签，然后再切换回到全部标签
* 点击播放按钮，排查到 tableView 中 cell 的代理内数据dic值正常，播放器代理中数据dic值为空，导致无法播放

## 代码贴出来
```
YDLTalkTableView.m

#pragma mark - lify cycle
#pragma mark - DFPlayer为播放器
- (instancetype)initWithFrame:(CGRect)frame style:(UITableViewStyle)style withTab:(NSString *)tab{
    self = [super initWithFrame:frame style:style];
    if (self) {
        [DFPlayer shareInstance].category = DFPlayerAudioSessionCategorySoloAmbient;
        [DFPlayer shareInstance].playMode = DFPlayerModeOnlyOnce;
        [DFPlayer shareInstance].isObserveWWAN = NO;
        [DFPlayer shareInstance].dataSource = self;
        [DFPlayer shareInstance].delegate = self;
        [[DFPlayer shareInstance] df_audioPause];
        self.delegate = self;
        self.dataSource  = self;
        if (@available(iOS 11.0, *)) {
            UIScrollView.appearance.contentInsetAdjustmentBehavior = UIScrollViewContentInsetAdjustmentNever;
            self.estimatedRowHeight = 0;
            self.estimatedSectionHeaderHeight = 0;
            self.estimatedSectionFooterHeight = 0;
        } else {
            // Fallback on earlier versions
        }
        [self registerClass:[YDLTalkTableViewCell class] forCellReuseIdentifier:NSStringFromClass([YDLTalkTableViewCell class])];
        [self setSeparatorStyle:UITableViewCellSeparatorStyleNone];
        [self setBackgroundColor:kBackGroundColor];
        self.tableFooterView = [UIView new];
        __weak typeof(self) weakSelf = self;
        [self loadHeaderRefreshWithHeaderRefreshingBlock:^{
            [weakSelf onHeader];
        }];
        [self loadFooterRefreshWithFootRefreshingBlock:^{
            [weakSelf onFooter];
        }];
    }
    return self;
}

#pragma mark - cell的代理方法
-(void)bofangTalk:(NSDictionary*)dict{
    _dict = dict;//此处dict值为正常的
    [MBProgressHUD showError:_dict[@"expert_name"] toView:_viewController.view];
    [[DFPlayer shareInstance] df_reloadData];
}

#pragma mark - DFPLayer dataSource
- (NSArray<DFPlayerModel *> *)df_playerModelArray{
    NSLog(@"%@", _dict);//此处dict值为nil
    NSMutableArray *_df_ModelArray;
    return _df_ModelArray;
}

```
## 问题解决

我们都知道代理是属于一对一的关系，请看下面这段代码：

```
- (instancetype)initWithFrame:(CGRect)frame style:(UITableViewStyle)style withTab:(NSString *)tab{
    self = [super initWithFrame:frame style:style];
    if (self) {
        [DFPlayer shareInstance].dataSource = self;
        [DFPlayer shareInstance].delegate = self;
}
```

当我们进入这个页面的时候首先创建的是“全部”标签下对应的tableView,这个时候播放器的代理是“全部”标签对应的tableView，当我们点击任意其他一个标签的时候，播放器的代理就会变成该标签对应的tableView，然后等我们回到“全部”标签的时候，是不会调用上面方法的，所以这个时候播放器的代理其实还是我们上次点击的标签所对应的tableView。所以我们就可以理解为什么执行cell的delegate的时候dict的值是正常的，而到执行播放器代理的时候dict的值就变为了nil了。


为什么会出现这种情况呢，有些人可能会说了：我平时这么写也没发现有问题啊。这就是我们这个案例特殊的一点，因为我们的播放器是通过单例来创建的，也就是说全局只会存在着一个实例，那么他的代理也应该是只对应一个的，以最后一个设置的为准，如果在上面的方法中我们的播放器通过[[DFPlayer alloc] init]来创建的话，那就没有问题了

## 总结

遇到问题很兴奋，解决问题更兴奋。

首先遇到 bug 不要慌，是时候展现自己真正的技术了（简单的排查思路）

* 解决问题一定要思路清晰，先根据 bug 现象判断出问题的大体可能，然后验证你的假想（我的项目播放是利用代理来回传进行播放的，所以大致定位到 delegate 出现相关问题）
* 然后用自己的经验和掌握的技能（断点、排除法、逐步分析法等等）找到出现问题的具体代码行（一顿操作操作排查到 tableView 中 cell 的代理内数据dic值正常，播放器代理中数据dic值为空，导致无法播放）
* 最后分析导致问题的可能，逐步更正（这里是有必要联系场景分析的）

默认选中的是全部标签，直接点击其余任意一个标签，然后再切换回到全部标签出现问题，综上

* 你可能就会想到，标签切换回来的时候代理失效了（想的不错，当然需要一些时间）
* 思考代理为什么会失效？（找到关键了，让代理不失效就ok了）
* 思考怎么让代理不失效？（又找到关键了，发现在标签切换的时候代理指向别处了，结合代理一对一的特性）
* 思考怎么将代理重新指向当前控制器（哎呀，又想到关键了，在播放事件中重新改变代理的指向而不是之前的在初始化时指向代理）
* 问题解决，完美

以此篇记录关于 delegate（设置单例的代理）引发的 bug



