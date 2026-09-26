###### <center><font color=green face="楷体">iOS——界面间传值</font></center>

# 界面间传值的含义

- 在实际开发中，一个 App 往往不止一个界面，不同界面之间经常需要共享数据
- 比如在列表页选中某一行，跳到详情页展示这一行的具体内容；或者在设置页填写了昵称，回到主页后主页要把新昵称显示出来
- 这种"数据从一个界面传输到另一个界面"的行为，就叫做**界面间传值**

并且我们需要考虑到，什么时候应该在当前页面向下一个页面传值(正向)，什么时候要向前一个页面传值(反向)

- **正向传值（A → B）**：A 把数据准备好，在 push 到 B 之前交给 B，B 展示出来。数据随着页面被切换也传入对应的界面，比如可以获取头像的信息，放大之类的
- **反向传值（B → A）**：B 填写或修改了数据，在 pop 回 A 的时候把数据"还回来"，A 拿到后更新界面，应用在密码账号验证等方面

# 三种方式总览

iOS 开发中最常用的界面间传值方式有三种：

1. 直接属性传值
   - 用于正向，直接对已经创建出来的下一个界面的某个属性赋值，这样数据就直接来到了下一个界面中
2. delegate反向传值
   - 下一个界面定义一个代理属性，创建的时候先把代理设置成自己，这样当下一个界面完成活动可以调用代理的方法，这样代理就可以实现活动或者是对代理对象的属性赋值
3. Block反向传值
   - 这里用到了block的封存数据的机制，因为block在创建的时候会捕获当时的变量，也就可以将当时的"self"，也就是当前界面封存，在下一个界面就可以对block进行调用，进一步对前一个界面赋值

# 具体实例

## 正向传值

### 原理

push 到 B 之前，直接给 B 的属性赋值，B 在 `viewDidLoad` 里读取这个属性并使用。

### 实现步骤

**第一步：在 B 的 .h 里声明一个属性来接收数据**

```objc
// BViewController.h
@interface BViewController : UIViewController
@property (nonatomic, copy) NSString *userName;
@end
```

**第二步：A 在 push 之前给这个属性赋值**

```objc
// AViewController.m
- (void)goToB {
    BViewController *bVC = [[BViewController alloc] init];
    bVC.userName = @"Tom";//直接给下一个界面的某个属性赋值
    [self.navigationController pushViewController:bVC animated:YES];//赋值完再切换
}
```

**第三步：B 在 viewDidLoad 里直接使用这个属性**

```objc
// BViewController.m
- (void)viewDidLoad {
    [super viewDidLoad];
    self.navigationItem.title = self.userName;  // 直接读取使用
}
```

> 赋值一定要在 `push` **之前**完成，一旦 push 执行，`viewDidLoad` 就已经结束了，再去赋值也不会有任何效果

## 反向传值

反向传值有两种主要方法：**Delegate** 和 **Block**，它们都可以完成反向传输数据，只是逻辑略有不同

### 方法一：Delegate

#### 原理

B 定义一个协议，A 遵守这个协议并把自己设为 B 的 delegate。B 需要把数据传回去的时候，调用协议方法，A 在协议方法里接收数据并更新界面。

可以把它理解为一种"委托关系"：B 不认识 A 是谁，只知道有一个对象遵守了某个协议，需要传数据时就通过协议调用，A 就会被这个调用触发

#### 实现步骤

**第一步：在 B 的 .h 里定义协议和 delegate 属性**

```objc
// BViewController.h
@protocol BViewControllerDelegate <NSObject>
- (void)didInputName:(NSString *)name;
@end

@interface BViewController : UIViewController
@property (nonatomic, weak) id<BViewControllerDelegate> delegate;  // weak防止循环引用，同时可以保存a的信息
@end
```

**第二步：B 在需要回传数据的时候调用 delegate 方法**

```objc
// BViewController.m
- (void)saveAndGoBack {
    if ([self.delegate respondsToSelector:@selector(didInputName:)]) {
        [self.delegate didInputName:self.textField.text];
    }
    [self.navigationController popViewControllerAnimated:YES];
}
```

**第三步：A 声明遵守 B 的协议**

```objc
// AViewController.h
#import "BViewController.h"

@interface AViewController : UIViewController <BViewControllerDelegate>
@end
```

**第四步：A push 时把自己设为 B 的 delegate，并实现协议方法**

```objc
// AViewController.m
- (void)goToB {
    BViewController *bVC = [[BViewController alloc] init];
    bVC.delegate = self;  // 把自己设为 B 的 delegate，这行不能漏
    [self.navigationController pushViewController:bVC animated:YES];
}

- (void)didInputName:(NSString *)name {
    self.nameLabel.text = name;  // 在这里接收数据并更新 UI
}
```

> `respondsToSelector:` 是一个安全检查，用来确认 delegate 对象真的实现了这个方法，防止调用不存在的方法导致崩溃

### 方法二：Block

#### 原理

A 把"拿到数据后要做什么"这段逻辑打包成一个 Block，在 push 的时候交给 B。B 拿到数据后直接调用这个 Block，Block 里的逻辑就会执行，由于block的封存特性，可以对a的属性进行操作

#### 实现步骤（三步走）

**第一步：在 B 的 .h 里声明 Block 类型和属性**

```objc
// BViewController.h
typedef void(^NameCallback)(NSString *name);//这里的typedef把这个block重命名为了NameCallback

@interface BViewController : UIViewController
@property (nonatomic, copy) NameCallback nameCallback;  // Block 属性用 copy
@end
```

**第二步：B 在需要回传数据时调用 Block**

```objc
// BViewController.m
- (void)saveAndGoBack {
    if (self.nameCallback) {
        self.nameCallback(self.textField.text);
    }
    [self.navigationController popViewControllerAnimated:YES];
}
```

**第三步：A 在 push 时给 B 传入 Block 的具体实现**

```objc
// AViewController.m
- (void)goToB {
    BViewController *bVC = [[BViewController alloc] init];

    __weak typeof(self) weakSelf = self;  //typeof(self)是自动推导变量类型, __weak是弱引用防止循环引用
    bVC.nameCallback = ^(NSString *name) {
        weakSelf.nameLabel.text = name;
    };

    [self.navigationController pushViewController:bVC animated:YES];
}
```

#### Block 的捕获机制

理解 Block 为什么能在 B 里修改 A 的 属性，关键在于"捕获"这个概念：

```
A 赋值 Block 时  →  Block 捕获 A 的 self，封存其内存地址
B 调用 Block 时  →  Block 用封存的地址找到 A，修改 A 的 UI
```

#### 循环引用问题

```objc
bVC.nameCallback = ^(NSString *name) {
    self.nameLabel.text = name;  // ← 直接使用 self，有问题
};
//如果直接使用self，self就会被block引用
//想释放self，就要先释放block，但是self同时又引用着block，导致谁也不能被释放

//用 __weak 打破循环
__weak typeof(self) weakSelf = self;
bVC.nameCallback = ^(NSString *name) {
    weakSelf.nameLabel.text = name;  // ← weakSelf 不会增加引用计数
};
```

> Block 属性之所以用 `copy` 而不是 `strong`，是因为 Block 在创建时默认存在栈上，一旦当前作用域结束就会被释放。`copy` 会把 Block 复制到堆上，生命周期就能得到保证，之后 B 随时调用都是安全的

# 选择建议

Delegate 和 Block 本质上是一样的功能，只在于不同情境的选择

- delegate主要用于传值内容很多并且格式规范的情况下使用
- Block主要用于简洁代码，快速开发中
