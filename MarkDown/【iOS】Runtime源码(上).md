# 引言

起初学OC最先接触到的就是OC看似奇怪的消息发送语法，也就是`[receiver, selector]`，当时的理解就是调用`receiver`的`selector`方法，只是冠以**“消息发送”**这个优雅的名字，接触Runtime之后才发现没有理解消息发送这个名字的真实含义；

其实我们平时使用`[receiver, selector]`的时候，编译后会变成

```objc
objc_msgSeng(receiver, selector)
```

如果消息中还有参数，就会变成

```objc
objc_msgSeng(receiver, selector, arg1, arg2, ...)
```

接着`Runtime`会先顺着`receiver`的**isa指针**进入`receiver`的`Class`中，再查找`Class`的方法列表，没找到就顺着`superClass`，直到找到`NSObject`，进入方法解析，进入消息转发

现在可以看出`[receiver, selector]`真的不仅仅是一句方法调用，因为这只是在编译阶段确定了要给谁发送什么消息，**而receiver要怎么响应这条消息，要在运行时决定是自己做还是临时加，或者是转发给别人**

Objective-C 的 Runtime 铸就了它动态语言的特性，这些知识内容较深，平时用的少，但是对于iOS开发者而言却是需要掌握的

# 简介

OC是一门动态语言，很多工作需要推到代码真正运行的时候才能做，这时候仅仅在编译时就确定代码就显得有些不够用了，于是还需要有一个运行时系统`(runtime system)`来执行编译后的代码，这就是OC的`Runtime`，是整个OC动态特性的基石

Runtime基本上是用C和汇编写的，上面介绍的`objc_msgSeng(receiver, selector)`的实现就是通过汇编，可见苹果为了维持这套动态系统的高效性的努力，这是源码中的相关定义，具体实现在.s的汇编文件中

![pasted-image.png · 12](https://img.cdn1.vip/i/6aae8bbd86ae5_1789823933.png)

这里有一份[编译好的 Runtime 源码](https://github.com/RetVal/objc-runtime/tree/objc.750)，可以下载下来自己瞅瞅

# Runtime基础数据结构

这里先列举一下比较重要的一些基础概念

- `SEL`：方法选择器，全名是 `selector`
- `id`：是一个参数类型，指向某个类实例的指针
- `Class`：指向了 `objc_class` 结构体的指针
- `Method`：代表了类中的某个方法的类型
- `Ivar`：成员变量的类型
- `IMP`：函数指针，由编译器生成，方法实现的代码就是由 `IMP` 指定
- `Cache`：方法调用的缓存器，为方法调用的性能进行优化
- `Property`：属性存储器

还记得刚刚举的例子`objc_msgSeng(receiver, selector)`，其实它的真身苹果在文档中是这样写的`id objc_msgSend(id self, SEL _cmd, ...);`，这里的`id`，`SEL`都是用C语言实现的，接下来将会向你一步步揭开这些内容的面纱

## id与Class

我们先来看一下源码(简化版)

```objc
typedef struct objc_object *id;

typedef struct objc_class *Class;

struct objc_object {
    isa_t isa;
};

struct objc_class : objc_object {
    // Class isa;
    Class superclass;
    cache_t cache;             // 方法缓存
    class_data_bits_t bits;    // 用于获取具体的类信息
    ...
}
```

这里我们先从`id`看起，`id`是什么，是一个指向`objc_object`结构体的指针

而`objc_object`结构体中有且仅有一个` isa`，它的类型是`isa_t`，这是它的源码

```objc
union isa_t {
    isa_t() { }
    isa_t(uintptr_t value) : bits(value) { }

    uintptr_t bits;

private:
    // Accessing the class requires custom ptrauth operations, so
    // force clients to go through setClass/getClass by making this
    // private.
    Class cls;

public:
#if defined(ISA_BITFIELD)
    struct {
        ISA_BITFIELD;  // defined in isa.h
    };

#if ISA_HAS_INLINE_RC
    bool isDeallocating() const {
        return extra_rc == 0 && has_sidetable_rc == 0;
    }
    void setDeallocating() {
        extra_rc = 0;
        has_sidetable_rc = 0;
    }
#endif // ISA_HAS_INLINE_RC
#endif
    void setClass(Class cls, objc_object *obj);
    Class getClass(bool authenticated) const;
    Class getDecodedClass(bool authenticated) const;
};
```

可以看到它是一个联合体，存储着一个`Class`类型的`cls`

而`Class`则是一个`objc_class`结构体的指针，从源码中可以看到，这个`objc_class`是继承于`objc_object`的，也就是说`objc_class`中本身就因为继承而包含一个`isa`指针，指向的就是`objc_class`这个结构体

> 这里的逻辑比较不好想，甚至乍一看会有鸡生蛋问题的感觉，但事实上objc_class的创建是不依赖于isa指针的，所以它可以直接被直接创建出来，于是objc_objct再拥有指向它的指针

- class和id的关系

`objc_object` 用来描述 OC 中的实例，当用口语描述实例时，总会说 「XX类的实例x」或「x是XX的实例」；`objc_object` 的 `isa` 在程序结构上表达类似的含义，它指向了该实例所对应的类，类在 runtime 中被定义成 `objc_class` 结构

从刚刚的源码又看到，`objc_class`继承于`objc_object`，于是它也有自己的`isa`，指向它的元类，对于元类而言，类本身也是一个对象；

同时我们注意到，`objc_class` 拥有一个 `superclass` 成员变量，它指向该类的父类，也就是展示了继承关系，也就给实例，类对象，元类三者建立了一个闭环的关系

![pasted-image.png · 857](https://img.cdn1.vip/i/6aae8f82a5910_1789824898.webp)

来看这张图，上方`objc_class`就是我们平时使用的类，其中存在`isa`指向类的类，还有指向父类的`superclass`，这两者关系其一为继承，另一为实例，并不是同一个层面

接着就是方法缓存`cache`，向一个类调用方法的时候会先检查方法缓存中能不能直接找到，如果找不到，再顺着isa找到派生它的类，在它的类的方法列表中寻找方法

`bits`存储的是一个类的核心内容，bits指向的`class_data_bits_t`中包含一个指向`class_rw_t`的指针，它包含了一个类的所有信息，如`method_list_t`类型的方法列表，还有属性列表

其中`class_rw_t`还有一个指向`class_ro_t`的ro指针，rw代表`read-write`，ro代表`read-only`，从源码中也能看出，只读的ro中存储的是一个类的基本信息，比如类名，占用空间等等，而可读写的rw中则有可以动态添加方法的方法列表等其他属性

## 元类(Meta Class)

先看一个例子

```objc
NSArray *array = [NSArray array];
```

这个例子中，`+array` 消息发送给了 `NSArray` 类，而这个 `NSArray` 是一个类。前面提到，类都是用`objc_class`定义的，所以它一定有一个`isa`指针，那么就有一个问题了，这个 `isa` 指向的是什么呢？为了调用 `+array` 方法，这个类的 `isa` 指针必须指向一个包含这些类方法的一个 `objc_class` 结构体。这就引出了 `meta-class` 的概念

> Meta Class是一个类对象的类

这里指的类对象，就是我们平时使用的类，对类方法发消息的那个类

当我们向一个对象发送消息时，Runtime 会在这个对象所属的这个类的方法列表中查找方法；而向一个类发送消息时，会在这个类的 `meta-class` 的方法列表中查找，其实本质就是先顺着isa指针进入它的类，然后再找方法

一个对象的类中存储这这个对象的所有实例方法，比如我们对上面声明出来的array调用`[array count]`，其实是进入array这个实例的类中找到的方法

一个类本身也有isa指针，同样可以被称为对象，这个特殊对象的所有实例方法都存在它的isa指向的类——元类中

所以当我们对一个类发消息，调用类方法，**换一个说法就是对类这个元类的实例对象，调用它的实例方法**，于是顺理成章地顺着isa进入元类中，找到了类对象的方法然后调用

这样就在结构上对`对象<--类，类<--元类`完成了一个统一

再深入一下，元类其实同样也是一个`objc_class`类型的类，它按理也有一个isa，那它的isa指向谁呢，总不能这样无限套娃下去，苹果给出的结果是元类的isa指向元类的父类的源头——根元类(虽然这个根元类的父类还是NSObject)

由于我们平时只需要用到类和实例，对根元类的用处几乎没有，于是根元类仅仅是作为一种结构上的统一化处理的手段在幕后工作，所以元类的isa可以这样处理

来看一下这个十分经典的图

![pasted-image.png · 122](https://img.cdn1.vip/i/6aae9c53c4b53_1789828179.webp)

它很好的诠释了刚刚我们讲解的实例，类，元类之间的关系

## class_rw_t

刚刚展示的`objc_class`的源代码中，有一个指向`class_rw_t`结构体的指针，它的定义大致如下

```objc
struct class_rw_t {
    uint32_t flags;
    uint32_t version;

    const class_ro_t *ro;

    method_array_t methods;
    property_array_t properties;
    protocol_array_t protocols;
}
```

其中，`const class_ro_t *ro`包含了所有**成员变量**的一维数组以及**各个基础的属性、方法和协议**
`method_array_t methods`、`property_array_t properties`、`protocol_array_t protocols`则包含了所有**拓展的属性方法和协议**,**注意：这里的属性只是getset方法，不包含成员变量。**

> 拓展(category)的属性这里并没有提到，所以不能在category中添加成员变量，但是完全用runtime创建的类是可以动态添加的

## SEL

`objc_msgSend`函数第二个参数类型为`SEL`，它是`selector`在Objc中的表示类型（Swift中是`Selector`类）。`selector`是方法选择器，可以理解为区分方法的 ID，而这个 ID 的数据结构是`SEL`:

```objc
typedef struct objc_selector *SEL;
```

其实它就是个映射到方法的C字符串的结构体，你可以用 Objc 编译器命令 `@selector()` 或者 Runtime 系统的 `sel_registerName` 函数来获得一个 `SEL` 类型的方法选择器。

不同类中相同名字的方法所对应的方法选择器是相同的，即使方法名字相同而变量类型不同也会导致它们具有相同的方法选择器，于是 Objc 中方法命名有时会带上参数类型

## IMP

`IMP`其实就是一个void返回值的函数指针，用于存放背后调用函数的内存地址，当真正需要执行对应函数时会通过IMP快速调用，源码中它被这样定义

![pasted-image.png · 23](https://img.cdn1.vip/i/6aafefab1e742_1789915051.png)

## Method

![pasted-image.png · 32](https://img.cdn1.vip/i/6aaff30ab139a_1789915914.webp)

这是method源码的小部分内容

可以看到它主要持有三个内容：`SEL`,`types`,`IMP`这里就把上面提到的`SEL`和`IMP`联系起来了

`types` 包含了函数返回值，参数编码的字符串，编码字符串对应的意义具体可以查看[这里](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtTypeEncodings.html#//apple_ref/doc/uid/TP40008048-CH100-SW1)。

那么method有什么用呢？method是作为存储方法名与方法对应函数的实际内存地址的数据结构存在的，它把每个方法封装起来，作为一个最小单位出现

前面提到的objc_`class`的`class_data_bits_t`字段中的方法列表`method_list_t`结构体中，储存的就是`method`

这一条具体对于消息发送的重要性在下面`objc_msgSend`方法里面会介绍

## Cache_t

`Cache_t`的定义：

```objc
struct cache_t {
    struct bucket_t *_buckets;  // 散列表
    mask_t _mask;   // 散列表的长度 -1
    mask_t _occupied;   // 已经缓存的方法数量
}

// bucket_t
struct bucket_t {
    cache_key_t _key;  // SEL 作为 key
    IMP _imp;   // 函数的内存地址
}
```

### buckets:

这是缓存的核心，是指向 Method 数据结构指针的数组，每一个指针指向一个存着`SEL(key)`和`IMP(value)`的键值对结构体

需要注意的是，指针可能是 NULL，表示这个缓存 bucket 没有被占用，另外被占用的 bucket 可能是不连续的，原因是当函数要加入缓存的时候会进行一次哈希算法，根据得到的下标放入数组

>  缓存的意义：`objc_class`的方法列表中存着所有方法，但是调用的时候却是遍历进行的，这样有性能上的开销，但是如果把一些常用函数放进一个哈希表里，这样就能实现O(1)的查找效率

### 查找流程

当进入缓存开始查找方法的时候，会进行一个叫做`cache loopup`的流程：

1. `mask = cache.mask();`先得到散列表的长度总数
2. `index = (uintptr_t)sel & mask;`用传入的`SEL`在进行哈希算法
3. 执行`bucket = cache.buckets() + index;`，这是一个指针加法，由于`buckets`是数组首地址指针，所以这样就相当于访问`buckets[index]`如果这样能访问到合法的`bucket`，此时缓存命中，直接返回`buckets`中的`IMP`真实函数内存地址
4. 如果未访问到，`Cache miss`，进入方法列表查找

> 铺垫了这么多终于能开始讲objc_msgSend函数了

# objc_msgSend函数

看起来像是`objc_msgSend`返回了数据，其实`objc_msgSend`从不返回数据而是你的方法被调用后返回了数据。下面详细叙述下消息发送步骤：

1. 检测这个 `selector` 是不是要忽略的。比如ARC下就不理会 `retain`, `release` 这些函数了
2. 检测这个 `target` 是不是 `nil` 对象。OC的特性是允许对一个 `nil`对象执行任何一个方法不会 Crash，因此会被忽略掉。
3. 如果上面两个都过了，那就开始查找这个类的 `IMP`，先从 `cache` 里面找，完了找得到就跳到对应的函数去执行。
4. 如果 `cache` 找不到就找一下方法列表。
5. 如果分发表找不到就到父类的列表去找，一直找，直到找到`NSObject`类为止。
6. 如果还找不到就要开始进入动态方法解析了，尝试添加方法或者重定向

并且`objc_msgSend`还有三个不同形式，分别是`objc_msgSend`, `objc_msgSend_stret`, `objc_msgSendSuper`, 和 `objc_msgSendSuper_stret`

编译器会根据不同情况选择这四个函数中的其中一个调用，如传递消息给父类，就会调用带有`super`的函数，返回值如果不是一个普通值而是数据结构的时候就调用带有`stret`的函数

## Super

当我们调用`[super selector]` 时，Runtime 会调用 `objc_msgSendSuper`方法，`objc_msgSendSuper` 方法有两个参数，`super` 和 `op`，`Runtime` 会把 `selector` 方法选择器赋值给 `op`。而 `super` 是一个 `objc_super` 结构体指针，`objc_super` 结构体定义如下：

```objc
struct objc_super { 
  id receiver; 
  Class super_class;
};
```

需要注意的是，这里的`receiver`仍然是调用者自己，也就是`self`，

举个例子

```objc
@implementation Son : Father
- (id)init
{
    self = [super init];
    if (self)
    {
        NSLog(@"%@", NSStringFromClass([self class]));
        NSLog(@"%@", NSStringFromClass([super class]));
    }
    return self;
}
@end
```

> 你是否会觉得会打印一个Son打印一个Father

答案是全部输出`Son`

原因是当调用`[super class]`的时候，要经历以下几轮调用

1. 创建`super`结构体`objc_super`，其中的`receiver`字段其实用的是`self`赋值的，`super_class`字段是用`class_getSuperclass(object_getClass(self))`赋值的，简化调用流程就是：`self->isa->superClass`获得父类对象
2. 原消息转变为对`objc_msgSendSuper2(&super, @selector(class))`，注意这里的参数一是刚刚生成的`super`结构体的指针

- **我们再来仔细研究一下这句`objc_msgSendSuper(&super, @selector(class))`的调用逻辑：**

1. 首先会从`super`结构体中取出`super_class`字段，这里获得父类对象
2. 取出第二个参数`@selector(class)`得到`class`方法的`SEL` (这里调用`@selector(class)`也是散列表查找)，先持有
3. 进入`super_class`的`Cache`，进行缓存查找，这一步的具体情况上面`Cache loopup`有分析
4. `cache miss`缓存未命中，此时进入方法列表`method_list_t`，它的结构类似于二维数组，里面存放着很多个`method_list_t`数组，包括自己的方法列表，各个分类的方法列表，这些数组是存放着`method_t`类型的指针数组
5. 遍历方法列表，取出每一个`method_t`中的`SEL`字段，与第二步中传入的`SEL`进行对比
6. 发现两个`SEL`一致，从当前`method_t`中取出`IMP`字段，作为函数地址调用，**它的参数必定会有一个调用者`receiver`**，这里的`receiver`从哪来呢？就是从`super`结构体中取出的`receiver`字段，在这里也就是self
7. 如果遍历完方法列表还没找到对应的`SEL`，就会从当前`class`顺着`superClass`指针进入父类，再重复一遍上面的流程，直到找到对应函数或者遇到了`NSObject`停止，此时进入方法解析阶段，不过这都是后话了

以上就是整个的调用流程，简单来说，**使用`[super class]`最后源码中的方法调用者还是`self`，只是从`self`的父类开始查找方法**

## 调用流程

事实上`objc_msgSend`函数的调用流程与上面讲的`super`的调用流程如出一辙

`objc_msgSend`的第一个参数`receiver`，第二个参数是`SEL`，后面更多的参数最后调用真实函数的时候会用到

1. 调用此方法的时候，第一个参数会默认赋值为`self`，而第二个参数会通过`@selector`方法获取，也就是获取一个`SEL`
2. 它会先顺着`self`的`isa`指针进入本`self`的类中，也就是进入了一个`objc_class`结构体，里面有`Cache_t,method_list_t`等字段
3. 先进行缓存查找，缓存命中直接取出`IMP`调用，未命中就进入方法列表
4. 同样取出方法列表的`method_t`结构体的`SEL`字段与最初传入的`SEL`字段进行比对，如果一致就取出`method`结构体中的`IMP`指针直接调用
5. 如果找遍了方法列表也没有，此时顺着`superClass`进入父类里面找，重复上述流程，直到遇到`NSObject`，找不到进入方法解析，消息转发阶段，这里的内容将会在笔者下一篇博客中详细讲到

# Category的实现原理

## Category相关数据结构

在`objc-runtime-new`中你可以找到`category_t`结构体的定义

![pasted-image.png · 281](https://img.cdn1.vip/i/6ab0c7301d939_1789970224.webp)

简化完，大概是这样

```objc
struct category_t {
    const char *name;
    classref_t cls;
    struct method_list_t *instanceMethods;
    struct method_list_t *classMethods;
    struct protocol_list_t *protocols;
    struct property_list_t *instanceProperties;
	  struct property_list_t *_classproperties;
};
```

从上面的定义中可以简单看出，`category`可以添加实例方法，添加类方法，还可以实现协议，实例属性，甚至类属性

现在就引出了一个问题，`category`是怎么被“合并”进`class`里面的？

**这就涉及到`Runtime`加载`category`时使用到的`attachCategories()`的主要内容：**

下面是对`attachCategories()`的简化版代码

```objc
static void attachCategories(Class cls, category_list *cats, bool flush_caches) {
    if (!cats) return;
    bool isMeta = cls->isMetaClass();

    method_list_t **mlists = (method_list_t **)malloc(cats->count * sizeof(*mlists));
    // Count backwards through cats to get newest categories first
    int mcount = 0;
    int i = cats->count;
    while (i--) {
        auto& entry = cats->list[i];

        method_list_t *mlist = entry.cat->methodsForMeta(isMeta);
        if (mlist) {
            mlists[mcount++] = mlist;
        }
    }

    auto rw = cls->data();

    prepareMethodLists(cls, mlists, mcount, NO, fromBundle);
    rw->methods.attachLists(mlists, mcount);
    free(mlists);
    if (flush_caches  &&  mcount > 0) flushCaches(cls);
}
```

首先介绍一下它的三个参数：

1. `Class class`这代表了该分类的目标`class`
2. `category_list *cats`这是一个结构体，里面有数组，数量等字段，数组中存了`class`目标类的所有分类的`entry`，而`entry`中有一个`cat`属性，它就是`category_t`类型，于是可以读出具体的分类信息
3. `flush_caches`它决定是否刷新方法缓存，由于`Runtime`要解析`category`的方法，所以说最后得到的目标类中，有些来自于`category`的方法并没有被加入缓存，这里可以选择是否加入

理解这些，我们就可以看这个循环是在干什么了

在`while`循环中，我们遍历了整个`cats`的`list`数组，得到每一个分类的`entry`，由于`enter`的`cat`属性就是`category_t`类型的，所以可以调用`category_t`结构体中的`methodsForMeta(isMeta)`方法返回一个`method_list_t`类型的方法列表，紧接着把这个方法列表放在二位数组`mlists`里面

- **循环结束后，就要把拷贝下来的方法列表接入到原目标类的方法列表上：**

`auto rw = cls->data();`

这行代码先通过原有类的`data()`方法获取了这个类的`class_rw_t`类型的指针赋值给`rw`

接着进行`prepareMethodLists(cls, mlists, mcount, NO, fromBundle);`

这里是处理一下是否需要排序、是否需要处理SEL等操作，以及是否要进行优化

>  这里处理、优化的内容笔者暂时没有学习，但其对整体逻辑影响不大

`rw->methods.attachLists(mlists, mcount);`

这行代码是将方法列表接到目标类上的关键，这里的`rw->methods`先获取了`class_rw_t`这个结构体的`method`字段，并且它继承了父类的`attchLists()`方法，这个方法如下

```objc
void attachLists(List* const * addedLists, uint32_t addedCount) {
    if (addedCount == 0) return;
    uint32_t oldCount = array()->count;
    uint32_t newCount = oldCount + addedCount;
    setArray((array_t *)realloc(array(), array_t::byteSize(newCount)));
    array()->count = newCount;
    memmove(array()->lists + addedCount, array()->lists, oldCount * sizeof(array()->lists[0]));
    memcpy(array()->lists, addedLists, addedCount * sizeof(array()->lists[0]));
}
```

作用就是把我们收集到的`category`方法列表接到目标类的方法列表上

注意这里并不是取代，而是拼在后面，可以近似的理解为扩充了原本的二维数组

> 需要注意的是，无论执行哪种逻辑，参数列表中的方法都会被添加到二维数组的前面。因此 category 中定义的同名方法不会替换类中原有的方法，但是对原方法的调用实际上会调用 category 中的方法

## load方法的执行

`+load()`方法在`class`和`category`中都有一个数组保存，在 runtime load_image 的时候，会一次调用 Class 和 Category 的 load 方法

load 方法的调用顺序是先调用父类的再调用子类的，先调用 Class 的，再调用 Category 的。

### 和`+initialize()`的同异

- `+load` 方法会被全部执行，而 `+initialize` 方法则只会像普通方法一样会被分类覆盖。
- `+initialize` 方法在类传递第一个消息之前执行，`+load` 在分类添加到 runtime 时执行。
- `+initialize` 方法先执行父类方法，再执行子类方法，`+load` 也是这样
