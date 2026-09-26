# NSUserDefaults 是什么

- NSUserDefaults 是一种轻量级的数据持久化方案，轻量级指的是它的存储适用于一些小型且重要的数据
- 它的用法非常类似于字典 `NSDictionary`，都是以键值对的方式存取数据，我们可以用一个字符串的 key 来存一个值，之后再拿着这个 key 去取出来

> 你可以把它理解成一个永久版的字典——普通字典在程序运行结束后就消失了，而 NSUserDefaults 里的数据会被写入本地文件，下次打开 App 还在，只是需要再从里面取出来

## 适合存什么

- 因为它是轻量级的，所以 NSUserDefaults 更适合存储类似以下数据
  1. 用户的偏好设置（如深色模式开关）
  2. 某个小数值、布尔值的开关状态

> 不适合用它存图片、大段文字、复杂对象列表等，NSUserDefaults 只适合存"配置"级别的小数据，原因是使用NSUserDefaults的时候运行app会直接将里面存储的所有数据一次性加载进内存，启动速度就会变慢

# 基本用法

## 获取实例

- NSUserDefaults其实是一个单例，我们可以通过它的类方法拿到单例对象

它叫做 `standardUserDefaults`

```objc
NSUserDefaults *defaults = [NSUserDefaults standardUserDefaults];
```

> 它是一个单例，整个 文件里拿到的都是同一个对象，所以不需要自己 alloc init

## 存数据

- 存数据用的是一系列 `setXxx:forKey:` 方法，根据数据类型不同，方法名也不一样

```objc
NSUserDefaults *defaults = [NSUserDefaults standardUserDefaults];

//存一个字符串
[defaults setObject:@"aaa" forKey:@"userName"];

//存一个整数
[defaults setInteger:18 forKey:@"age"];

//存一个布尔值
[defaults setBool:YES forKey:@"isAudlt"];

//存一个浮点数
[defaults setFloat:9.9 forKey:@"score"];
```

- 可以看到，根据值的类型，方法名会有变化：
  - 对象类型（NSString、NSArray、NSDictionary 等）用 `setObject:forKey:`
  - 基础类型（int、BOOL、float、double）有各自对应的方法

> 注意 key 是一个 NSString，需要存取时一致，很容易出现由于字符串不一致导致的存取错误

## 取数据

- 取数据同样根据类型有对应的方法，只需要传入存数据时用的 key

```objc
NSUserDefaults *defaults = [NSUserDefaults standardUserDefaults];

//取字符串
NSString *username = [defaults stringForKey:@"userName"];

//取整数
NSInteger age = [defaults integerForKey:@"age"];

//取布尔值
BOOL isFirstLaunch = [defaults boolForKey:@"isAudlt"];

//取浮点数
float score = [defaults floatForKey:@"score"];
```

> 如果取一个从来没存过的 key，不会崩溃，基础类型会返回 0 或 NO，对象类型会返回 nil，也就是默认的形态

## 删数据

- 删数据直接用 `removeObjectForKey:` 方法，传入 key 就可以

```objc
[defaults removeObjectForKey:@"username"];
```

# 关于 synchronize

```objc
[defaults synchronize];
```

> 这是在以前的 iOS 版本里，用来强制将内存中的数据立刻写入磁盘的方法，但现在已经被优化了，会自动添加，无须手动调用，所以写不写无所谓

# 实际使用举例

## 判断是否是第一次启动

- 这是 NSUserDefaults 最常见的使用场景之一，在 App 里判断用户是否是第一次打开，决定是否展示引导页

```objc
NSUserDefaults *defaults = [NSUserDefaults standardUserDefaults];
BOOL isFirst = [defaults boolForKey:@"isFirst"];

if (!isFirst) {
    //第一次启动，展示引导页
    [self showOnboardingPage];
    //标记为已经启动过
    [defaults setBool:YES forKey:@"isFirst"];
}
```

> 这里的逻辑是，第一次进来时 `boolForKey:` 取到的是 NO，所以 `!isFirst` 是 YES，走进 if 里展示引导页，同时立刻把它存为 YES，下次打开就不会再进来了

## 保存用户名

```objc
//保存用户名
- (void)saveUsername:(NSString *)name {
    [[NSUserDefaults standardUserDefaults] setObject:name forKey:@"username"];
}

//需要显示用户名时取出来
- (NSString *)loadUsername {
    return [[NSUserDefaults standardUserDefaults] stringForKey:@"username"];
}
```

## 注意

- NSUserDefaults不要存储大型数据如图片，段落等加载时间长的内容
- 同时由于NSUserDefaults是以plist文件的形式存储的，所以信息容易被读取，不要将密码等敏感信息存储在NSUserDefaults中
- 建议提前把key字符串固定，防止手动写的时候前后不一致导致出错