###### <center><font face="楷体" color=green>AutoLayout初步学习</font></center>
# AutoLayout的概念

## 自动约束的含义

- 早期iOS屏幕只有一种尺寸，直接用frame写死坐标就够用了。但现在屏幕大小五花八门，写死坐标换个设备就乱了。

- AutoLayout的思路是：不直接告诉视图"在哪里"，而是告诉它"和谁的关系是什么"。

**每条约束本质上是一个方程：**
```
item1.attribute = multiplier × item2.attribute + constant
```

- 比如"A的左边距离B的右边20单位"就是：

```
A.left = 1.0 × B.right + 20
```

系统收集所有约束，联立求解，算出每个视图的frame，就像是进行解方程，虽然每个边和父视图的关系加起来有很多，但只需要解出来宽高和左右上下坐标四个未知量，所以四个方程即可，**约束是规则，frame是结果。**

## 自动约束的优点

- 规则定好之后，父视图尺寸怎么变，子视图都会跟着重新计算，始终保持正确的相对关系。比如你写"距左边20"，父视图宽300时子视图有280的空间，父视图宽600时子视图自动有580的空间。这样就不用频繁的更改子视图在父视图中的布局


# 创建普通视图时

## 自动转化约束

用代码创建视图时，系统默认开启了AutoresizingMask的布局系统。简单理解就是：你可以给视图设置一些"弹性规则"，告诉它在父视图变化时如何伸缩。

- 例如
```objc
view.autoresizingMask = UIViewAutoresizingFlexibleWidth;
// 代表子视图随父视图的宽方向大小变化而伸缩
```

可设置的规则有这几种：

| 值 | 含义 |
|---|---|
| UIViewAutoresizingFlexibleWidth | 宽度跟随父视图伸缩 |
| UIViewAutoresizingFlexibleHeight | 高度跟随父视图伸缩 |
| UIViewAutoresizingFlexibleTopMargin | 上边距弹性变化 |
| UIViewAutoresizingFlexibleBottomMargin | 下边距弹性变化 |
| UIViewAutoresizingFlexibleLeftMargin | 左边距弹性变化 |
| UIViewAutoresizingFlexibleRightMargin | 右边距弹性变化 |
> 这里的弹性变化就是当父视图变化的时候，子视图的大小也会变化，并且“吃掉”变化的那部分而变多或变少
什么都不设置的话，默认行为就是frame完全写死，父视图怎么变它都不动。

- 这个的问题在于没办法描述复杂的相对关系。系统会自动把AutoresizingMask的规则**翻译成几条AutoLayout约束**。但这些约束是系统自动生成的，不受你控制，如果同时又手写了自己的约束，两组约束就会打架，布局就乱了
- 所以我们在手动写AutoLayout的时候需要先关闭这个方式系统生成的和我们写的冲突，后面会有介绍


# 坐标约束的条件

## 确定坐标需要的条件

AutoLayout的目标是推导出视图的四个值：x、y、width、height。只要这四个值都能推出来，约束怎么组合都行：

| 组合 | 是否合法 |
|---|---|
| 左 + 右 + 上 + 下（四边） | 宽高都能算出来，不需要再写宽高 |
| 左 + 上 + 宽 + 高 | 经典写法 |
| 左 + 上 + 右 + 高 | 宽能从左右推出来 |
| 左 + 上 + 宽 |  高不知道，缺约束 |
> 类似于三个方程解不出来四个未知量

- 约束给少了就会导致系统报warning，视图位置不确定，这时候需要添加条件

- 约束给多了或者矛盾了会让系统报error，需要根据需要修改约束

- 此外，AutoLayout只负责算出视图的矩形框，不管框里的内容。内容超出框之后：`clipsToBounds = NO`会溢出显示，`clipsToBounds = YES`会直接裁掉超出的部分。


# AutoLayout使用方法

## 关闭自动生成

- 刚刚提到过，用代码写AutoLayout时，第一件事是关掉系统的自动转换，不然系统自动生成的约束和你手写的会冲突：

```objc
view.translatesAutoresizingMaskIntoConstraints = NO;
```

这行写在`addSubview:`前后都可以，但必须在`activateConstraints:`之前

## 签名及其各个参数介绍

```objc
[NSLayoutConstraint constraintWithItem:item1        // 要约束的视图（子视图）
                             attribute:attr1        // 这个视图的哪个属性
                             relatedBy:relation     // 等于 / 大于等于 / 小于等于
                                toItem:item2        // 参照的视图（父视图）
                             attribute:attr2        // 参照视图的哪个属性
                            multiplier:1.0          // 倍数，通常写1.0
                              constant:20];         // 偏移量
```

对应回方程就是：`item1.attr1 = 1.0 × item2.attr2 + 20 // item1的attr1属性（可能是宽高等）是item2的相同属性加20个单位的结果`

如果是给视图设置固定宽高，不需要参照其他视图，`toItem`传`nil`，`attribute`传`NSLayoutAttributeNotAnAttribute`，表示方程右边没有参照物。

## attribute和relatedBy的类型介绍

常用的`NSLayoutAttribute`：

| 值 | 含义 |
|---|---|
| NSLayoutAttributeLeft / Right | 左边 / 右边 |
| NSLayoutAttributeTop / Bottom | 上边 / 下边 |
| NSLayoutAttributeCenterX / CenterY | 水平中心 / 垂直中心 |
| NSLayoutAttributeWidth / Height | 宽度 / 高度 |
| NSLayoutAttributeNotAnAttribute | 无参照，用于固定宽高 |

`NSLayoutRelation`的三个值：

| 值 | 含义 |
|---|---|
| NSLayoutRelationEqual | 等于 |
| NSLayoutRelationGreaterThanOrEqual | 大于等于 |
| NSLayoutRelationLessThanOrEqual | 小于等于 |

## 数组激活的写法

- 创建`NSLayoutConstraint`对象只是写好了规则，必须激活才会生效。有两种方式：

```objc
// 单条激活
[constraint setActive:YES];
> 这样写的话，就要对每一条约束都执行一遍setActive，比较麻烦

// 批量激活
[NSLayoutConstraint activateConstraints:@[c1, c2, c3, c4]];
```

`activateConstraints:`接收一个数组，对里面每条约束逐个激活。数组只是个容器，不同视图的约束也可以放在同一个数组里一起激活，只要约束之间不冲突就行，这个方法会对数组里的约束一一进行运行激活

- 注意`activateConstraints:`必须在`addSubview:`之后调用，不然没办法得到两个视图之间的关系

## 实例代码

在屏幕中间创建一个蓝色方框，内部居中显示一行文字：

```objc
- (void)viewDidLoad {
    [super viewDidLoad];
    
    // 创建蓝色方框
    UIView *blueView = [[UIView alloc] init];
    blueView.backgroundColor = [UIColor blueColor];
    blueView.translatesAutoresizingMaskIntoConstraints = NO;//关闭自动生成
    [self.view addSubview:blueView];
    
    [NSLayoutConstraint activateConstraints:@[//这里是字面量创建数组的写法
        
        // 宽200
        [NSLayoutConstraint constraintWithItem:blueView
                                     attribute:NSLayoutAttributeWidth//宽
                                     relatedBy:NSLayoutRelationEqual
                                        toItem:nil
                                     attribute:NSLayoutAttributeNotAnAttribute//代表不参照
                                    multiplier:1.0
                                      constant:200],
        
        // 高100
        [NSLayoutConstraint constraintWithItem:blueView
                                     attribute:NSLayoutAttributeHeight//高
                                     relatedBy:NSLayoutRelationEqual
                                        toItem:nil
                                     attribute:NSLayoutAttributeNotAnAttribute
                                    multiplier:1.0
                                      constant:100],
        
        // 水平居中，参照self.view
        [NSLayoutConstraint constraintWithItem:blueView
                                     attribute:NSLayoutAttributeCenterX//中心x坐标
                                     relatedBy:NSLayoutRelationEqual
                                        toItem:self.view
                                     attribute:NSLayoutAttributeCenterX
                                    multiplier:1.0
                                      constant:0],
        
        // 垂直居中，参照self.view
        [NSLayoutConstraint constraintWithItem:blueView
                                     attribute:NSLayoutAttributeCenterY//中心y坐标
                                     relatedBy:NSLayoutRelationEqual
                                        toItem:self.view
                                     attribute:NSLayoutAttributeCenterY
                                    multiplier:1.0
                                      constant:0],
    ]];
    
    // 文字标签
    UILabel *label = [[UILabel alloc] init];
    label.text = @"Hello AutoLayout";
    label.textColor = [UIColor whiteColor];
    label.translatesAutoresizingMaskIntoConstraints = NO;//依旧关闭
    [blueView addSubview:label]; // label加到blueView里，表明视图关系
    
    // label不需要宽高约束，因为有字体等内部计算宽高
    [NSLayoutConstraint activateConstraints:@[
        
        // 水平居中，参照blueView
        [NSLayoutConstraint constraintWithItem:label
                                     attribute:NSLayoutAttributeCenterX
                                     relatedBy:NSLayoutRelationEqual
                                        toItem:blueView
                                     attribute:NSLayoutAttributeCenterX
                                    multiplier:1.0
                                      constant:0],
        
        // 垂直居中，参照blueView
        [NSLayoutConstraint constraintWithItem:label
                                     attribute:NSLayoutAttributeCenterY
                                     relatedBy:NSLayoutRelationEqual
                                        toItem:blueView
                                     attribute:NSLayoutAttributeCenterY
                                    multiplier:1.0
                                      constant:0],
    ]];
}
```

# 现代约束库

> 初步学习到这里仅作了解，后续内容笔者会继续学习补上

原始的NSLayoutConstraint API功能完整，但每条约束都要写一大串，实际项目里挺繁琐的。苹果后来提供了更简洁的写法：

1. **Anchor API**：通过视图的anchor属性链式添加约束，是目前最主流的纯代码AutoLayout写法
2. **VFL（Visual Format Language）**：用字符串描述约束，如`@"H:|-20-[view(200)]"`，看起来直观但没办法描述视图间的对齐关系，现在基本被Anchor API取代