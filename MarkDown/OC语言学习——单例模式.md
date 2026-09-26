###### <center><font color=green face="楷体">单例讲解</font></center>
# 单例的作用
- 在iOS开发中，常常会遇到某个类全局只需要一个实例对象的情况，**反复创建多余的实例对象，不仅会浪费内存，也会导致数据不一致**，就像国家只能有一个财政部，如果创建出了很多个，就把账本分散了
- 所以单例的作用就是<font color=red>在其内部自动实例化出一个全局可访问的唯一的对象</font>
## 单例的应用场景
1. 网络请求中心
2. 音乐播放器
> 还有其他的应用场景，这里不一一列出

- 归根结底，单例的应用场景就是全局从创建到程序结束都需要存在且不适合被多次创建的情况，就比如音乐播放器的播放引擎，不需要很多个音乐同时播放，所以它需要用到单例模式创建
# 单例实现的原理
## 饿汉模式
- 从上面的介绍不难看出，单例模式似乎是一个高级的“全局变量”，可以通过统一的入口获得，所以结合我们学到的static，<font color=red>可以先在单例中先用static声明一个全局的实例</font>，再在创建，初始化等方法中初始化或直接返回这个节点即可，这就是饿汉模式
- 常用写法如下
```objc
#import "Person.h"
@implementation Person

static Person* instance = nil;

+ (void)load {
    instance = [[super allocWithZone: NULL] init];
}
+ (instancetype)sharedInstance {
    return instance;
}
+ (instancetype)allocWithZone:(struct _NSZone *)zone {
    if (!instance) {
        instance = [super allocWithZone: zone];
    }
    return instance;
}
- (id)copyWithZone:(NSZone *)zone {
    return self;
}
- (id)mutableCopyWithZone:(NSZone *)zone {
    return self;
}
@end
```
- 我们重点看一下这里的load方法和sharedInstance方法
1. load方法
    - load是初始化的方法，调用它它会在全局先创建一个当前单例类的对象，此方法只用于创建
2. sharedInstance方法
    - 这个方法用于具体场景中的访问，因为外部无法直接访问单例内部的static声明的对象，这就需要通过一个方法来返回已经被创建出来的对象，可以看到直接返回的是当前的instance
    - 同时在内部使用的是`[[super allocWithZone:NULL] init];`，为什么要这样写？其实是因为本类中重写过`allocWithZone`方法，重写的方法又会调用自己，所以直接调用self的allocWithZone会导致循环，直到栈溢出
## 懒汉模式
>上面讲的饿汉模式在实现上比较简单，但这也带来了比较大的性能开销，所以有了将要讲的懒汉模式
- 懒汉模式指的是在使用的时候再创建出单例，这样就不用一开始就全局声明，大大减少了性能开销
- 但是由于这样做在多线程的情况下有可能会遇到两个线程同时创建对象的情况，这就破坏了单例的全局唯一性，所以在使用懒汉模式的时候，会使用GCD的`dispatch_once`来实现，它保证多线程调用的时候，只会调用一次初始化代码
- 我们先看一下懒汉模式的实现，再讲解能防止多线程调用的方法
```objc
+ (instancetype)sharedInstance {
    static Singletion *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[super allocWithZone: NULL] init];
    });
    return instance;
}
+ (instancetype)allocWithZone:(struct _NSZone *)zone {
    return [self sharedInstance];
}
- (id)copyWithZone:(NSZone *)zone {
    return self;
}
- (id)mutableCopyWithZone:(NSZone *)zone {
    return self;
}
```
- 我们可以明显的看到，在懒汉模式中，static被放在了创建单例的方法里面，这也就是说只有调用该方法才会创建
- 同时在sharedInstance方法内部，<font color=red>还有一个被 `static dispatch_once_t声明的onceToken`，这是懒汉模式能保证多线程调用的关键</font>，经过学长博客学习，得出这个onceToken其实就类似指示器，分以下三种情况：
1. 当onceToken为0的时候,线程执行block内的内容，同时将onceToken更改为其他值，执行完毕后，onceToken的值被更改为-1
2. 当onceToken为-1时，线程跳过block中的代码不执行
3. 当onceToken为其他值的时候说明正在被某一个线程调用，即线程阻塞，等其调用完后，onceToken也就来到了-1，当前线程就会跳过，这样就保证了多线程的情况下不会多次创建
# 单例的优缺点
## 优点
1. 单例可以实现在全局创建唯一的实例对象，可以实现全局的通信，方便传输数据
2. 通过内部的私有化，保证了数据的唯一性
3. 对于一些创建成本高的对象，只需要初始化一次，节省了大量的重复开销
## 缺点
1. 生命周期不可控，因为单例自创建开始就会一直到进程结束，不能像普通对象一样按需释放，内存始终在占用
2. 全局都能访问单例对象太好用了，导致其可能会被开发者不断地添加内容，过于臃肿，甚至出现"这个对象什么都能干什么都知道"的上帝对象情况
# 重写其他方法
## 为什么要重写
- 由于单例的全局唯一性，我们要避免在单例对象可能的一些操作中不小心对其再次创建，就比如普通的copy
- 我们如果只调用给出的获取单例方法(如上面的sharedInstance方法)还好，万一直接对这个单例类使用了`alloc, copy, mutableCopy`呢？会直接创建出新的实例对象，将会破坏单例的唯一性
- 为了避免这样的情况发生，我们就要对上面提到的三种方法重写
```objc
+ (instancetype)allocWithZone:(struct _NSZone *)zone;

- (id)copyWithZone:(NSZone *)zone;

- (id)mutableCopyWithZone:(NSZone *)zone;
```
## 如何重写
> 由于现代单例常用懒汉模式，所以本人以懒汉模式为准讲解下面的重写
- 具体重写的内容应该围绕返回当前单例对象的目的进行
1. `allocWithZone`
- 这是初始化方法，也是个类方法，所以它的重写必然要保证不会初始化出新的单例对象，所以直接调用sharedInstance方法是一个好的选择
- 写法如下
```objc
+ (instancetype)allocWithZone:(struct _NSZone *)zone {
    return [self sharedInstance];
}
```
2. `copyWithZone`
- 这是一个对象方法，也就是说，在调用这个方法前，已经安全地创建过一次单例对象了`(在保证重写了前面方法地基础上)`，<font color=red>这也就意味着static instance也已经被赋值了，所以我们直接返回这个instance就好</font>,<font color=blue>而我们又知道，self在实例方法中代表当前实例对象本身，为了防止instance变量被改名，所以这里返回self更健全</font>(当然在这里返回self和返回instance本质上一样)
- 写法如下
```objc
- (id)copyWithZone:(NSZone *)zone {
    return self;
}
```
3. `mutableCopyWithZone`
- 这里的写法和copyWithZone一样，不过多赘述
- 写法如下
```objc
- (id)mutableCopyWithZone:(NSZone *)zone {
    return self;
}
```