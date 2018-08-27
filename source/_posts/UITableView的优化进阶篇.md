---
title: UITableView的优化进阶篇
date: 2018-2-16 17:23:33

description: 优化方案：异步渲染内容到图片和按需加载内容。

categories: [性能优化]
tags: [Objective-C]
toc: false 
---

UITableView是iOS中最常用的控件之一，毫不夸张的说基本每个app的都会用到UITableView，简单的使用可能不会有什么问题，但是利用UITableView进行复杂的布局（比如朋友圈的实现），就可能会出现一些性能问题，例如：滑动时卡顿，cell加载太慢等。

那么优化就势在必得。。。

## 在优化之前 

### 异步渲染内容到图片

这是一张典型的微博页面，这个页面若是采用普通的 View 控件拼凑的话，会需要多少个控件呢？我们这里简单划分一下。粗略划分了一下，我们大概需要 13 个 view 才能完成。


![](https://upload-images.jianshu.io/upload_images/656644-ea7cbf59b7b44594.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

对 TableView 的优化有时候可以直接考虑对 TableViewCell 的优化。对于复杂 View 的优化，首先考虑减少 View 的布局层级。我们将这个复杂的问题简单化，我们把 TableViewCell 按下图所示分割成三个部分，分别用红色，绿色，蓝色区分开来。 通过和实际的页面对比，我们可以看到红色部分的名字，日期，来源以及蓝色部分相对来说比较简单，布局变化比较小，所以我们可以考虑将这些内容全部绘制到一张图片上，来达到减少 View 的布局层级的目的。

![](https://upload-images.jianshu.io/upload_images/656644-bda7883871b7e11f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

我们排除其他干扰控件，使用 Xcode 来查看 TableViewCell 的布局层次，可以清晰的看到红色部分的名字，日期，来源以及整个蓝色蓝色部分都是直接绘制在图片上，图片使用一个 UIImageView 来承载。

![](https://upload-images.jianshu.io/upload_images/656644-9ee28421b61fc161.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

```
// 使用 CoreText 绘制
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
    CGRect rect = [_data[@"frame"] CGRectValue];
    UIGraphicsBeginImageContextWithOptions(rect.size, YES, 0);
    CGContextRef context = UIGraphicsGetCurrentContext();
    // 最外层的大框
    // [[UIColor colorWithRed:250/255.0 green:250/255.0 blue:250/255.0 alpha:1] set];     
    [[UIColor redColor] set];
    CGContextFillRect(context, rect);
    if ([_data valueForKey:@"subData"]) {
        // 第二层框
        // [[UIColor colorWithRed:243/255.0 green:243/255.0 blue:243/255.0 alpha:1] set];
        [[UIColor greenColor] set];
        CGRect subFrame = [_data[@"subData"][@"frame"] CGRectValue];
        CGContextFillRect(context, subFrame);
        // 线
        [[UIColor colorWithRed:200/255.0 green:200/255.0 blue:200/255.0 alpha:1] set];
        CGContextFillRect(context, CGRectMake(0, subFrame.origin.y, rect.size.width, 0.5));
    }
    {
        // 名字
        float leftX = SIZE_GAP_LEFT+SIZE_AVATAR+SIZE_GAP_BIG;
        float x = leftX;
        float y = (SIZE_AVATAR-(SIZE_FONT_NAME+SIZE_FONT_SUBTITLE+6))/2-2+SIZE_GAP_TOP+SIZE_GAP_SMALL-5;
        [_data[@"name"] drawInContext:context withPosition:CGPointMake(x, y) andFont:FontWithSize(SIZE_FONT_NAME) andTextColor:[UIColor colorWithRed:106/255.0 green:140/255.0 blue:181/255.0 alpha:1] andHeight:rect.size.height];
        y += SIZE_FONT_NAME+5;
        float fromX = leftX;
        float size = [UIScreen screenWidth]-leftX;
        // 时间和来源
        NSString *from = [NSString stringWithFormat:@"%@  %@", _data[@"time"], _data[@"from"]];
        NSString *from = [NSString stringWithFormat:@"%@ %@", _data[@"time"], _data[@"from"]]; [from drawInContext:context withPosition:CGPointMake(fromX, y) andFont:FontWithSize(SIZE_FONT_SUBTITLE) andTextColor:[UIColor colorWithRed:178/255.0 green:178/255.0 blue:178/255.0 alpha:1] andHeight:rect.size.height andWidth:size];
    }
    {
        CGRect countRect = CGRectMake(0, rect.size.height-30, [UIScreen screenWidth], 30);
        // [[UIColor colorWithRed:250/255.0 green:250/255.0 blue:250/255.0 alpha:1] set];
        [[UIColor blueColor] set];
        CGContextFillRect(context, countRect);
        float alpha = 1;
        // 评论
        float x = [UIScreen screenWidth]-SIZE_GAP_LEFT-10;
        NSString *comments = _data[@"comments"];
        if (comments) {
            CGSize size = [comments sizeWithConstrainedToSize:CGSizeMake(CGFLOAT_MAX, CGFLOAT_MAX) fromFont:FontWithSize(SIZE_FONT_SUBTITLE) lineSpace:5];
            x -= size.width;
            [comments drawInContext:context withPosition:CGPointMake(x, 8+countRect.origin.y) andFont:FontWithSize(12) andTextColor:[UIColor colorWithRed:178/255.0 green:178/255.0 blue:178/255.0 alpha:1] andHeight:rect.size.height];
            // 图片画到 context
            [[UIImage imageNamed:@"t_comments.png"] drawInRect:CGRectMake(x-5, 10.5+countRect.origin.y, 10, 9) blendMode:kCGBlendModeNormal alpha:alpha]; commentsRect = CGRectMake(x-5, self.height-50, [UIScreen screenWidth]-x+5, 50); x -= 20;
         }
         // 转发
         NSString *reposts = _data[@"reposts"];
         if (reposts) {
             CGSize size = [reposts sizeWithConstrainedToSize:CGSizeMake(CGFLOAT_MAX, CGFLOAT_MAX) fromFont:FontWithSize(SIZE_FONT_SUBTITLE) lineSpace:5];
             x -= MAX(size.width, 5)+SIZE_GAP_BIG;
             [reposts drawInContext:context withPosition:CGPointMake(x, 8+countRect.origin.y) andFont:FontWithSize(12) andTextColor:[UIColor colorWithRed:178/255.0 green:178/255.0 blue:178/255.0 alpha:1] andHeight:rect.size.height];
             [[UIImage imageNamed:@"t_repost.png"] drawInRect:CGRectMake(x-5, 11+countRect.origin.y, 10, 9) blendMode:kCGBlendModeNormal alpha:alpha]; repostsRect = CGRectMake(x-5, self.height-50, commentsRect.origin.x-x, 50); 
             x -= 20;
         }
         // ...
         [@"•••" drawInContext:context withPosition:CGPointMake(SIZE_GAP_LEFT, 8+countRect.origin.y) andFont:FontWithSize(11) andTextColor:[UIColor colorWithRed:178/255.0 green:178/255.0 blue:178/255.0 alpha:.5] andHeight:rect.size.height];
         if ([_data valueForKey:@"subData"]) { 
             // 线 
             [[UIColor colorWithRed:200/255.0 green:200/255.0 blue:200/255.0 alpha:1] set]; CGContextFillRect(context, CGRectMake(0, rect.size.height-30.5, rect.size.width, .5));
          }
     }
     // 获取绘制的图片，然后切换到主线程设置图片
     UIImage *temp = UIGraphicsGetImageFromCurrentImageContext();
     UIGraphicsEndImageContext();
     dispatch_async(dispatch_get_main_queue(), ^{ 
         if (flag==drawColorFlag) { 
              postBGView.frame = rect; 
              postBGView.image = nil; 
              postBGView.image = temp; 
          } 
     }); 
});

```

在上面代码中，除了做到减少 View 的布局层级之外还使用了一个非常重要技术-异步渲染内容到图片。使用 dispatch_async 将绘制工作放到后台操作，减少主线程的计算工作量

```
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{});
```
使用 CGContextFillRect 用于填充 View 的背景颜色。

```
//  [[UIColor colorWithRed:250/255.0 green:250/255.0 blue:250/255.0 alpha:1] set];
[[UIColor redColor] set];
CGContextFillRect(context, rect);
```
利用 CoreText 来做文本排版，具体的 CoreText 实现细节可以参考 Demo 代码

```
{
        // 名字
        float leftX = SIZE_GAP_LEFT+SIZE_AVATAR+SIZE_GAP_BIG;
        float x = leftX;
        float y = (SIZE_AVATAR-(SIZE_FONT_NAME+SIZE_FONT_SUBTITLE+6))/2-2+SIZE_GAP_TOP+SIZE_GAP_SMALL-5;
        [_data[@"name"] drawInContext:context withPosition:CGPointMake(x, y) andFont:FontWithSize(SIZE_FONT_NAME) andTextColor:[UIColor colorWithRed:106/255.0 green:140/255.0 blue:181/255.0 alpha:1] andHeight:rect.size.height];
        y += SIZE_FONT_NAME+5;
        float fromX = leftX;
        float size = [UIScreen screenWidth]-leftX;
        // 时间和来源
        NSString *from = [NSString stringWithFormat:@"%@  %@", _data[@"time"], _data[@"from"]];
        NSString *from = [NSString stringWithFormat:@"%@ %@", _data[@"time"], _data[@"from"]]; [from drawInContext:context withPosition:CGPointMake(fromX, y) andFont:FontWithSize(SIZE_FONT_SUBTITLE) andTextColor:[UIColor colorWithRed:178/255.0 green:178/255.0 blue:178/255.0 alpha:1] andHeight:rect.size.height andWidth:size];
}
```
异步生成图片之后切换到主线程设置图片

```
dispatch_async(dispatch_get_main_queue(), ^{ 
         if (flag==drawColorFlag) { 
              postBGView.frame = rect; 
              postBGView.image = nil; 
              postBGView.image = temp; 
          } 
 }); 
```

处理好了减少 View 布局层级和异步绘制之后，我们还需要处理一个圆角头像的问题。圆角头像最简单的处理方法就是使用一张圆形镂空的图片来实现，不过这个实现方案有个缺陷就是对 View 的背景颜色有要求。这里采用的处理方案就是这个最简单的处理方法。

![](https://upload-images.jianshu.io/upload_images/656644-6a2080dcdb710dce.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/520)

处理好了背景问题，接下来时候看看微博正文的问题了。微博的正文放在 label 控件里面，而转发的微博详情内容放在 detailLabel 里面。这个 label 是自定义控件 VVeboLabel，里面的 - (void)setText:(NSString *)text 方法具体的实现方式也是采用 CoreText 异步绘制实现的。

```
//设置文本内容，将文本内容设置在单独的 View 上面
- (void)drawText{

   if (label==nil||detailLabel==nil) {
        [self addLabel];
   }
   label.frame = [_data[@"textRect"] CGRectValue];
   [label setText:_data[@"text"]];
   if ([_data valueForKey:@"subData"]) { 
      detailLabel.frame = [[_data valueForKey:@"subData"][@"textRect"] CGRectValue]; 
      [detailLabel setText:[_data valueForKey:@"subData"][@"text"]];
      detailLabel.hidden = NO;
   }
}
```

![](https://upload-images.jianshu.io/upload_images/656644-b5dd6c1ccbe395a1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

可能你会问了，使用 CoreText 异步绘制的文本内容如何设置监听事件呢？CoreText 又如何处理点击高亮问题呢？我们通过 - (void)touchesBegan:(NSSet *)touches withEvent:(UIEvent *)event 方法来获取用户的点击位置。将获取到的用户点击位置与事先保存文本位置比较，若是用户点击位置位于文本区域内，那么说明用户点击了文本。为了能够做出高亮效果，VVeboLabel 控件内部必须维护一个字段 highlighting 和一个用于显示高亮文本图片的 highlightImageView，当 highlighting == YES 的时候，异步绘制高亮文本内容生成图片并使用 highlightImageView 显示该图片，用于表示控件的高亮状态。

```
- (void)touchesBegan:(NSSet *)touches withEvent:(UIEvent *)event{ 

     CGPoint location = [[touches anyObject] locationInView:self]; 
     for (NSString *key in framesDict.allKeys) { 
          CGRect frame = [[framesDict valueForKey:key] CGRectValue]; 
          // 将获取到的用户点击位置与事先保存文本位置比较 
          if (CGRectContainsPoint(frame, location)) { 
               NSRange range = NSRangeFromString(key); 
               range = NSMakeRange(range.location, range.length-1); 
               currentRange = range; 
               [self highlightWord]; 
               // *** 省略代码 
          } 
    } 
}
```

```
//  VVeboLabel.m
// ... 省略代码
if (highlighting) { 
   if (highlighting) { 
      highlightImageView.image = nil; 
      if (highlightImageView.width!=screenShotimage.size.width) { 
          highlightImageView.width = screenShotimage.size.width; 
      } 
      if (highlightImageView.height!=screenShotimage.size.height) { 
          highlightImageView.height = screenShotimage.size.height;
      } 
      highlightImageView.image = screenShotimage; 
   } 
   
} else { 

  if ([temp isEqualToString:text]) { 
     if (labelImageView.width!=screenShotimage.size.width) { 
        labelImageView.width = screenShotimage.size.width; 
     } 
     if (labelImageView.height!=screenShotimage.size.height) { 
        labelImageView.height = screenShotimage.size.height; 
     } 
     highlightImageView.image = nil; 
     labelImageView.image = nil; 
     labelImageView.image = screenShotimage;
   } 
}
 
 // ... 省略代码
    
```

VVeboLabel 控件处理高亮情况的 View 结构层次如下图。

![](https://upload-images.jianshu.io/upload_images/656644-45d5d9f65a4b8da6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

## 按需加载内容

UITableView 的优化除了在 UITableViewCell 的绘制方面优化之后，还可以在加载数据方面优化，按需加载内容，避免加载暂时无用的数据，从而减少数据量，减少 UITableView 的绘制工作量,达到优化的目的

判断按需加载的 indexPaths , 如果目标行与当前行相差超过指定行数，只在目标滚动范围的前后指定3行加载。这样可以减少 UITableView 的绘制工作量

```
    - (void)scrollViewWillEndDragging:(UIScrollView *)scrollView withVelocity:(CGPoint)velocity targetContentOffset:(inout CGPoint *)targetContentOffset{

        NSIndexPath *ip = [self indexPathForRowAtPoint:CGPointMake(0, targetContentOffset->y)];

        NSIndexPath *cip = [[self indexPathsForVisibleRows] firstObject];

        NSInteger skipCount = 8;

         // 目标行与当前行相差超过指定行数

        if (labs(cip.row-ip.row)>skipCount) {

            // 目标位置的行

            NSArray *temp = [self indexPathsForRowsInRect:CGRectMake(0, targetContentOffset->y, self.width, self.height)];

            NSMutableArray *arr = [NSMutableArray arrayWithArray:temp];

            //  velocity.y0 上拉

            if (velocity.y<0) {

                NSIndexPath *indexPath = [temp lastObject];

                if (indexPath.row+33) {

                    [arr addObject:[NSIndexPath indexPathForRow:indexPath.row-3 inSection:0]];

                    [arr addObject:[NSIndexPath indexPathForRow:indexPath.row-2 inSection:0]];

                    [arr addObject:[NSIndexPath indexPathForRow:indexPath.row-1 inSection:0]];
                }
            }
            [needLoadArr addObjectsFromArray:arr];
        }
    }
```

当 UITableView 开始绘制 Cell 的时候，若是 indexpath 包含在按需绘制的 needLoadArr 数组里面，那么就异步绘制该 Cell ，如果没有则跳过该 Cell 。

```
    - (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath{

        VVeboTableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:@"cell"];

        if (cell==nil) {

            cell = [[VVeboTableViewCell alloc] initWithStyle:UITableViewCellStyleDefault

                                             reuseIdentifier:@"cell"];

        }

        // 绘制 Cell
        [self drawCell:cell withIndexPath:indexPath];

        return cell;

    }

    // 按需绘制 Cell
    - (void)drawCell:(VVeboTableViewCell *)cell withIndexPath:(NSIndexPath *)indexPath{

        NSDictionary *data = [datas objectAtIndex:indexPath.row];

        cell.selectionStyle = UITableViewCellSelectionStyleNone;

        [cell clear];

        cell.data = data;

        // 按需绘制，只要在 needLoadArr 里面的 indexPath 才需要绘制 Cell

        if (needLoadArr.count>0&&[needLoadArr indexOfObject:indexPath]==NSNotFound) {

            [cell clear];

            return;
        }

        if (scrollToToping) {

            return;
        }
        [cell draw];
   }
```

这篇博客文章主要是学习 VVebo 的 UITableView 优化技巧，VVebo 的作者将 VVeboTableViewDemo开源在 GitHub，大家可以查阅代码，感谢作者。这个方案也有存在一个不足，在 TableView 快速滑动的时候，页面会出现空白。

### 文章链接

[UITableView优化](https://www.jianshu.com/p/2d077da3af94)



