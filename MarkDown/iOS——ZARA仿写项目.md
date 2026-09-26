<center><font color=green face="楷体">iOS——ZARA仿写项目总结</font></center>

# 文件结构

## 思路

搭建文件结构前，我们需要弄清楚MVC框架都需要哪些内容，我们从内容入手分析

- 需要展示三个界面——首页，我的，分类，所以需要使用TabBar嵌套Navigation的方式实现

在SceneDelegate中创建：

```objc
- (void)scene:(UIScene *)scene willConnectToSession:(UISceneSession *)session options:(UISceneConnectionOptions *)connectionOptions {
    UIWindowScene *windowScene = (UIWindowScene *)scene;
    self.window = [[UIWindow alloc] initWithWindowScene:windowScene];
    
    ZAHomeViewController *home = [[ZAHomeViewController alloc] init];
    UINavigationController *homeNav = [[UINavigationController alloc] initWithRootViewController:home];
    homeNav.tabBarItem = [[UITabBarItem alloc] initWithTitle:@"首页" image:[UIImage systemImageNamed:@"house"] tag:0];//控制器1
    
    ZACategoryViewController *cate = [[ZACategoryViewController alloc] init];
    UINavigationController *cateNav = [[UINavigationController alloc] initWithRootViewController:cate];
    cateNav.tabBarItem = [[UITabBarItem alloc] initWithTitle:@"分类" image:[UIImage systemImageNamed:@"cart"] tag:0];//控制器2
    
    ZAProfileViewController *prof = [[ZAProfileViewController alloc] init];
    UINavigationController *profNav = [[UINavigationController alloc] initWithRootViewController:prof];
    profNav.tabBarItem = [[UITabBarItem alloc] initWithTitle:@"我的" image:[UIImage systemImageNamed:@"person"] tag:0];//控制器3
    
    UITabBarController *bar = [[UITabBarController alloc] init];
    bar.viewControllers = @[homeNav, cateNav, profNav];//三个navigation塞进tabbar里面
    
    self.window.rootViewController = bar;
    [self.window makeKeyAndVisible];
}
```



以下是文件结构

## MVC框架

```
项目/
├── Controller/
│   ├── ZACategoryViewController.h
│   ├── ZACategoryViewController.m
│   ├── ZAHomeViewController.h
│   ├── ZAHomeViewController.m
│   ├── ZAProfileViewController.h
│   ├── ZAProfileViewController.m
│   ├── ZAUserDetailViewController.h
│   └── ZAUserDetailViewController.m
│
├── Model/
│   ├── ZAFuncModel.h
│   ├── ZAFuncModel.m
│   ├── ZAProfileModel.h
│   └── ZAProfileModel.m
│
└── View/
    ├── ZAFuncCell.h
    ├── ZAFuncCell.m
    ├── ZAProfileCell.h
    └── ZAProfileCell.m
```

我们先给出文件的结构，接着我会解释为什么这样设计：

***从Model入手***

1. 我们知道Model要用于为需要复用的数据整理打包成一个数据结构，所以“我的”界面的tableView的信息就需要单独打包成Model，而且需要将个人信息与下面展示的界面分开，所以Model需要两个h/m文件
2. 对应的View直接配套Model不多赘述
3. 对于Controller首先需要三个h/m文件来完成首页，我的，分类的UI控制，其次需要点击头像进入cell界面，多接一个即可

- 大体思路如此，下面我针对需要实现的重点内容进行讲解

# 重点需要实现的内容

## 无限轮播图

具体内容可以查看笔者这篇文章：[iOS——无限轮播图学习](https://blog.csdn.net/2505_93363502/article/details/160798568?spm=1001.2014.3001.5501)

我们针对这里数据的初始化进行简单讲解

```objc
- (void)setupPicture {
    self.picture = @[@"005", @"001", @"002", @"003", @"004", @"005", @"001"];
    NSArray *strData = self.picture;
    UIImageView *prev = nil;
    for (NSInteger i = 0; i < strData.count; i++) {
        UIImageView *imageView = [[UIImageView alloc] init];
        imageView.image = [UIImage imageNamed:strData[i]];
        [self.scrollView addSubview:imageView];
        
        [imageView mas_makeConstraints:^(MASConstraintMaker *make) {
            make.top.bottom.equalTo(self.scrollView);
            make.width.mas_equalTo(self.view.bounds.size.width);
            if (i == 0) {
                make.left.equalTo(self.scrollView);
            } else if (i == strData.count - 1) {
                make.left.equalTo(prev.mas_right);
                make.right.equalTo(self.scrollView);
            } else {
                make.left.equalTo(prev.mas_right);
            }
        }];
        prev = imageView;
    }
}
```

- 首先创建数组，这里的数组内容是每个图片的名字，图片需要被放在Assets里面
- 接着设置prev，这一步的目的是保证每张图片可以紧贴着上一张图片，当然也可以写死坐标，但这样不够灵活，也不够轻松
- 可以看到，我们只需要单独约束第一章和最后一张的左右紧贴ScrollView，剩余图片只保证左边紧贴上一张图片(prev)的右边即可

## UIPageControl与页面联动

```objc
- (void)setupPageControl {
    self.pageControl = [[UIPageControl alloc] init];
    self.pageControl.numberOfPages = self.picture.count - 2;
    self.pageControl.pageIndicatorTintColor = [UIColor whiteColor];
    self.pageControl.currentPageIndicatorTintColor = [UIColor grayColor];
    self.pageControl.currentPage = 0;
    [self.view addSubview:self.pageControl];
    
    [self.pageControl mas_makeConstraints:^(MASConstraintMaker *make) {
        make.bottom.equalTo(self.scrollView).offset(-8);
        make.centerX.equalTo(self.scrollView);
    }];
    [self.scrollView setContentOffset:CGPointMake(self.scrollView.bounds.size.width, 0) animated:NO];
}
```

上面是创建初始化pageControl，下面讲解同步更新pageControl

```objc
- (void)scrollViewDidScroll:(UIScrollView *)scrollView {
    CGFloat currX = scrollView.contentOffset.x;
    CGFloat width = self.scrollView.bounds.size.width;
    
    if (currX <= 0) {
        [scrollView setContentOffset:CGPointMake(width * (self.picture.count - 2), 0) animated:NO];
    } else if (currX >= width * (self.picture.count - 1)) {
        [scrollView setContentOffset:CGPointMake(width, 0) animated:NO];
    }
    
    currX = scrollView.contentOffset.x;
    self.pageControl.currentPage = currX / width - 1;
}
```

- 上方部分就是正常的无限轮播图的跳转逻辑
- 注意看下面重新取了currX，原因是当中间的跳转逻辑生效之后，此时currX已经改变，为了同步更新pageControl，就要获得最新的currX

## segmentControl与ScrollView联动

segmentControl与ScrollView的创建与初始化这里不多赘述，直接看联动逻辑

```objc
- (void)scrollViewDidScroll:(UIScrollView *)scrollView {
    CGFloat currX = scrollView.contentOffset.x;
    CGFloat width = self.scrollView.bounds.size.width;
    currX += width / 2;
    NSInteger index = currX / width;
    self.seg.selectedSegmentIndex = index;
}

- (void)setupSeg {
    self.seg = [[UISegmentedControl alloc] initWithItems:@[@"女士", @"男士", @"香水"]];
    [self.seg addTarget:self action:@selector(segChange) forControlEvents:UIControlEventValueChanged];//绑定滑动方法
    self.seg.selectedSegmentIndex = 0;
    self.navigationItem.titleView = self.seg;
}

- (void)segChange {
    NSInteger index = self.seg.selectedSegmentIndex;
    CGFloat width = self.scrollView.bounds.size.width;
    [self.scrollView setContentOffset:CGPointMake(index * width, 0) animated:YES];//更新图片位置
}
```

- 第一个方法`currX += width / 2;`是为了保证滑动到一半的时候更新segmentControl

## 多界面传值更新头像

这里我使用的是代理传值，也就是在ZAProfileCell.h里面声明协议，让Controller执行

```objc
- (void)exchangePicture:(NSString *)picture {
    ZAProfileModel *model = [[ZAProfileModel alloc] init];
    model.picture = picture;
    model.userName = @"dyyxs";
    model.subTitleName = @"秋雨梧桐叶落莳";
    [self.section replaceObjectAtIndex:0 withObject:model];
    NSIndexSet *set = [NSIndexSet indexSetWithIndex:0];
    [self.tableView reloadSections:set withRowAnimation:UITableViewRowAnimationFade];
}

//------------------------------------------分割线
#import <UIKit/UIKit.h>
#import "ZAProfileModel.h"

NS_ASSUME_NONNULL_BEGIN

@interface ZAProfileCell : UITableViewCell
@property (nonatomic, strong) ZAProfileModel *model;
- (void)configWithModel:(ZAProfileModel *)model;
@end

NS_ASSUME_NONNULL_END
```
