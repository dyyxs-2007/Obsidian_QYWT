###### <center><font color=green face="楷体">OC考核问题总结</font></center>
# 空指针，野指针，悬空指针
1. 空指针
- 空指针指向0(nil/NULL),即不指向任何有效内存地址
- 如果赋值为nil就能创造
- 在OC中，向ni发送消息是安全的，程序不会崩溃，只会返回空值或0
1. 野指针
- 野指针指的是在创建出来指针变量后未初始化，这样的指针指向的是随机的内存地址，访问它可能会造成崩溃或者异常的数据损坏
2. 悬空指针
- 悬空指针指的是原先指向一个有效的对象，但是后来这个对象被释放，指针却仍指着被释放掉的地方，再次访问时，会触发`EXC_BAD_ACCESS`错误

# 容器类的完全深拷贝方法
- 由于OC的容器类在mutableCopy的时候默认对容器内的对象进行引用而不是拷贝，所以有时拷贝容器需要递归地把容器内的对象都进行深拷贝
- 归档与解档
> 要求容器内所有自定义类都遵守<NSCoding>和<NSSecureCoding>协议
```objc
NSArray *deepCopy = [NSKeyedUnarchiver unarchiveObjectWithData:NSKeyedArchiver archivedDataWithRootObject:oldArray];
```
> 注意，这是老式的写法(iOS12以前)

- 归档与解档简单讲就是给对象完全扫描后存入二进制文件，再读出，就实现了对一个对象的完全克隆
- 还有手动写递归的方式笔者正在学习中

# 自定义类的copy方法
- 自定义类想要实现copy方法需要先遵守`<NSCopying>`协议
```objc
//.h文件
@interface User : NSObject <NSCopying>
@property (nonatomic, copy) NSString *name;
@property (nonatomic, assign) NSInteger age;
@end

//.m文件
@implementation User
- (id)copyWithZone:(NSZone *)zone {
    User *copy = [[[self class] allocWithZone:NULL] init];
    copy.name = [self.name copy];
    copy.age = self.age;
    return copy;
}
@end
```
# 自定义类去重
- 自定义类去重需要重写当前类的hash方法和isEqual方法，这是因为set的去重逻辑是先判断hash值是否相等，如果相等，再使用isEqual进行确认
`以先前的User类为例`
```objc
@implementation User
- (BOOL)isEqual:(id)object {
    if (self == object) {
        return YES;
    }
    if (![object isKindOfClass:[User class]]) {
        return NO;
    }
    User *other = (User *)object;
    return [self.name isEqual:other.name];
}

- (NSUInteger)hash {
    return [self.name hash] ^ self.age;
}
@end
```

# C类型与OC转换
1. 基础数据类型
- (int, float)转换为OC类型
- OC使用NSNumber包装
```objc
int cInt = 10;
float cFloat = 3.77f;
//c->OC
//使用字面量
NSNumber *ocInt = @(cInt);
NSNumber *ocFloat = @(cFloat);
//传统写法是[NSNumber numberWithInt:...]方法

//OC->c
int backInt = [ocInt intValue];
float backFloat = [ocFloat floatValue];
```
2. NSString与C字符串
```objc
NSString *ocString = @"Hello ggg";
//OC->C
const char *cString = [ocString UTF8String];

//C->OC
NSString *backString = [NSString stringWithUTF8String:cstring];
```
3. 结构体与OC类
- 由于C中结构体是值类型，所以OC提供NSValue来包装结构体
```objc
//对于一些常用结构体(CGPoint, CGRect)
CGPoint point = CGPointMake(10, 20);
//转换
NSValue *pointValue = [NSValue valueWithCGPoint:point];

//对于自定义结构体
typedef struct {
    int id;
    double score;
} MyStudent
MyStudent stu = {101, 98.5};

//@encode(MyStudent)用来生成结构体的类型描述字符串
NSvalue *stuValue = [NSValue valueWithBytes:&stu objCType:@encode(MyStudent)];
```
- 事实上，这样包装完取值还是要先转换回结构体，包装只不过是为了存储和传递罢了
