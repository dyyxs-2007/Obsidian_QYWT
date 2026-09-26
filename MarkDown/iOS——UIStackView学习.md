# UIStackView是什么

## 内部属性

1. axis	决定横着排列还是竖着排列
2. distribution   决定子视图怎么分配空间
3. aligment       决定子视图怎么对齐
4. spacing        决定子视图间距

## 基础代码结构

```objc
UIStackView *stack = [[UIStackView alloc] init];
stack.axis = UILayoutConstraintAxisVertical;// 排列方向
stack.distribution = UIStackViewDistributionFill;// 分配方式
stack.alignment = UIStackViewAlignmentFill;// 对齐方式
stack.spacing = 8;// 间距

[self.view addSubview:stack];//只需要对stackView进行约束
[stack mas_makeConstraints:^(MASConstraintMaker *make) {
    make.center.equalTo(self.view);
    make.width.mas_equalTo(300);
}];

// 添加子视图用 addArrangedSubview，不是 addSubview
[stack addArrangedSubview:label1];
[stack addArrangedSubview:label2];
```

# 子视图的宽高规则

- 设置了哪个方向的axis，那个方向就自己管，另一个方向由stackView管

## 设置子视图高度

```objc
UIView *redView = [[UIView alloc] init];
redView.backgroundColor = [UIColor systemRedColor];
[stack addArrangedSubview:redView];

[redView mas_makeConstraints:^(MASConstraintMaker *make) {
    make.height.mas_equalTo(60);
}];//只需要约束高度就好，宽度由stackView管理
```

# distribution(主轴空间分配)

- 枚举类型后缀：

1. `Fill`——拉伸其中一个子视图填满
2. `FillEqually`——所有子视图等高/宽
3. `FillProportionally`——按内容大小比例分配
4. `EqualSpacing`——子视图保持自身大小，间距自动相等
5. `EqualCentering`——子视图中心点间距相等

> EqualSpacing在使用的时候，要保证子视图有大小，不然不会显示

# alignment(交叉轴对齐方式)

- 枚举类型后缀：

1. `Fill`——撑满宽度
2. `Leading`——靠左对齐
3. `Trailing`——靠右对齐
4. `Center`——居中
5. `Top`——靠上
6. `Bottom`——靠底

# 简单演示

- 竖向等分三色块

```objc
UIStackView *stack = [[UIStackView alloc] init];
stack.axis = UILayoutConstraintAxisVertical;
stack.distribution = UIStackViewDistributionFillEqually;
stack.alignment = UIStackViewAlignmentFill;
stack.spacing = 8;

[self.view addSubview:stack];
[stack mas_makeConstraints:^(MASConstraintMaker *make) {
    make.center.equalTo(self.view);
    make.width.mas_equalTo(300);
    make.height.mas_equalTo(200);
}];

UIView *v1 = [[UIView alloc] init];
v1.backgroundColor = [UIColor systemRedColor];
[stack addArrangedSubview:v1];

UIView *v2 = [[UIView alloc] init];
v2.backgroundColor = [UIColor systemBlueColor];
[stack addArrangedSubview:v2];

UIView *v3 = [[UIView alloc] init];
v3.backgroundColor = [UIColor systemGreenColor];
[stack addArrangedSubview:v3];
```

![IMG_FD3095889E9D-1](https://img.cdn1.vip/i/6a12bb3b2a615_1779612475.webp)

# StackView嵌套

- StackView的好用之处就在于可以进行嵌套，也就是说可以把很多StackView直接放进大StackView进行排列，这样可以简单完成很复杂的UI设计
- 比如如果想要实现以下布局：

```
头像	名字
		 粉丝数
```

- 可以通过这样的嵌套：

```objc
		//头像等内容的创建
		UIImageView *avatarView = [[UIImageView alloc] init];
    UIImage *avatarImage = [UIImage systemImageNamed:@"person.circle.fill"];
    avatarView.image = avatarImage;
    avatarView.tintColor = [UIColor systemBlueColor];
    avatarView.contentMode = UIViewContentModeScaleAspectFit;

    // 固定头像尺寸
    [avatarView mas_makeConstraints:^(MASConstraintMaker *make) {
        make.width.height.mas_equalTo(52);
    }];

    UILabel *nameLabel = [[UILabel alloc] init];
    nameLabel.text = @"张三";
    nameLabel.font = [UIFont boldSystemFontOfSize:16];
    nameLabel.textColor = [UIColor labelColor];

    UILabel *fansLabel = [[UILabel alloc] init];
    fansLabel.text = @"粉丝 12.4万";
    fansLabel.font = [UIFont systemFontOfSize:13];
    fansLabel.textColor = [UIColor secondaryLabelColor];

// 外层横向 StackView
UIStackView *hStack = [[UIStackView alloc] init];
hStack.axis = UILayoutConstraintAxisHorizontal;
hStack.spacing = 12;
hStack.alignment = UIStackViewAlignmentCenter;

// 内层竖向 StackView
UIStackView *vStack = [[UIStackView alloc] init];
vStack.axis = UILayoutConstraintAxisVertical;
vStack.spacing = 4;

// 内层添加子视图
[vStack addArrangedSubview:nameLabel];
[vStack addArrangedSubview:fansLabel];

// 外层添加头像和内层
[hStack addArrangedSubview:avatarView];
[hStack addArrangedSubview:vStack];

// 只需要给外层写位置约束
[self.view addSubview:hStack];
[hStack mas_makeConstraints:^(MASConstraintMaker *make) {
    make.center.equalTo(self.view);
}];
```

- 效果如下

![IMG_2992](https://img.cdn1.vip/i/6a12bdd9719f7_1779613145.webp)

# UIStackView的优势

1. 动态显示隐藏

- 普通 View 隐藏后空间还占着，StackView 里的子视图隐藏后空间自动消失，其他视图会自动补位显示，将hidden属性设置no之后空间会自动回来

2. 使用`[v1 removeFromSuperview];`可以动态删除子视图，同理使用add可以动态更新
3. 嵌套使用StackView，复杂布局时代码量很少