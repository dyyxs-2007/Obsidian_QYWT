###### <center><font color=green face="楷体">iOS——无限轮播图内容整理</font></center>

# 无限轮播图是什么

- 轮播图在 App 里随处可见，比如首页的 Banner，点击可以跳转，会自动滚动切换，还能循环播放。听起来好像很复杂，但其实核心原理非常简单，理解了之后实现起来也就那几步

- 它的底层是一个 `UIScrollView`，把多张图片横向排列在里面，每次滚动一个屏幕宽度就切换到下一张。关键问题在于：`ScrollView` 本身是有终点的，滚到最后一张就没了，怎么实现"无限"循环？

- 答案是**复制首尾 + 边界瞬移**，这也是整个功能的核心思路，下面会详细说

# 无限循环的实现原理

## 为什么要复制首尾

我们有 A B C D 四张图，直接排列的话滚到 D 就到头了。要实现无限循环，在真实数据前后各复制一张：

```
真实数据：     [ A ]  [ B ]  [ C ]  [ D ]

实际布局：  [D复制]  [ A ]  [ B ]  [ C ]  [ D ]  [A复制]
             index0  index1  index2  index3  index4  index5
                      ↑
               默认从这里开始（真实第一张）
```

然后在 `scrollViewDidScroll:` 里监听偏移量，一旦检测到滑到了"复制页"，就立刻用 `animated:NO` 静默跳回对应的真实页。这个跳转是瞬间完成的，用户完全感知不到，看起来就像一直在循环滚动一样

## 边界检测逻辑

用 `scrollViewDidScroll:` 代理方法来做这件事。每次 ScrollView 滚动都会调这个方法，在里面判断当前偏移量是否越界，越界了就悄悄跳回去：

```objc
- (void)scrollViewDidScroll:(UIScrollView *)scrollView {
    CGFloat pageWidth = scrollView.bounds.size.width;
    CGFloat offsetX   = scrollView.contentOffset.x;
    NSInteger total   = self.realCount + 2; // 含首尾各一张复制

    // 滑到最左侧的复制页 → 跳回真实最后一张
    if (offsetX < pageWidth) {
        CGFloat realLastX = pageWidth * (total - 2);
        [scrollView setContentOffset:CGPointMake(realLastX, 0) animated:NO];
    }

    // 滑到最右侧的复制页 → 跳回真实第一张
    if (offsetX >= pageWidth * (total - 1)) {
        [scrollView setContentOffset:CGPointMake(pageWidth, 0) animated:NO];
    }
}
```

这里的 `animated:NO` 就是魔法所在，瞬移不会触发动画，用户看不出来

## 视图的布局结构

ScrollView 里不是直接放图片，而是先放一个 `contentView` 作为容器，再把所有图片加在 `contentView` 里：

```
UIScrollView
    └── UIView (contentView)         ← 承载所有图片，宽度由图片撑开
            ├── UIImageView (D复制)
            ├── UIImageView (A)
            ├── UIImageView (B)
            ├── UIImageView (C)
            ├── UIImageView (D)
            └── UIImageView (A复制)
```

约束上有一个关键点：`contentView` 的高度要锁死等于 `scrollView`，这样就只能横向滚动；宽度不设，让内部的图片从左到右一张张排列，自然把 `contentView` 撑开，`contentSize` 也就确定了

```objc
[self.contentView mas_makeConstraints:^(MASConstraintMaker *make) {
    make.edges.equalTo(self.scrollView);  // 四边贴合 scrollView
    make.height.equalTo(self.scrollView); // 锁死高度，禁止纵向滚动
    // 宽度不写，由子视图撑开
}];
```

图片的约束也有讲究，第一张贴 `contentView` 左边，后续每张贴在前一张的右边，**最后一张必须贴 `contentView` 右边**，否则 `contentView` 不知道自己有多宽：

```objc
// 第一张
make.left.equalTo(self.contentView);

// 中间每张
make.left.equalTo(previousView.mas_right);

// 最后一张（多加这一条）
make.right.equalTo(self.contentView);
```

## 初始定位

布局完成后默认是从 index=0（也就是复制的 D）开始显示的，需要手动跳到 index=1 才是真实第一张：

```objc
- (void)scrollToRealFirst {
    dispatch_async(dispatch_get_main_queue(), ^{
        CGFloat pageWidth = self.scrollView.bounds.size.width;
        [self.scrollView setContentOffset:CGPointMake(pageWidth, 0) animated:NO];
    });
}
```

这里用 `dispatch_async` 的原因是：`viewDidLoad` 时布局还没完成，直接取 `bounds.size.width` 可能拿到 0。把这段代码推到下一个 RunLoop，布局就已经算好了，能拿到正确的宽度

# NSTimer 计时器的添加方式

## 创建与启动

用 `NSTimer` 来定时触发翻页。调用 `scheduledTimerWithTimeInterval:` 方法，传入时间间隔、响应对象、触发的方法名，`repeats:YES` 表示会一直重复：

```objc
- (void)startAutoScroll {
    self.autoScrollTimer = [NSTimer scheduledTimerWithTimeInterval:2.5
                                                           target:self
                                                         selector:@selector(scrollToNext)
                                                         userInfo:nil
                                                          repeats:YES];
}
```

翻到下一页的逻辑很简单，就是在当前 `contentOffset.x` 的基础上加一个 `pageWidth`，边界处理交给 `scrollViewDidScroll:` 去做：

```objc
- (void)scrollToNext {
    CGFloat pageWidth = self.scrollView.bounds.size.width;
    CGFloat nextX = self.scrollView.contentOffset.x + pageWidth;
    [self.scrollView setContentOffset:CGPointMake(nextX, 0) animated:YES];
}
```

## 停止与销毁

停止 Timer 要调 `invalidate`，停完之后记得把属性置为 `nil`，这是个好习惯：

```objc
- (void)stopAutoScroll {
    [self.autoScrollTimer invalidate];
    self.autoScrollTimer = nil;
}
```

## 用户拖拽时暂停

用户手动滑动的时候，Timer 还在跑的话会和手势冲突。所以在用户开始拖拽时暂停，手指离开后恢复：

```objc
// 手指触碰 → 暂停
- (void)scrollViewWillBeginDragging:(UIScrollView *)scrollView {
    [self stopAutoScroll];
}

// 手指离开 → 恢复
- (void)scrollViewDidEndDragging:(UIScrollView *)scrollView
                   willDecelerate:(BOOL)decelerate {
    [self startAutoScroll];
}
```

## 页面消失时释放

页面退出时一定要把 Timer 停掉，不然 Timer 会一直强引用 `self`，导致 ViewController 无法释放，产生内存泄漏：

```objc
- (void)viewDidDisappear:(BOOL)animated {
    [super viewDidDisappear:animated];
    [self stopAutoScroll];
}
```

# 易错点总结

**1. 初始显示的是复制页而不是第一张**

原因是在 `viewDidLoad` 里直接取 `scrollView.bounds.size.width`，此时布局还没完成，拿到的是 0，`setContentOffset` 没有生效。套一层 `dispatch_async(dispatch_get_main_queue(), ^{ })` 推到下一帧就好了

**2. 滑到头尾卡住，不循环**

多半是忘了设 `self.scrollView.delegate = self`，或者 `@interface` 那里没有声明遵守 `<UIScrollViewDelegate>` 协议，导致 `scrollViewDidScroll:` 根本没有被调用

**3. contentView 宽度没被撑开，所有图片叠在一起**

最后一张图片忘了加 `make.right.equalTo(self.contentView)` 这条约束，系统不知道 `contentView` 有多宽，布局就乱了。这条约束是专门用来"收口"的，不能漏

**4. 页面退出后 Timer 还在偷偷跑**

没有在 `viewDidDisappear:` 里调 `stopAutoScroll`。Timer 持有 `self` 的强引用，不主动 `invalidate` 的话，即便页面退出了 ViewController 也不会被释放

**5. 忘了把 pagingEnabled 打开**

没有开 `pagingEnabled = YES` 的话，ScrollView 不会自动对齐到整页，滑动会停在奇怪的位置。开启之后每次滑动都会自动吸附到整数倍 pageWidth 的位置

> 一般来说，按这个顺序检查：先确认 `pagingEnabled` 和 `delegate` 有没有设，再看图片约束有没有"收口"，最后再排查 `dispatch_async` 和 Timer 的问题，基本可以覆盖大多数 bug