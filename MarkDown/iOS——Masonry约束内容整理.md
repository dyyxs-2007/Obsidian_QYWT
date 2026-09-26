###### <center><font color=green face="楷体">iOS——Masonry约束内容整理</font></center>

# Masonry约束是什么

- 对于iOS开发来说，视图的位置和大小怎么确定，作为初学者我们一开始接触到的是使用frame直接写死一个视图的坐标和大小，但直接写死带来的后果就是视图之间变得过于死板，一个窗口发生变化，为了匹配对应的变化，往往要改很多个视图的坐标和大小
- 所以我们接触到了 Auto Layout 这套机制来做这件事，但原生的写法十分繁琐，详情可以查看我的这篇文章[【iOS】 AutoLayout初步学习](https://blog.csdn.net/2505_93363502/article/details/160081039?spm=1001.2014.3001.5501) ，设置一个约束就要写一大串 `NSLayoutConstraint`，代码可读性很差,效率也不够高

- Masonry 解决了这个问题。它是对 Auto Layout 的一层封装，将又长又难读的代码，变成了简洁的点语法链式调用， 其实层还是在帮你生成 `NSLayoutConstraint`，只是写起来十分简洁

# Masonry约束与AutoLayout之间的关系

- Masonry 并不是抛弃了 Auto Layout的机制，而是完全对 Auto Layout 的封装。你用 Masonry 写的每一条约束，最终都会被翻译成系统的 `NSLayoutConstraint` 对象添加到视图上

- 除此之外，Masonry 还会自动把 `translatesAutoresizingMaskIntoConstraints` 设置为 `NO`，在正常的AutoLayout中需要手动关闭，但这里它会自动帮我们做这件事

# CocoaPods的引入方式

Masonry 通过 CocoaPods 安装(这里不多赘述)，在 Podfile 里加一行就行：

```
pod 'Masonry'
```

然后在需要使用的文件里导入头文件：

```objc
#import <Masonry/Masonry.h>
```

# Masonry约束的具体内容

## 三个核心方法

Masonry 提供了三个用来设置约束的方法，分别对应不同的使用场景：

```
mas_makeConstraints      第一次给视图添加约束时
mas_updateConstraints    更新某已有的约束，这样可以保持其他约束不变
mas_remakeConstraints    把旧的约束全部清掉，重写该视图的约束
```

- 实际应用中用得最多的是 `mas_makeConstraints`，在视图第一次布局的时候调用。
- 如果需要临时对视图约束进行修改，比如按钮点击后视图位置发生变化，就可以用 `mas_updateConstraints` 只改你需要改的那几条，或者用 `mas_remakeConstraints` 整体重写。

>  此外，调用这三个方法之前，视图必须已经被添加到父视图上，这也是使用约束的前提条件，否则没有对照

```objc
[self.view addSubview:redView]; // 先加进去

[redView mas_makeConstraints:^(MASConstraintMaker *make) {
    make.top.equalTo(self.view).offset(100);//这里的约束稍后会介绍
    make.left.equalTo(self.view).offset(20);
    make.width.mas_equalTo(200);
    make.height.mas_equalTo(50);
}];
```

## 基本结构与写法

Masonry 约束的基本结构长这样：make代表视图本身，先约束哪条边(top)，再说等于谁(self.view)，最后加上偏移量(100)

```objc
make.top.equalTo(self.view).offset(100);
// 解读：这个视图的 top，等于 self.view 的 top，再往下偏移 100pt
```

## 点语法用法

Masonry 的点语法是它最直观的地方，常用的属性和方法整理如下：

| 点语法 / 方法                     | 说明                                                         |
| --------------------------------- | ------------------------------------------------------------ |
| `.top` `.bottom` `.left` `.right` | 对应视图的上下左右四条边                                     |
| `.leading` `.trailing`            | 首边和尾边，推荐用这个替代 left/right，支持阿拉伯语等 RTL 布局 |
| `.width` `.height`                | 宽和高                                                       |
| `.centerX` `.centerY`             | 水平和垂直居中轴                                             |
| `.center`                         | 同时设置 centerX 和 centerY                                  |
| `.edges`                          | 同时设置 top + left + bottom + right                         |
| `.size`                           | 同时设置 width 和 height                                     |
| `.equalTo(view)`                  | 等于某个视图或 mas_ 属性，传的是对象                         |
| `.mas_equalTo(value)`             | 等于某个具体数值，传的是基本类型或结构体                     |
| `.offset(CGFloat)`                | 在约束基础上偏移，正负决定方向                               |
| `.insets(UIEdgeInsets)`           | 配合 `.edges` 做四边内缩                                     |
| `.multipliedBy(CGFloat)`          | 乘以系数，常用于按比例设宽高                                 |
| `.priority(MASLayoutPriority)`    | 设置这条约束的优先级                                         |

举几个常见写法：

```objc
// 贴满父视图
make.edges.equalTo(self.view);

// 四边各内缩 16pt
make.edges.equalTo(self.view).insets(UIEdgeInsetsMake(16, 16, 16, 16));

// 宽度是父视图的一半
make.width.equalTo(self.view).multipliedBy(0.5);

// 水平居中 + 固定宽高
make.centerX.equalTo(self.view);
make.width.height.mas_equalTo(100);
```

### 链式点语法

- 可能你注意到了这里的写法可以在make后面连续用`.`连接好几个边，这就是Masonry的链式点语法，**它支持把多个属性写到一个链上**就不用分开写好几行，我们以代码演示：

```objc
// 分开写
make.top.equalTo(self.view).offset(20);
make.left.equalTo(self.view).offset(20);

// 链式合并写
make.top.left.equalTo(self.view).offset(20);
```

- 这里的`.`单纯代表对哪个边约束，两种写法完全相同

> 需要注意的是，链式写法的 `offset` 或 `mas_equalTo` 会同时作用于链上所有属性。所以如果几个方向的偏移量不一样，就不能合并，还是得分开写。

## equalTo 与 mas_equalTo 的区别

这两个方法名字很像，但接受的参数类型不一样，也代表两种不同的用法

```objc
// equalTo：接视图对象或 mas_ 属性，  因为有对比，所以是针对其他视图的偏移
make.width.equalTo(otherView);          // 宽度和 otherView 相同
make.top.equalTo(viewA.mas_bottom);     // top 等于 viewA 的 bottom

// mas_equalTo：接基本数据类型或结构体， 用于写死高度，大小等等
make.width.mas_equalTo(100);
make.size.mas_equalTo(CGSizeMake(100, 50));
make.edges.mas_equalTo(UIEdgeInsetsMake(10, 10, 10, 10));
```

记忆方式很简单：带 `mas_` 前缀的就是接数字或结构体，没有前缀的就是接视图。

## offset方向规律

`offset` 的方向和坐标轴是一致的，但对于不同的边，正负产生的效果会不一样：

| 属性                | 正值效果               | 负值效果               |
| ------------------- | ---------------------- | ---------------------- |
| `.top.offset(n)`    | 向下移动（远离顶部）   | 向上移动               |
| `.bottom.offset(n)` | 向下撑出（超出父视图） | 向上收缩（常用 `-16`） |
| `.left.offset(n)`   | 向右移动（远离左边）   | 向左移动               |
| `.right.offset(n)`  | 向右撑出               | 向左收缩（常用 `-16`） |

- **只需要记住，offset的正数是向下或者向右， 负数与正数相反，类似于frame的坐标即可**

### 约束优先级

有时候同一个属性会有多条约束，这时候就需要用优先级来决定哪条先被满足。

Masonry 内置了几个常用的优先级常量：

```objc
MASLayoutPriorityRequired         // 1000，必须满足，默认值
MASLayoutPriorityDefaultHigh      // 750，高优先级
MASLayoutPriorityDefaultLow       // 250，低优先级
```

用法是在约束后面接上 `.priority()`：

```objc
make.height.mas_equalTo(200).priority(MASLayoutPriorityDefaultHigh);
make.height.greaterThanOrEqualTo(@44).priority(MASLayoutPriorityRequired);
```

优先级的使用场景：比如你希望一个视图高度正常情况下是 200，但最小不能低于 44（比如内容压缩的时候），就可以给这两条约束分别设不同的优先级，让系统在约束冲突时知道该优先保证哪一条。

## 对UIScrollView约束时的特殊处理

- ScrollView 是 Masonry 里最容易出问题的地方，原因在于 ScrollView 有一个 `contentSize`（内容区域的大小），它和 ScrollView 本身的 frame 是两回事。

- 直接把子视图约束到 ScrollView 上，系统搞不清楚 contentSize 是多少，结果就是 ScrollView 没法滚动。

- 所以我们需要加一个中间层 `contentView`， 然后把所有视图放在中间层上

```objc
UIScrollView *scrollView = [[UIScrollView alloc] init];
UIView *contentView = [[UIView alloc] init];

[self.view addSubview:scrollView];
[scrollView addSubview:contentView];

// scrollView 张开贴满屏幕
[scrollView mas_makeConstraints:^(MASConstraintMaker *make) {
    make.edges.equalTo(self.view);
}];

// contentView 的 edges 等于 scrollView，撑开 contentSize
// 同时锁定宽度等于 scrollView，这样只能纵向滚动，也可以锁定高度，这样就能横向滚动
[contentView mas_makeConstraints:^(MASConstraintMaker *make) {
    make.edges.equalTo(scrollView);
    make.width.equalTo(scrollView);
}];

// 最后一个子视图的 bottom 必须连到 contentView，系统才能知道contentView到底多高
//因为添加进入的视图有高度，而contentView并没有约束高度，所以你需要在这里对bottom相对的约束一下让系统可以根据视图的高度计算出contentView的大小
[lastSubview mas_makeConstraints:^(MASConstraintMaker *make) {
    make.bottom.equalTo(contentView).offset(-20);
}];
```

- 逻辑是这样的：contentView 的 edges 贴着 scrollView，这样 contentView 的大小就决定了 scrollView 的 contentSize；宽度锁死等于 scrollView 的宽，就只允许纵向滚动；最后一个子视图的 bottom 连到 contentView 的 bottom，contentView 的高度才能被内容撑开，scrollView 才能真正滚动起来

### 点语法速查表

常用的 mas_ 属性：当你在 `equalTo()` 里需要对齐另一个视图的某条边时，需要用 `mas_` 前缀属性来取到那条边。

| mas_ 属性                      | 说明                |
| ------------------------------ | ------------------- |
| `view.mas_top`                 | 上边                |
| `view.mas_bottom`              | 下边                |
| `view.mas_left`                | 左边                |
| `view.mas_right`               | 右边                |
| `view.mas_centerX`             | 水平中线            |
| `view.mas_centerY`             | 垂直中线            |
| `view.mas_width`               | 宽度                |
| `view.mas_height`              | 高度                |
| `view.mas_safeAreaLayoutGuide` | 安全区域（iOS 11+） |

典型用法：

```objc
// B 的顶部紧贴 A 的底部，间距 12pt
make.top.equalTo(viewA.mas_bottom).offset(12);

// 对齐安全区域顶部
make.top.equalTo(self.view.mas_safeAreaLayoutGuide);
```
