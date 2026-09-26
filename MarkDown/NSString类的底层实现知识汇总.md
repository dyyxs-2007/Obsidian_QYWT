###### <center><font face="楷体" color=green>NSString的底层实现原理与知识汇总</font><center>

# 类簇的概念与理解
## 类簇的一般组成与逻辑
- 类簇的本质就是：<font color=red>父类声明公用方法，由各个子类实现</font>
- 详细铺开说，类簇由三个部分组成，下面我一一介绍
### 抽象基类 
- 抽象基类就是对外界的接口，它声明了所有子类需要实现的方法
- 在外界会以这个基类声明对象，对象会根据具体的需要进入这个接口中的其他部分
### 具体子类
- 一个抽象基类可能会有很多个子类，用于实现不同的功能，这些子类继承于抽象基类，所以需要实现基类所声明出的公用方法
- 同时子类也可以实现自己的方法，用于分工完成不同的任务
### 基类方法
#### 普通方法
- 这里就声明了上面所说的公用方法，需要每一个子类去实现
#### 工厂方法
- 如何确定具体使用的是哪个子类呢？这时候就需要工厂方法，这些方法通常会给出一个初始化的方法，用这个初始化的方法去声明基类对象，方法会自动确定需要哪个子类完成任务
> 上面这三个内容也被称为“**工厂模式**”，对外只暴露基类接口和能完成呢内容，具体怎么分工，怎么完成，工厂内部自行决定，这样的设计思维就是面向接口编程
## NSString类簇的组成
- 看完上面一般类簇的组成，我们来看一下NSString这个类簇的组成
- 我们还是从三个方面入手:
1. 抽象基类：
   在NSString中，作为对外的接口，这个基类就叫做NSString，继承于NSObject，代码长这样
   ```objc
   @interface NSString : NSObject;
   @property (readonly) NSUInteger length;
   ...
   @end
   ```
   我们先只看声明部分，说NSString继承于NSObject,还有对应的属性
2. 具体子类
- NSString的具体子类由四个常用的和未公开数量的苹果内部私有子类组成，我们只需要关心前四个即可，这里只做简单介绍下文会重点讲解
    1. __NSCFConstantString
       - 这个子类用于处理常量型数据
    2. __NSCFString
       - 正常的堆上的字符串对象
    3. NSTaggedPointerString
        - 短字符串对象在存储上的优化
    4. NSMutableString
        - 可变字符串
1. 基类方法
- 工厂方法(用于确定该哪个子类来实现需要的功能，如)：
```objc
// 工厂方法
+ (instancetype)stringWithString:(NSString *)string;
+ (instancetype)stringWithFormat:(NSString *)format, ...;
+ (instancetype)stringWithUTF8String:(const char *)nullTerminatedCString;
```
> 我们看到这里的几个stringwith...或许十分熟悉，初始化不就用的是它吗？没错👍，它能帮我们分出到底应该使用哪个类来存储变量

- 普通方法(用于规范每个子类都需要实现的功能，如)：
```objc
// 普通方法
- (unichar)characterAtIndex:(NSUInteger)index;
- (BOOL)isEqualToString:(NSString *)aString;
- (NSString *)stringByAppendingString:(NSString *)aString;
- (NSRange)rangeOfString:(NSString *)searchString;
- (NSString *)substringFromIndex:(NSUInteger)from;
- (NSString *)uppercaseString;
- (NSString *)lowercaseString;
```
> 诸如这些方法都是我们常用的获取字符串相关参数的方法，比如返回字符串长度，比较字符串是否相等，无论使用的是哪个类，都可以调用这些参数来得到需要的数据
# 对NSString内部深入探讨
## __NSCFConstantString
- 存储字符串字面量
> 字符串字面量指的是<u>在编译期间就可以完全确定的字符串</u>，如`@"hello"`,这样的字符串是不可改变的，被存储在一个**只读的数据段**
- __NSCFConstantString专门用于存储这些编译期间就能完全确定的字符串，由于它储存在只读数据段，所以不能修改，因此如果你声明出两个一样的字符串字面量，其实他们都指向了同一块可读数据段的区域
```objc
数据段里的实例对象
┌─────────────────┐
│ isa             │──→ NSCFConstantString 类对象
│ length          │
│ buffer          │──→ "Hello"（只读字符数据）
└─────────────────┘
```
- __NSCFConstantString的初始化方式主要有两种:
1. 直接赋值：`NSString* string1 = @"hello";`
2. 调用stringWithString方法：`NSString* string2 = [NSString stringWithString:@"hello"];`
- <font color=red>以上两种方法创建出来的变量string1和string2其实指向了同一块数据段区域</font>，只是变量名不同，所以在使用以上两种方式声明字符串的时候就会从创造出在数据段存储的字符串
> NSCFConstantString的引用计数是-1，以前是无符号整型的上限，现在规范为-1，其实就是代表这个变量不归引用计数管，无论给这个引用计数+1或者-1它始终不受影响
## __NSCFString
- 存储长动态字符串
- 这个就是我们比较常用的初始化方式
- 不妨思考一下，动态的反义词是什么？是静态，和刚刚的一样，静态指的就是在编译期间就能确定的量，但是动态呢？显然是运行的时候才能确定的量，那什么样的字符串是运行的时候才能确定的量呢？其实就是可以传参的初始化方式`NSString *s = [NSString stringWithFormat:@"Hello %@", name];`
- 用上面这种方式创建出来的就是长动态字符串，它在运行的时候会创建一个新字符串来存储新的内容，实现动态的效果，这是__NSCFString内部的样子:
```objc
堆
__NSCFString 对象
┌─────────────────┐
│ isa             │  → 指向类对象
│ 引用计数         │  → 内存管理
│ length          │  → 字符串长度
│ buffer          │  → 指向实际字符数据
└─────────────────┘
```
我们也可以看到，和数据段不一样，这样创建出来的对象是存在堆上的，是需要被内存管理的
## NSTaggedPointerString
- 存储短动态字符串
- 短动态字符串其实就是长动态字符串的一种优化，我们知道，对象的本质就是指针，一个指针在64位机里占8字节大小，所以当一个动态字符串足够小的时候，为了尽量节省堆上的空间，可以直接把它写进指针里，只需要把指针首位地址上改成1，相当于告诉运行时：这是个存字符串的指针，没有在堆上开辟空间，而是直接把数据和变量名都存在栈上
```objc
栈
┌─────────────────────────────────────────┐
│  s  =  1  010  [h][i][长度][编码]        │
│         ↑  ↑    ↑  ↑                    │
│         │  │    字符数据                 │
│         │  tag标识是NSString             │
│         最高位1，代表Tagged Pointer       │
└─────────────────────────────────────────┘
没有堆对象
```

## NSMutableString
- 存储可变字符串

可变字符串在这个类簇下显得比较特殊，因为它直接对外暴露它自己的接口，我们可以直接声明这个类的对象
- 可以将其理解为__NSCFString的拓展，增加了可以动态管理的容量，不够了就申请
```objc
堆
NSMutableString 对象
┌─────────────────────────┐
│ isa                     │ → 指向 NSMutableString 类对象
│ 引用计数                 │ → 内存管理
│ length = 5              │ → 实际字符数
│ capacity = 10           │ → 最大容量(满了就会自动扩容)
│ buffer ─────────────────┼──→ [H][e][l][l][o][_][_][_][_][_]
└─────────────────────────┘     用了5个    预留了5个
```
# NSString的一些特殊性质和容易踩的坑
1. 对NSString对象进行copy操作的时候，返回的是原对象指针，因为苹果觉得NSString既然无法被修改，所以对它进行copy操作的时候会直接返回原对象指针，**并且将引用计数➕1**(因为它在堆上)
> 如果是__NSCFConstantString,由于它的引用计数很大很大，所以这个引用计数➕1没有起到作用
2. 不要用 == 来比较字符串，在oc中"=="比较的是内存地址，也就是两个指针首否指向一个变量，比较字符串时，应该用NSString重写过的isEqualToString
3. NSString的属性用copy而不用strong，copy的含义就是对其他变量赋值的时候是拷贝一份不可变对象，这样就防止了NSString将对象传递出去给NSMutableString却被修改了原本信息
4. 不能把copy的结果赋给NSMutableString，因为普通copy只返回不可变对象
5. 不能把字面量赋值给NSMutableString，因为只读数据段的isa指向的是__NSConstantString而不是NSMutableString,这样就会导致找不到对应的方法而崩溃
6. NSString的字符串拼接是创建一个新字符串来存储拼接后的字符串，这样反复进行对内存消耗很大而且原字符串的内存释放要做好
