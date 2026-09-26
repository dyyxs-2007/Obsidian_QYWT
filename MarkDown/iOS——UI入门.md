###### <center><font color=green face="楷体">iOS——UI入门</font></center>
# 简介
- 在Xcode里面创建出来第一个iOS应用的时候，明显能感觉到界面中的文件变多了很多
- 在UI学习中，我们主要需要关注这两个文件：ViewController和SceneDelegate
1. ViewController
- 这是一个创建时自动生成的类，用来展示初始界面，当然它和我们自定义出来的一个类没什么不同，本质只是它自动成为根视图控制器
- 在后面的学习中，可以通过更改根视图控制器来更换初始的界面是谁，在刚开始学习时，可以把它理解为，将代码写在这个文件的时候对UI元素的布局都在这里
2. SceneDelegate
- 这是用来设置初始窗口和根控制器的文件，它可以控制第一个展示的是哪个界面

# UIView
## 概念
- UIView 是 UIKit 中一切可见元素的基础，无论是按钮、文字还是图片，本质上都是 UIView 或它的子类
- 可以把它理解为屏幕上的一块矩形区域，它负责两件事：在这块区域内绘制内容，以及响应用户的触摸操作

## 基本写法
- UIView 在创建时需要传入一个 `frame` 参数，用来确定它的位置和大小，创建后通过 `addSubview:` 添加到父视图上才能显示
```objc
UIView *myView = [[UIView alloc] initWithFrame:CGRectMake(50, 100, 200, 150)];
myView.backgroundColor = [UIColor blueColor];
[self.view addSubview:myView];
```
> 如果只创建了 UIView 但没有调用 `addSubview:`，它是不会出现在屏幕上的，这一点初学时容易忽略

## UIView 的一些常用属性和用法分析
1. 外观相关
    - `backgroundColor`：设置背景颜色
    - `alpha`：控制整体透明度，范围从 0.0（完全透明）到 1.0（完全不透明）
    - `isHidden`：直接控制显示或隐藏，与 `alpha = 0` 的区别在于隐藏后不参与触摸事件响应
2. 高级属性layer
    - layer是UIView能将真正内容画到屏幕上的东西，事实上，UIView是对于layer的包装，所以layer能做到很多UIView在视图上做不到的事情，如阴影，圆角与裁切，边框效果
    - 通过 `layer.cornerRadius` 设置圆角半径
    - 需要同时将 `clipsToBounds` 设为 `YES`，否则子视图超出边界的部分不会被裁剪，圆角也可能不生效

# UILabel
## 概念
- UILabel 是专门用于展示静态文字的视图，继承自 UIView，因此 UIView 拥有的属性它都可以使用`(所以需要addSubview)`
- 这里说的"静态"是指用户无法直接点击编辑，文字内容只能由代码控制，如果需要用户输入，则要用 UITextField

## 基本写法
- 创建方式与 UIView 相同，通过 `frame` 确定位置和大小，再 `addSubview:` 添加上去
```objc
UILabel *label = [[UILabel alloc] initWithFrame:CGRectMake(50, 100, 200, 50)];
label.text = @"Hello, World";
label.textColor = [UIColor blackColor];
label.textAlignment = NSTextAlignmentCenter;
[self.view addSubview:label];
```

## UILabel 的一些常用属性和用法分析
1. 文字内容与样式
    - `text`：设置显示的文字
    - `textColor`：设置文字颜色
    - `font`：设置字体与字号，例如 `[UIFont systemFontOfSize:16]`
    - `textAlignment`：文字对齐方式，常用 `NSTextAlignmentLeft`、`NSTextAlignmentCenter`、`NSTextAlignmentRight`
2. 多行显示
    - 默认情况下 UILabel 只显示一行，超出部分会被截断
    - 将 `numberOfLines` 设为 `0`，Label 就会根据内容自动换行，不再限制行数
> 要注意的是，设置 `numberOfLines = 0` 后，Label 的高度需要能容纳下所有文字，否则还是会被截断，后面学习自动布局后这个问题会更容易处理

# UIButton
## 概念
- UIButton 是用于响应用户点击操作的控件，同样继承自 UIView`(所以也需要addSubview)`
- 与 UILabel 最本质的区别在于，UIButton 可以绑定事件，用户点击后会触发对应的代码逻辑去执行

## 基本写法
- 推荐用 `buttonWithType:` 创建，常用类型是 `UIButtonTypeSystem`，它自带点击高亮效果
```objc
UIButton *button = [UIButton buttonWithType:UIButtonTypeSystem];
button.frame = CGRectMake(100, 200, 120, 44);
[button setTitle:@"点击我" forState:UIControlStateNormal];
[button addTarget:self action:@selector(buttonTapped) forControlEvents:UIControlEventTouchUpInside];
[self.view addSubview:button];

- (void)buttonTapped {
    NSLog(@"按钮被点击了");
}
```

## UIButton 的一些常用属性和用法分析
1. 文字与颜色
    - `setTitle:forState:`：设置不同状态下的文字，最常用的状态是 `UIControlStateNormal`
    - `setTitleColor:forState:`：设置文字颜色
    - `backgroundColor`：设置按钮背景色
    - button的属性中有一个titleLabel，它可以访问button的文字栏，可以将它视为label进行操作，如居中对齐等
2. 事件绑定
    - 通过 `addTarget:action:forControlEvents:` 将按钮与方法关联，`forControlEvents` 参数代表触发时机
    - 最常用的触发时机是 `UIControlEventTouchUpInside`，即手指在按钮范围内抬起时才触发，这样用户如果按下后悔了，可以把手指滑出按钮范围来取消操作
> `action:` 里传入的是一个 `SEL`，也就是方法的选择器，通过 `@selector()` 来获取，括号内写方法名，本质上是在运行时去查找并调用对应的方法