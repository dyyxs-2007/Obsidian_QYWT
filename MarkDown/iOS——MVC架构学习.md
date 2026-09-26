###### <center><font face="楷体" color=green>MVC架构初步学习</font></center>

# MVC架构的概念

## 三个部分

MVC 是一种解耦代码的设计，把一个功能拆成三块来写，分别是：

- **Model（模型层）**：数据内容是什么
- **View（视图层）**：数据怎么显示给用户看(UI怎么画)
- **Controller（控制器层）**：负责在两者之间传值，处理交互逻辑，比如点击，滑动

三个首字母合起来就是 Model-View-Controller，缩写 MVC。

## 使用MVC架构的好处

初学写 OC 的时候，我们经常把所有逻辑全堆在 ViewController 里。数据初始化，画UI，处理交互逻辑全写在一起，代码一长就没办法看，导致改不到想改的地方反而弄出bug

MVC 的思路是：职责分离。每一块只管自己该管的事：

- Model 只负责数据，不关心怎么显示
- View 只负责展示，不关心数据从哪来
- Controller 负责拼接起来两边

这样的好处是：
1. 改数据结构只针对 Model，改UI布局只动 View
2. 复用方便，同一个 Model 可以配不同的 View，省去很多不必要的代码

# 三部分的内容

## Model层

Model 就是数据的载体。比如要展示一篇文章，Model 里面就放着标题、内容、发布时间这些文章数据

Model 本身不做 UI，不直接操作视图，它只负责把数据存好、整理好，让其他两个部分取用

```objc
// 上面举例的文章的 Model
@interface Article : NSObject
@property (nonatomic, copy) NSString *title;
@property (nonatomic, copy) NSString *content;
@property (nonatomic, copy) NSString *date;
@end
```
> 注意Model一般只关注.h文件，当成一个数据结构用即可
## Controller

Controller 由于需要拼接Model和View，它负责：

- 从 Model 取数据
- 把数据交给 View 去显示
- 接收 View 上交互动作，再去更新 Model 或者刷新 View

在 iOS 开发里，ViewController 就是天然的 Controller

## View层

View 就是用户看到的那层界面，UILabel、UIButton、UITableViewCell 这些都是 View界面展示的东西

View 不处理业务逻辑，只负责把 Controller 交给它的数据"渲染"出来，所以它只接收Controller传递来的Model即可

> View 可以把用户的操作（比如点击、滑动）通过代理或者回调告诉 Controller，但它本身不决定点了之后做什么，诸如此类属于交互逻辑，本就应该交给Controller来处理

# Model该怎么设计

Model 的设计原则很简单：数据是什么样子，Model 就设计成什么样子，比如cell展示标题，头像，就写标题和头像的title进去

```objc
@interface UserModel : NSObject
@property (nonatomic, copy)   NSString *Name;
@property (nonatomic, copy)   NSString *avarPicture;
@end
```

# MVC传递的流程

整体的流程可以这么理解：
![alt text](f6f20ec663234c8b8065d091e63a02ff.png)
拆开来说就是：

1. 用户在 View 上做了操作（比如点了个按钮）
2. View 通过反向传值 把事件告诉 Controller
3. Controller 根据事件去操作 Model
4. Model 数据变化之后，Controller 拿到新数据
5. Controller 把新数据设置给 View，View 刷新显示

> 注意 Model 和 View 之间是没有直接联系的，都要经过 Controller 与他们交互

# 举例

## tableView

用一个常见场景来说明：展示一个用户列表，每行显示头像、名字、简介。

### Model设计

每一行对应一个用户，Model 就按用户信息来设计：

```objc
// UserModel.h
@interface UserModel : NSObject
@property (nonatomic, copy) NSString *name;//名字
@property (nonatomic, copy) NSString *bio;//简介
@property (nonatomic, copy) NSString *avatarUrl;//头像
@end
```

### Controller和View设计

Cell 就是 View 层，它只负责把数据显示出来，不管数据从哪来：

```objc
// UserCell.h
@interface UserCell : UITableViewCell
- (void)configureWithModel:(UserModel *)model; // 接收 Model
@end

// UserCell.m
- (void)configureWithModel:(UserModel *)model {
    self.nameLabel.text = model.name;
    self.bioLabel.text  = model.bio;
}
```

Controller 负责持有Model、搭建 tableView：

```objc
// UserListViewController.m
@interface UserListViewController () <UITableViewDataSource>
@property (nonatomic, strong) UITableView *tableView;
@property (nonatomic, strong) NSMutableArray<UserModel *> *users; // Model 数组
@end

@implementation UserListViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    [self loadData]; 
    [self setupTableView];
}

- (void)loadData {
    NSArray *data = @[
        @[@"小李", @"18岁", @"image1"],
        @[@"小芳", @"不爱吃水果", @"image2"]
    ]

    for (int i = 0; i < data.count; i++) {
        UserModel *model = [[UserModel alloc] init];
        model.name = data[i][0];
        model.bio = data[i][1];
        model.avatarUrl = data[i][2];
        [self.users addObject:model];
    }
}

- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section {
    return self.users.count;
}

- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath {
    UserCell *cell = [tableView dequeueReusableCellWithIdentifier:@"UserCell" forIndexPath:indexPath];
    UserModel *model = self.users[indexPath.row]; // Controller 从 Model 数组取数据
    [cell configureWithModel:model];              // 交给 View 去显示
    return cell;
}

@end
```

这样三层的职责就很清晰：
- `UserModel`：只管数据长什么样
- `UserCell`：只管一行怎么显示
- `UserListViewController`：负责把数据填进 Cell，管理整个列表的逻辑

# 总结

MVC 的核心就一句话：**让每部分只做自己该做的事。**

| 层 | 职责 | 不该做的事 |
|---|---|---|
| Model | 存数据、整理数据 | 操作 UI、直接更新视图 |
| View | 显示数据、传递用户操作 | 处理交互逻辑、直接改数据 |
| Controller | 协调 Model 和 View | 把太多业务逻辑堆在这里 |