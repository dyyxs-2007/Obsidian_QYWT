# 基础知识
## 定义
### Block是什么
Block 是 Apple 在 C 语言的基础上添加的扩展功能，由于 Ojective-C、C++ 都是源自于 C，所以这三种语言都能够使用它。对于 Block 的功能，可以用一句话概括：**能持有作用域变量的匿名函数。**
C语言标准本身是不允许函数匿名的，但是依靠结构体的封装，我们可以实现出匿名函数的功能

事实上，Block就是OC对于闭包的实现，也给OC带来了函数式编程的可能
## 语法
Block的语法并不复杂，名为匿名函数，和普通函数的差别并不大，用法如下
- `^` `返回值类型` `参数列表` `表达式` 
其中返回值类型可以省略，下面是一些使用Block的例子
```objc
//最简形式
^{
	NSLog(@"This is a block");
}

//不省略的block
^void (void) {
	NSLog(@"This is a block");
}

//普通的block
^int (int a, int b) {
	NSLog(@"This is a block");
	return a + b;
}
```

### 作为变量

- 对于函数来说，可以把函数赋值给一个函数指针，同样的，block这种匿名函数，也可以赋值给block类型的变量，作为变量来使用：

```objc
// 完整的写法：
//// 声明
void (^simpleBlock)(void); 
//// 定义
simpleBlock = ^{
    NSLog(@"This is a block");
};
//// 声明、定义一起
void (^simpleBlock)(void) = ^{
    NSLog(@"This is a block");
};
// 调用 block
simpleBlock();
```

### 作为参数

Blcok作为参数的时候可以按照缩写规则那样省去`()`，比如

```objc
// Block 没有参数，则 () 可以省略：
[UIView animateViewDuration:5.0 animation:^() { // 这个 block 没有参数，这里的 () 就可以省略。
	view.opacity = 0.5;
}];
```

- 把block作为参数传给另一个方法时：

```objc
- (void)functionA {
    [self functionB:^(int a) {
        NSLog(@"this is block");
    }];
}

- (void)functionB:(void (^)(int))myBlock {
    myBlock(1);
}
```

苹果的建议是在一个方法中最好只使用一个 block 变量，并且如果这个方法如果还带有其他非 block 变量，那么 block 变量应该放在最后一个

### 用`typedef`简化Block定义

使用block的过程中难免会觉得语法不够简洁，用来声明变量的时候也总是写一大堆难以分清的括号，这时候`typedef`支持你把冗长的定义简化：

```objc
// typedef 一个 block
typedef void (^SimpleBlock)(void);


SimpleBlock anotherBlock = ^{
    ...
};

- (void)beginFetchWithCallbackBlock:(SimpleBlock)callbackBlock {
	...
	callbackBlock();
}
```

此时`SimpleBlock`就是一个代表了`void (^)(void)`的一个类型

- 这是一个用`typedef`简化代码的例子

```objc
// 简化前：
void (^(^complexBlock)(void (^)(void)))(void) = ^ (void (^aBlock)(void)) {
	...
	return ^{
    	...
	};
};
// 使用上面 typedef 的 SimpleBlock 简化后：
SimpleBlock (^betterBlock)(SimpleBlock) = ^ (SimpleBlock aBlock) {
    ...
    return ^{
        ...
    };
};
```

### Block被属性持有时

Block被属性持有的时候，需要使用copy属性修饰符，这样可以保证block能被拷贝到堆上而不被提前释放，关于这一点，后面会在实现中具体讲到

```objc
@interface MyObject : NSObject
@property (nonatomic, copy) void (^myBlock)(void);
@end
```

### Block被指针引用

Block可以像其他变量一样被指针引用，语法上完全一致

```objc
typedef int (^MyBlock)(int);
MyBlock aBlock = ^int(int a) {
	return a + 1;
};
MyBlock* aBlockPointer = &aBlock;
int result = (*aBlockPointer)(10); // result = 11.
```

## Block的应用场景

### 遍历数组时简化枚举

```objc
// NSArray 的一个方法，接受一个 block 作为参数，这个 block 会在该方法里每枚举一个对象时被调用一次：
- (void)enumerateObjectsUsingBlock:(void (^)(id obj, NSUInteger idx, BOOL *stop))block;
// 这个 block 接受三个参数，前两个是当前的对象和其 index，后面的 BOOL 值可以用来控制什么时候停止该枚举。
NSArray *array = ...
[array enumerateObjectsUsingBlock:^ (id obj, NSUInteger idx, BOOL *stop) {
    NSLog(@"Object at index %lu is %@", idx, obj);
}];
[array enumerateObjectsUsingBlock:^ (id obj, NSUInteger idx, BOOL *stop) {
    if (...) {
        *stop = YES;
    }
}];
```

### 简化并发调用

在GCD中常常用到Block

```objc
dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
dispatch_async(queue, ^{
    NSLog(@"Block for asynchronous execution");
});
```

### 界面间反向传值

```objc
- (void)didTapped {
  if (self.delegateBlock) {
	    self.delegateBlock()
  }
}
```



# Block的实现

- 前面的内容展示了Block的应用层，接下来将进入内部看看源码，分析一下block的原理

## 研究工具

***Clang***

为了研究编译器是如何实现 block 的，我们需要使用 clang。clang 提供一个命令，可以将 Objetive-C 的源码改写成 c 语言的，借此可以研究 block 具体的源码实现方式。该命令是

```
clang -rewrite-objc block.c
```

## Block的构成

block的数据结构定义如下：([数据来源](https://www.galloway.me.uk/2013/05/a-look-inside-blocks-episode-3-block-copy/))

![pasted-image.png · 72](https://img.cdn1.vip/i/6af5779d9bcb6_1794471837.webp)

对应的结构体结构如下

```objc
struct Block_descriptor {
    unsigned long int reserved;
    unsigned long int size;
    void (*copy)(void *dst, void *src);
    void (*dispose)(void *);
};

struct Block_layout {
    void *isa;
    int flags;
    int reserved;
    void (*invoke)(void *, ...);
    struct Block_descriptor *descriptor;
    /* Imported variables. */
};
```

首先我们可以看到block有一个isa成员，这说明block本质上是一个对象

接着还有一个成员叫`invoke`，它是一个函数指针，而block本身是一个匿名函数，所以它将真正的函数访问地址给了invoke，让它作为函数的入口

剩下的内容我们稍后进入源码中继续讲解

### 工厂模式

前面提到了isa，那它指向的到底是什么呢？

事实上，OC中的block有三大类，它们分别是

1. `_NSConcreteGlobalBlock` 全局的静态block，不访问任何变量
2. `_NSConcreteStackBlock` 保存在栈上的block，作用域结束后销毁
3. `_NSConcreteMallocBlock` 保存在堆上的block，引用计数为0时销毁

>  但是就像上面基础知识部分，我们在使用block并没有显示的声明到底是哪一种block，只是使用了block的语法，我们怎么知道有这三种类型的呢，这三种类型又有什么区别呢

我们可以借助一些代码上的小手段来挖出block的结构，使用下面这段代码可以获取到block的类

```objc
static Class _BlockClass() {
	static dispatch_once_t onceToken;
	static Class blockClass;
	dispatch_once(&onceToken, ^{
		void (^testBlock)() = [^{} copy];
		blockClass = [testBlock class];
		while(class_getSuperclass(blockClass) && class_getSuperclass(blockClass) != [NSObject class]) {
			blockClass = class_getSuperclass(blockClass);
      //在这里可以打印出遍历到的类名
		}
		[testBlock release];
	});
	return blockClass;
}
```

在Xcode中使用上述代码调试结构如下

![pasted-image.png · 1226](https://img.cdn1.vip/i/6af5811ba4ca2_1794474267.webp)

第一次循环，拿到的类名是`__NSGlobalBlock__`和上面给出的Global大致相同

再次循环结束后，父类访问到`NSObject`终止

![pasted-image.png · 1040](https://img.cdn1.vip/i/6af58143be7cf_1794474307.webp)

可以看到，最后访问的类为`NSBlock`

当我修改代码，使得block内捕获了一个变量时，用相同查看类，结果却如下

![pasted-image.png · 1074](https://img.cdn1.vip/i/6af582d06dd35_1794474704.webp) 

可以看到，此时类变成了`__NSMallocBlock__`

接着运行

![pasted-image.png · 1072](https://img.cdn1.vip/i/6af582fc479f3_1794474748.webp)

得到的父类依旧是`NSBlock`

- 可以基本上得出一个结论：

block是一个工厂类，以`NSBlock`作为父类，其拥有三个抽象基类，分别对应`Global`, `Stack`, `Malloc`

当我们使用block的时候，它会给我们分配合适的抽象基类解决不同的问题

那么，这三个类型分别有什么区别呢？

事实上，block作为一个对象，它们分别存储在静态区(Global)，栈区(Stack)和堆区(Malloc)

我们从`Global`开始，探索三种不同block的底层实现

#### _NSConcreteGlobalBlock

- 对这一段代码使用刚刚的Clang研究工具翻译为C源码

```objc
#include <stdio.h>
int main() {
	void (^blk)(void) = ^{
		int tag = 8;
		printf("Block, %d\n", tag);
	};
	blk();
	return 0;
}
```

翻译后，得到的核心代码截选如下

```objc
struct __block_impl {
	void *isa;
	int Flags;
	int Reserved;
	void *FuncPtr;
};
struct __main_block_impl_0 {
	struct __block_impl impl;
	struct __main_block_desc_0* Desc;
	__main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int flags=0) {
		impl.isa = &_NSConcreteStackBlock;
		impl.Flags = flags;
		impl.FuncPtr = fp;
		Desc = desc;
	}
};
// 这里对应的就是 block 的匿名函数。参数 __cself 为指向 block 值的指针，类似 Objective-C 中实例方法中指向对象自身的变量 self。
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
	int tag = 0;
	printf("Block, %d\n", tag);
 }
static struct __main_block_desc_0 {
	size_t reserved;
	size_t Block_size;
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0)};
int main() {
	void (*blk)(void) = (void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA);
	((void (*)(__block_impl *))((__block_impl *)blk)->FuncPtr)((__block_impl *)blk);
	return 0;
}
```

可以看到在数据结构上和图片中给出的差不多

事实上这里面的几个函数，结构体都是比较值得探讨的，我们先看看他们都是什么意思：

1. `static void __main_block_func_0(struct __main_block_impl_0 *__cself)`

   - 实际的入口函数，这里拷贝着变量

2. `struct __block_impl`结构体

   - block的核心数据存储结构体，包含了isa指针，flag(用来保存copy时需要使用的附加信息)，reserved(用于内存管理，保留变量)，FuncPtr函数的实际入口地址

3. `__main_block_desc_0`结构体

   - 这里存储了关于block的大小，保留变量等内容，用于block的复制和销毁

4. `struct __block_impl_0`结构体

   - block的外层结构体，这个结构体中包含了block的所有信息
   - 其中的

   ```objc
   __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int flags=0) {
   		impl.isa = &_NSConcreteStackBlock;
   		impl.Flags = flags;
   		impl.FuncPtr = fp;
   		Desc = desc;
   	}
   ```

   是构造函数

##### 运行流程

###### 第一阶段

以上是几个结构体的大致功能，我们姑且称之为还没组装成block的“零件”

将它们串在一起的是翻译后的main函数里的两行：

```objc
	void (*blk)(void) = (void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA);
	((void (*)(__block_impl *))((__block_impl *)blk)->FuncPtr)((__block_impl *)blk);
```

先看第一行：

```objc
void (*blk)(void) = (void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA);
```

先看等号左边，声明了一个**函数指针**，叫做blk，那么右边的部分也就是给它赋值的函数指针了，我们把右边单拿出来看

```objc
(
  void (*)()
)&__main_block_impl_0(
  										(void *)__main_block_func_0, &__main_block_desc_0_DATA
											);
```

为了看起来方便，我按照括号匹配分行，首先`void(*)()`这是一个类型，代表函数指针，那第一和第三行的括号就很显然了，是把括号后面的部分强制转换为括号中间那个函数指针，那括号后面是什么呢

```objc
&__main_block_impl_0(
  										(void *)__main_block_func_0, &__main_block_desc_0_DATA
											);
```

要知道这是什么，就要看看`__main_block_impl_0`，在上面的介绍中，这是block的所有功能集合结构体，它有一个构造函数，这里使用的就是构造函数初始化了一个`__main_block_impl_0`出来，得到的是这个类本身，用&取地址就是一个指针类型，通过前面强制转换就变成了能用的函数指针了

- 不过在这个过程中，构造函数的参数也十分重要

1. `(void*)__main_block_func_0,`，这是刚刚讲过的真实函数
2. `&__main_block_desc_0_DATA`，这个就是初始化好的block的copy，销毁信息，伴随它的生命周期

以上这两个参数都不应该由对象本身创造，所以需要在main函数中作为参数传入

而对象本身的isa指针，在这里也会自行判断，具体表现在`__main_block_impl_0`结构体中的赋值语句

###### 第二阶段

第一阶段创建出了block这个对象，接下来就要调用这个对象，即调用这个block了

于是main函数中的第二行：

```objc
((void (*)(__block_impl *))((__block_impl *)blk)->FuncPtr)((__block_impl *)blk);
```

同样的，我们把括号掰开看

```objc
(
  (void (*)(__block_impl *))
(
  (__block_impl *)blk)->FuncPtr)(
  																(__block_impl *)blk
																);
```

这次先看中间那行，` (__block_impl *)blk)->FuncPtr)`这是什么意思呢，箭头左边是一个强制转换，把我们第一阶段生成的block结构体对象blk强制转换成`__block_impl *`类型，这是为了能使用这个结构体中的具体成员(虽然结构上是同形的)，再看一眼``__block_impl`结构体的源码，其中有一个成员变量就是`FuncPtr`，其实就是真实的函数入口

但是这个变量在定义部分只是一个指向void*的指针，为了调用这个函数，需要先把这个指针强制转换成一个函数指针，也就是第一到第三行的代码，强制转换成一个函数指针，并且还是个有参数的函数指针

这个函数的参数在定义处也写的很清楚：`struct __main_block_impl_0 *__cself`，说白了，就是blk自己，所以再强制转换一下传进去就好了

这一步四舍五入可以写成这个代码

`blk()`

事实上就是调用block里面真实的函数，只是为了类型安全做了很多强制转换

#### block截获变量

刚刚分析了从写下block语法到真实函数运行的流程，下面还有一个问题没有解决，那就是block是如何捕获变量的，它是否能修改捕获的变量？

要分析这个问题，刚刚的源码不太够用，因为它的tag是在block内创建的，直接生成在真实函数内部了，捕获也就无从谈起

#### _NSConcreteStackBlock

我们将以下代码用Clang翻译一下

```objc
#include <stdio.h>
int main() {
	int dummy = 256;
	int tag = 10;
	const char *say = "tag = %d\n";
    void (^blk)(void) = ^{
        printf(say, tag);
    };
    blk();
    return 0;
}
```

翻译后核心代码

```objc
struct __block_impl {
    void *isa;
    int Flags;
    int Reserved;
    void *FuncPtr;
};
struct __main_block_impl_0 {
    struct __block_impl impl;
    struct __main_block_desc_0* Desc;
    // 这里可以看到，这里的两个 Block 结构体的成员变量与外面的两个自动变量完全相同，这是等会要来拷贝用的。
    const char *say;
    int tag;
    // 下面是结构体的构造函数。
    __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, const char *_say, int _tag, int flags=0) : say(_say), tag(_tag) {
        impl.isa = &_NSConcreteStackBlock;
        impl.Flags = flags;
        impl.FuncPtr = fp;
        Desc = desc;
    }
};
// 这里可以看到，只是把自动变量的值拷贝进来了，并且只拷贝了 Block 里用到的自动变量，如：say 和 tag，并没有拷贝 dummy。
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
    const char *say = __cself->say; // bound by copy
    int tag = __cself->tag; // bound by copy
    
    printf(say, tag);
}
static struct __main_block_desc_0 {
    size_t reserved;
    size_t Block_size;
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0)};
int main() {
    int dummy = 256;
    int tag = 10;
    const char *say = "tag = %d\n";
    
    void (*blk)(void) = (void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, say, tag);
    
    ((void (*)(__block_impl *))((__block_impl *)blk)->FuncPtr)((__block_impl *)blk);
    
    return 0;
}
```

大体上，和刚刚`Global`的内容一样，区别在于：

代表block对象的所有信息的结构体`__main_block_impl_0`中多了两个变量，它们是检测到block使用了外部声明的变量后，自动在`__main_block_impl_0`结构体中创建的同名变量，但是此时并未赋值，赋值操作在哪呢，我们看main函数的第一行中，调用该结构体的构造方法时，传入了这两个值，赋值的数据源就是main中的原始数据

同时，在真实函数`__main_block_func_0`中多了这两行代码

```objc
const char *say = __cself->say;
    int tag = __cself->tag;
```

刚刚其实说过__cself是什么，其实就是`__main_block_impl_0`结构体自己，它的构造函数被调用后，它里面的tag和say和main函数里的一样了

于是这里不难发现，在真实函数中，它又声明了同名的变量，把原来的变量值拷贝了一份，注意这里仅是值拷贝

也就是说，我们无法在内部修改一个普通的被block捕获的变量，因为真实函数调用结束后，这些变量的生命周期就结束了，改不到真实变量上

那么block捕获变量都无法改变吗，并非，只需要在前面加上`__block`就行了，那它究竟是怎么处理了变量，能在block中同步修改？

##### __block声明下的变量捕获

`__block` 说明符更准确的表述是：**__block 存储域类说明符**。在 C 语言中还有这些存储域类说明符：typedef、extern、static、auto、register。这些说明符用于指定将变量值设置到哪个存储域中。例如，auto 表示作为自动变量存储在栈上，static 表示作为静态变量存储在数据区等。下面来看看 __block

- 我们将这段代码翻译一下

```objc
#include <stdio.h>
int main() {
	int __block val = 3;
    void (^blk)(void) = ^{
		val = 300;
	};
    blk();
    return 0;
}
```

截取结果相关片段

```objc
struct __block_impl {
    void *isa;
    int Flags;
    int Reserved;
    void *FuncPtr;
};
// 这个结构体是用来定义原来的自动变量 val 的。
struct __Block_byref_val_0 {
    void *__isa;
    __Block_byref_val_0 *__forwarding; // 这个成员变量是一个指向自己实例的指针。
    int __flags;
    int __size;
    int val; // 这个成员变量将用来存原来自动变量的值。
};
struct __main_block_impl_0 {
    struct __block_impl impl;
    struct __main_block_desc_0* Desc;
    // 原来的自动变量被重新定义成了一个结构体，val 成了指向这个结构体的一个实例的指针。
    __Block_byref_val_0 *val; // by ref
  
    __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, __Block_byref_val_0 *_val, int flags=0) : val(_val->__forwarding) {
        impl.isa = &_NSConcreteStackBlock;
        impl.Flags = flags;
        impl.FuncPtr = fp;
        Desc = desc;
    }
};
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
    __Block_byref_val_0 *val = __cself->val; // bound by ref
    
    // 下面的赋值可以看出，是把原自动变量变成了 __main_block_impl_0 结构体的实例的指针。再通过指针去访问这个实例的成员变量。其中成员变量 __forwarding 指向实例自己，成员变量 __val 存储值。
    (val->__forwarding->val) = 300;
}
static void __main_block_copy_0(struct __main_block_impl_0*dst, struct __main_block_impl_0*src) {_Block_object_assign((void*)&dst->val, (void*)src->val, 8/*BLOCK_FIELD_IS_BYREF*/);}
static void __main_block_dispose_0(struct __main_block_impl_0*src) {_Block_object_dispose((void*)src->val, 8/*BLOCK_FIELD_IS_BYREF*/);}
static struct __main_block_desc_0 {
    size_t reserved;
    size_t Block_size;
    void (*copy)(struct __main_block_impl_0*, struct __main_block_impl_0*);
    void (*dispose)(struct __main_block_impl_0*);
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0), __main_block_copy_0, __main_block_dispose_0};
int main() {
	// 可以看到，变量 val 竟然成为了结构体 __Block_byref_val_0 的实例。
    __attribute__((__blocks__(byref))) __Block_byref_val_0 val = {(void*)0,(__Block_byref_val_0 *)&val, 0, sizeof(__Block_byref_val_0), 3};
    
    void (*blk)(void) = (void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, (__Block_byref_val_0 *)&val, 570425344);
    
    ((void (*)(__block_impl *))((__block_impl *)blk)->FuncPtr)((__block_impl *)blk);
    
    return 0;
}
```

上面的代码有两个函数需要说明一下：`_Block_object_assign` 函数实现把 Block 从栈上拷贝到堆上；`_Block_object_dispose` 函数实现把 Block 从堆上释放

也就是`__main_block_copy_0`和`__main_block_dispose_0`的函数体，关于copy这一点我们后面再讲

再次观察这个相似的代码，我们看到被`__block`修饰的val变量，在`__main_block_impl_0`中变成了`__Block_byref_val_0`类型的同名指针，也就是一个新的结构体指针，我们再进入这个结构体内看看细节：

```objc
struct __Block_byref_val_0 {
    void *__isa;
    __Block_byref_val_0 *__forwarding; // 这个成员变量是一个指向自己实例的指针。
    int __flags;
    int __size;
    int val; // 这个成员变量将用来存原来自动变量的值。
};
```

首先它也有一个isa指针，说明它也是一个对象，同时它有一个size成员，用来拷贝，以及一个拷贝中会用到的flags成员

比较重要的是val和forwarding，val其实也是一个它自己声明的变量，与外部捕获的变量同名

而`__forwarding`则是一个指向自己的指针，这个指针的妙用一会会提到

而__block到底是怎么实现在block内部修改外部变量的呢？关键在于mian函数中的这句

`__attribute__((__blocks__(byref))) __Block_byref_val_0 val = {(void*)0,(__Block_byref_val_0 *)&val, 0, sizeof(__Block_byref_val_0), 3};`

我们依旧拆开看：

```objc
__attribute__((__blocks__(byref))) __Block_byref_val_0
  val = {
  				(void*)0,
  					(__Block_byref_val_0 *)&val, 
  					0, 
						sizeof(__Block_byref_val_0), 3
			};
```

第一行还是一个类型转换，转换为一个`__Block_byref_val_0`的类型

类型转换的是原本声明的一个int类型的val，但是，怎么能把int类型变成这个结构体的呢？事实上它是直接生成的`__Block_byref_val_0`结构体，把原本的int值作为一个同名成员变量保存下来了

这样block内部就通过`__main_block_impl_0`新加的`__Block_byref_val_0`字段访问到val真实值，block外部也能通过val变量本身(已经被修改为结构体)来访问到它内部的val真实值

但是还有个问题，在真实函数`static void __main_block_func_0(struct __main_block_impl_0 *__cself)`中，为什么会有这样一句

```objc
    (val->__forwarding->val) = 300;
```

直接访问val不就行了吗，先访问一下自己再取值不是多此一举吗

![pasted-image.png · 45](https://img.cdn1.vip/i/6af595af96574_1794479535.webp)

**回答这个问题之前，我们需要先介绍一下堆区存储的`_NSConcreteMallocBlock`**

#### _NSConcreteMallocBlock

刚刚我们见过了存储在栈上和静态区的block，什么时候会用到堆区的block呢

刚刚提到过，当我们捕获了一个变量的时候，如果是栈区block，在函数作用域结束后，它会自行销毁，但如果我们想在函数结束后继续使用block，比如作为参数把block传给另一个函数的时候，单纯的栈区不够用了

block提供了Block 和 __block 变量从栈上复制到堆上的方法来解决这个问题，将存储在栈上的 Block 复制到堆上，即使 Block 语法记述的变量作用域生命周期结束，堆上的 Block 还是可以继续存在。复制到堆上的 Block 将 `_NSConcreteMallocBlock` 类对象写入 Block 结构体实例的 isa 成员变量，而 __forwarding 成员变量就可以实现无论 __block 变量存储在栈上还是堆上都能准确的被访问。

![pasted-image.png · 77](https://img.cdn1.vip/i/6af596a7c705b_1794479783.webp)

##### block的拷贝

而拷贝的写法很简单，返回的时候写一句`[blk coopy]`即可，事实上，它会调用刚刚源码中的这个函数

```objc
static void __main_block_copy_0(struct __main_block_impl_0*dst, struct __main_block_impl_0*src) {_Block_object_assign((void*)&dst->val, (void*)src->val, 8/*BLOCK_FIELD_IS_BYREF*/);}
```

而这个函数最终会调用这个函数

```objc
static void *_Block_copy_internal(const void *arg, const int flags) {
    struct Block_layout *aBlock;
    const bool wantsOne = (WANTS_ONE & flags) == WANTS_ONE;

    // 1
    if (!arg) return NULL;

    // 2
    aBlock = (struct Block_layout *)arg;

    // 3
    if (aBlock->flags & BLOCK_NEEDS_FREE) {
        // latches on high
        latching_incr_int(&aBlock->flags);
        return aBlock;
    }

    // 4
    else if (aBlock->flags & BLOCK_IS_GLOBAL) {
        return aBlock;
    }

    // 5
    struct Block_layout *result = malloc(aBlock->descriptor->size);
    if (!result) return (void *)0;

    // 6
    memmove(result, aBlock, aBlock->descriptor->size); // bitcopy first

    // 7
    result->flags &= ~(BLOCK_REFCOUNT_MASK);    // XXX not needed
    result->flags |= BLOCK_NEEDS_FREE | 1;

    // 8
    result->isa = _NSConcreteMallocBlock;

    // 9
    if (result->flags & BLOCK_HAS_COPY_DISPOSE) {
        (*aBlock->descriptor->copy)(result, aBlock); // do fixup
    }

    return result;
}
```

[代码来源](https://www.galloway.me.uk/2013/05/a-look-inside-blocks-episode-3-block-copy/)

我们可以看到，在这个过程中，不件事所有内容都被拷贝了一份，isa指针也被改为了`_NSConcreteMallocBlock`，同时被`__block`修饰的变量生成的结构体也被拷贝了一份

值得注意的是，这里并没有销毁原本的block，也就是说，此时栈上的block和堆上的block同时存在

**事实上，在现代ARC中，当一个栈区 block 被赋值给一个强引用变量(__strong)时，ARC 会自动插入 block = [block copy];，把它拷贝到堆上，变成 MallocBlock，基本等同于仅有MallocBlock的存在**

这就引出了刚刚那个问题的答案：为什么需要有一个`__forwarding`指针指向`__block`修饰的变量自己

##### __forwarding真正存在的原因

刚刚提到，copy过后在堆和栈上会有两套完全一样的block存在，对于作用域未结束的函数来说，它自己访问的是栈区的block的val结构体，然而block内部访问的却是堆上的block的val

这就导致block内部修改了val，但是函数内部却看不见，不符合我们想修改变量的原则

问题出在哪？其实就在于栈区的block的`__main_block_impl_0`结构体中的val结构体是栈区的，但是block访问的是堆区的val结构体

那在拷贝block的时候直接把栈区的block的val结构体的`__forwarding`成员指向堆区的val结构体就好了，这样无论是block内部还是block外部访问的都是堆区的val结构体

那可能有人又会问，为什么不直接把val指向堆区，而是把val的`__forwarding`指向堆区的val，这不绕圈了吗，注意，block内部的val并不是指针，而是一个写死的结构体，所以只能改指针了

不过这时候还有一个小细节需要注意，那就是栈区的block的`__main_block_impl_0`中存的`val`还是栈区的结构体，所以使用的时候需要先访问栈区的val结构体，再通过它的`__forwarding`属性访问到堆区的val结构体，再访问到val的真实值

也就有了这样看起来奇怪的代码` (val->__forwarding->val) = 300;`

![pasted-image.png · 125](https://img.cdn1.vip/i/6af59cb444073_1794481332.webp)

- 当堆上的block被销毁时，它的__block 变量也会被一块销毁

![pasted-image.png · 171](https://img.cdn1.vip/i/6af59cce4e594_1794481358.webp)

可以看到，以reserved计数释放内存的方法和OC的引用计数如出一辙

# 使用Block需要注意的问题

## 使用C语言数组

使用c语言数组的时候可能会出现一些意想不到的问题，比如下面的例子

```objc
// 这样是有问题的：
const char text[] = "hello";
void (^blk)(void) = {
	printf("%c\n", text[2]);
}
// 这样是没问题的：
const char* text = "hello";
void (^blk)(void) = {
	printf("%c\n", text[2]);
}
```

这个问题的原因在于block内部对变量的生成方式，被block转换后的函数可能是这样的

```objc
void func(const char text[]) {
	const char text2[] = text; // 这种赋值方式是不符合 C 语言规范的。
	printf("%c\n", text[2]);
}
int main() {
	cosnt char text[] = "hello";
	func(text);
}
```

## Block对ARC环境下对象的持有

先写一个类用于后面持有情况测试

```objc
// MyObject.h
#import <Foundation/Foundation.h>
@interface MyObject : NSObject
@property (nonatomic, strong) NSString *text;
@end
// MyObject.m
#import "MyObject.h"
@implementation MyObject
#pragma mark - Property
- (NSString *)description {
    return _text;
}
@end
```

### __weak修饰

```objc
// ViewController.m
#define PrintObject(prefix, obj) { NSLog(@"%@: (指针内存地址: %p, 指针值: %p, 指向的对象值: %@)", prefix, &obj, obj, obj); }
- (void)viewDidAppear:(BOOL)animated {
    [super viewDidAppear:animated];
    
    typedef void (^MyBlock)();
    
    MyBlock blk;
    
    {
        MyObject *obj = [[MyObject alloc] init];
        obj.text = @"I-am-an-obj.";
        PrintObject(@"obj", obj);
        
        typeof(obj) __weak weakObj = obj;
        PrintObject(@"weakObj", weakObj);
        
        blk = ^() {
            PrintObject(@"weakObj in Block", weakObj);
        };
        
        blk();
    } // obj、weakObj 的作用域结束。
    
    blk();	    
}
```

程序打印出的内容如下：

```objc
obj: (指针的内存地址: 0x7fff5e514120, 指针值: 0x7ff56c1003e0, 指向的对象值: I-am-an-obj.)
weakObj: (指针的内存地址: 0x7fff5e514118, 指针值: 0x7ff56c1003e0, 指向的对象值: I-am-an-obj.)
weakObj in Block: (指针的内存地址: 0x7ff56bf1cce0, 指针值: 0x7ff56c1003e0, 指向的对象值: I-am-an-obj.)
weakObj in Block: (指针的内存地址: 0x7ff56bf1cce0, 指针值: 0x0, 指向的对象值: (null))
```

结论：

- `obj`、`Block 外部的 weakObj`、`Block 内部的 weakObj` 是 3 个不同的指针变量，但它们都指向了同样的对象。
- Block 捕获 `weakObj` 指向的对象时是对其进行**弱引用持有**，这是因为 `weakObj` 是用 `__weak` 修饰的。当 `obj` 的作用域结束时，它指向的对象没有被其他强引用持有，所以立即被释放，这时 Block 内部持有的弱引用 `weakObj` 也被设置为 nil。

### __strong修饰

```objc
- (void)viewDidAppear:(BOOL)animated {
    [super viewDidAppear:animated];
    typedef void (^MyBlock)();
    
    MyBlock blk;
    
    {
        MyObject *obj = [[MyObject alloc] init];
        obj.text = @"I-am-an-obj.";
        PrintObject(@"obj", obj);
        
        typeof(obj) __strong strongObj = obj;
        PrintObject(@"strongObj", strongObj);
        
        blk = ^() {
            PrintObject(@"strongObj in Block", strongObj);	        };
        
        blk();
    } // obj、strongObj 的作用域结束。
    
    blk();
}
```

输出如下

```objc
obj: (指针的内存地址: 0x7fff54284120, 指针值: 0x7f9012c23e70, 指向的对象值: I-am-an-obj.)
strongObj: (指针的内存地址: 0x7fff54284118, 指针值: 0x7f9012c23e70, 指向的对象值: I-am-an-obj.)
strongObj in Block: (指针的内存地址: 0x7f9012d1cc20, 指针值: 0x7f9012c23e70, 指向的对象值: I-am-an-obj.)
strongObj in Block: (指针的内存地址: 0x7f9012d1cc20, 指针值: 0x7f9012c23e70, 指向的对象值: I-am-an-obj.)
```

结论：

- `obj`、`Block 外部的 strongObj`、`Block 内部的 strongObj` 是 3 个不同的指针变量，但它们都指向了同样的对象。
- Block 捕获 `strongObj` 指向的对象时是对其进行**强引用持有**，这是因为 `strongObj` 是用 `__strong` 修饰的。当 `obj`、`Block 外部的 strongObj` 的作用域结束时，它们指向的对象依然被 Block 强引用持有了，所以不会立即释放。

### __block修饰

```objc
- (void)viewDidAppear:(BOOL)animated {
    [super viewDidAppear:animated];
    
    typedef void (^MyBlock)();
    MyBlock blk;
    
    {
        MyObject *obj = [[MyObject alloc] init];
        obj.text = @"I-am-an-obj.";
        PrintObject(@"obj", obj);
        
        typeof(obj) __block blockObj = obj;
        PrintObject(@"blockObj", blockObj);
        
        blk = ^() {
            PrintObject(@"blockObj in Block", blockObj);
        };
        
        blk();
    } // obj、blockObj 的作用域结束。
    
    blk();
}
```

输出如下：

```
obj: (指针的内存地址: 0x7fff57581120, 指针值: 0x7f9eac800440, 指向的对象值: I-am-an-obj.)blockObj: (指针的内存地址: 0x7fff57581118, 指针值: 0x7f9eac800440, 指向的对象值: I-am-an-obj.)blockObj in Block: (指针的内存地址: 0x7f9eac800e28, 指针值: 0x7f9eac800440, 指向的对象值: I-am-an-obj.)blockObj in Block: (指针的内存地址: 0x7f9eac800e28, 指针值: 0x7f9eac800440, 指向的对象值: I-am-an-obj.)
```

结论：

- 在 ARC 环境下，`__block` 的表现看起来跟 `__strong` 没有什么区别，在 Block 中都对对象做了**强引用持有**。但是需要注意的是在 MRC 环境下，`__block` 的表现是不一样的，在 Block 中不会对对象进行 retain。

### __ weak与 __strong混搭

```objc
- (void)viewDidAppear:(BOOL)animated {
    [super viewDidAppear:animated];
    typedef void (^MyBlock)();
    
    MyBlock blk;
    
    {
        MyObject *obj = [[MyObject alloc] init];
        obj.text = @"I-am-an-obj.";
        PrintObject(@"obj", obj);
        
        typeof(obj) __weak weakObj = obj;
        PrintObject(@"weakObj", weakObj);
        
        blk = ^() {
            typeof(weakObj) __strong strongObj = weakObj;
            PrintObject(@"weakObj in Block", weakObj);
            PrintObject(@"strongObj in Block", strongObj);
        };
        
        blk();
    } // obj、weakObj 的作用域结束。
    
    blk();
}
```

这个例子中，block内部使用strong持有了外部了weak修饰的对象

- 输出如下：

```objc
obj: (指针的内存地址: 0x7fff54625120, 指针值: 0x7fb832210110, 指向的对象值: I-am-an-obj.)
weakObj: (指针的内存地址: 0x7fff54625118, 指针值: 0x7fb832210110, 指向的对象值: I-am-an-obj.)
weakObj in Block: (指针的内存地址: 0x7fb832210010, 指针值: 0x7fb832210110, 指向的对象值: I-am-an-obj.)
strongObj in Block: (指针的内存地址: 0x7fff54625048, 指针值: 0x7fb832210110, 指向的对象值: I-am-an-obj.)
weakObj in Block: (指针的内存地址: 0x7fb832210010, 指针值: 0x0, 指向的对象值: (null))
strongObj in Block: (指针的内存地址: 0x7fff54625048, 指针值: 0x0, 指向的对象值: (null))
```

结论：

- `obj`、`Block 外部的 weakObj`、`Block 内部的 weakObj`、`Block 内部的 strongObj` 是 4 个不同的指针变量，但它们都指向了同样的对象。
- Block 捕获 `weakObj` 指向的对象时是对其进行**弱引用持有**，这是因为 `weakObj` 是用 `__weak` 修饰的。这与上面第 1） 种情况是一样的，但是这里在 Block 的实现中，又通过 `typeof(weakObj) __strong strongObj = weakObj;` 这句代码想要去强引用持有 `weakObj` 指向的对象，但是在执行这句代码之前，如果 `weakObj` 已经被置为 nil，那么这里的 `strongObj` 也并不能强引用持有住对象而延长对象的生命周期。这里很有意思的一点是，这句代码在 `weakObj` 这个中间人的作用下并没有一开始就强引用持有 `obj` 指向的对象，而是等到代码执行到这句才开始，并且这里是通过一个**局部变量** `strongObje` 来强引用持有对象，当局部变量作用域结束，它和它所持有的对象就都会被释放，不会造成内存泄露的问题。这点非常有用，即它不会造成强引用循环，这在后面「避免 Block 使用的对象被提前释放」那节会有用到。

## 循环引用

循环引用指的是block对self强引用，并且self对block也强引用，

```objc
// MyObject.h
#import <Foundation/Foundation.h>
@interface MyObject : NSObject
@property (nonatomic, copy) void ^myBlock(void) 
@end
// MyObject.m
#import "MyObject.h"
@implementation MyObject
 
- (void)test {
  	self.myBlock = ^{
      	[self class];
    }
} 
@end
```

这段代码中，self有一个myblock属性，是self对block的强引用，而block中有self，所以就是谁也不能释放谁

为了解决这个问题，有下面几个方案

### __ weak与 __strong

刚刚介绍过这个混搭，日常开发中的常用解法，既能保证循环引用被打破，也能保证block内的代码不被随意释放

```objc
__weak typeof(self) weakSelf = self;
self.myBlock = ^{
    __strong typeof(weakSelf) strongSelf = weakSelf;
    if (!strongSelf) return;
    [strongSelf doSomething];
};
```

### 将self作为参数传入block

这种做法的原理是block不通过捕获self来调用self相关代码，而是把self作为block的一个参数，即只读

这样block就不持有self，引用环就被破解了

```objc
// 1. 定义时，把控制器作为参数传进来，不在内部直接写 self
self.myBlock = ^(MyViewController *vc) {
    [vc doSomething];
};

// 2. 调用时把 self 传进去
self.myBlock(self);
```

### 手动置空指针

```objc
__block typeof(self) blockSelf = self;
self.myBlock = ^{
    [blockSelf doSomething];
    blockSelf = nil; // 手动置为 nil！
};
```

既然循环引用是两者互相持有，谁也无法释放对方，那加入第三者，形成一个三角引用，这样断掉其中一环不就能解除循环引用了吗

于是这里引入了第三者`blockSelf`，由于用__block修饰过，此时的blockSelf已经变成了一个指向`__Block_byref_val_0`结构体的指针，而这个结构体内部持有self，而self本身持有myBlock，myBlock内部又捕获了blockSelf，此时我们在block实现的最后一步，吧中间者`blockSelf`置空，就能断掉引用环

### 销毁Block

这种方法就是在用完block之后直接把它销毁(置空)，但是局限性较大，仅适用于只调用一次的回调

```objc
self.completionBlock = ^{
    [self doSomething];
};

//此时用到了block
if (self.completionBlock) {
    self.completionBlock();
    self.completionBlock = nil; //用完直接置空
}
```

