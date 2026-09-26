# GCD简介

> 引自 [百度百科](https://link.juejin.cn/?target=http%3A%2F%2Fbaike.baidu.com%2Fitem%2FGCD) **Grand Central Dispatch（GCD）** 是 Apple 开发的一个多核编程的较新的解决方法。它主要用于优化应用程序以支持多核处理器以及其他对称多处理系统。它是一个在线程池模式的基础上执行的并发任务。在 Mac OS X 10.6 雪豹中首次推出，也可在 iOS 4 及以上版本使用。

GCD是纯c语言，是一门与block有关的技术，它通过派发**队列**将线程抽象出来，根据系统资源情况，适时地创建，复用，摧毁后台线程

# 基础概念

- **Serial vs. Concurrent 串行 vs. 并发**

他们是用来描述当前任务对于其他任务是如何被执行的，任务串行执行就是每次只有一个任务被执行，任务并发执行就是在同一时间可以有多个任务被执行。

- **Synchronous vs. Asynchronous 同步 vs. 异步**

在 GCD 中，它们描述当一个函数相对于另一个任务完成，此任务是该函数要求 GCD 执行的。一个同步函数只在完成了它预定的任务后才返回。一个异步函数，刚好相反，会立即返回，预定的任务会完成但不会等它完成。因此，一个异步函数不会阻塞当前线程去执行下一个函数。

# 队列分类

1. **Serial Queues 串行队列**
   添加到这个队列中的任务的执行时机受到 GCD 的控制；唯一能确保的事情是 GCD 一次只执行一个任务，并且按照我们添加到队列的顺序来执行。
   由于在串行队列中不会有两个任务并发运行，因此不会出现同时访问临界区的风险；相对于这些任务来说，这就从竞态条件下保护了临界区，实现了锁的功能。
2. **Concurrent Queues 并发队列**
   在并发队列中的任务能得到的保证是它们会按照被添加的顺序开始执行，但这就是唯一的保证了。任务可能以任意顺序完成，但是不知道何时开始运行下一个任务，或者任意时刻有多少 Block 在运行，这完全取决于GCD如何调度

# 常用队列类型

1. **The main queue**
   这是一个串行队列，提交在本队列上面的任务会进入主线程的RunLoop中被执行，并且由于是系统默认生成的，所以无法调用dispatch_resume()和dispatch_suspend()来控制执行继续或中断。此线程也被用于在一个并发队列完成任务后更新UI的队列
2. **Global queues**
   全局队列是并发队列，并由整个进程共享。进程中存在三个全局队列：高、中（默认）、低三个优先级队列。同样无法控制主线程dispatch队列的执行继续或中断。需要注意的是，三个队列不代表三个线程，可能会有更多的线程。并发队列可以根据实际情况来自动产生合理的线程数。
3. **用户队列**
   用户自己创建的队列。可以创建单线程的串行队列，也可以创建多线程的并行队列。

# 队列创建方式

1. **dispatch_queue_t queue = dispatch_queue_create(“com.dispatch.serial”, DISPATCH_QUEUE_SERIAL);**
   生成一个串行队列。第一个参数是队列的名称，在调试程序时会非常有用，所以尽量不要重名。第二个参数表示生成的队列是串行的，如果传入 null 默认是串行的。
2. **dispatch_queue_t queue = dispatch_queue_create(“com.dispatch.concurrent”, DISPATCH_QUEUE_CONCURRENT);**
   生成一个并发执行队列，参数解释见1
3. **dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);**
   获得全局队列(系统的队列，与并行队列类似，可以直接使用，不能指定名字)（并行）
4. **dispatch_queue_t queue = dispatch_get_main_queue()**
   获得主线程队列(串行)

# 提交任务

GCD提交任务需要调用`dispatch_async`或`dispatch_sync`函数，传入一个队列和一个block。队列会在轮到这个block执行时执行这个block的代码

## dispatch_async

dispatch_async 函数会立即返回, block会在后台异步执行。

```objc
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        [self goDoSomethingLongAndInvolved];
        NSLog(@"Done doing something long and involved");
});
```

需要注意的是，如果在完成某一个异步任务之后需要更新UI，那么你需要将更新UI的代码放在主线程里，这时候就需要使用刚刚获取主线程队列的代码，使用嵌套的dispatch，在外层中执行后台任务，在内层中将任务dispatch到main queue：

```objc
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        [self goDoSomethingLongAndInvolved];
        dispatch_async(dispatch_get_main_queue(), ^{
            [textField setStringValue:@"Done doing something long and involved"];
        });
});
```

## dispatch_sync

dispatch_sync 同步执行 block，函数不返回，一直等到 block 执行完毕。一般情况下是在**当前线程**中完成，因为派发同步任务，本身就要等到任务完成才能继续执行，那么就没有必要再开一个线程去专门执行这个同步任务，执行完后，再返回该线程了。但是如果在其他线程里往主队列里派发同步任务，那么这个同步任务还是会在主线程里执行，当前线程阻塞。

> **执行到 dispatch_sync 的时候，表示这个 block 必须要在当前线程立刻执行完**

我们在实际中最好尽可能避免使用 dispatch_sync，**嵌套使用一个串行队列时极易产生程序死锁**，比如sync主线程

![pasted-image.png · 74](https://img.cdn1.vip/i/6aa66ea522b7c_1789292197.webp)

注意这里死锁的很重要一个条件也因为**主队列是一个串行的队列**(主队列中只有一条主线程)。如果我们如下例，在并行队列中提交，则不会造成死锁：

```objc
dispatch_async(dispatch_get_global_queue(0, 0), ^{
  dispatch_sync(dispatch_get_global_queue(0, 0), ^{
      NSLog(@"任务一");
  });
  NSLog(@"任务二");
});
```

原因是并行队列中任务一虽被提交仍然是在queue的队尾，在任务二之后，但是因为是并行的，所以任务一并不会一直等任务二结束才去执行，而是直接执行完。此时任务二因为任务一的结束，sync阻塞也就消除，任务二就能执行

### 死锁

队列是串行或并发的，操作队列的函数是同步或者异步执行的（也就是在当前线程执行完返回和立即返回另开线程执行的区别）。串行队列其实就相当于加了一个资源锁，无论在多少个线程里，只有当队列中前一个元素执行完，后一个元素的代码块才能继续执行，并行队列则没有任何要求，有需要执行就立即执行。 

所以：

1. **串行队列要求队列前的必须先执行完，才能执行后面**。并行队列无要求
2. **dispatch_sync 要求当前代码块必须先执行完，才能执行后面**。dispatch_async 无要求

因此，dispatch_sync 和 串行队列在一起嵌套使用时，会容易产生死锁。

# 常用方法

## dispatch_apply

重复执行block，需要注意的是这个方法是同步返回，也就是说等到所有block执行完毕才返回。多个block的运行是否并发或串行执行也依赖queue的是否并发或串行。

```objc
dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
dispatch_apply([array count], queue, ^(size_t index){
    [self doSomethingIntensiveWith:[array objectAtIndex:index]];
});
[self doSomethingWith:array];
```

如果需要异步执行这些代码，只需要用dispatch_async方法，将所有block放在后台线程

## dispatch_after

延迟提交 block：

```
double delayInSeconds = 1.0; 
dispatch_time_t popTime = dispatch_time(DISPATCH_TIME_NOW, (int64_t)(delayInSeconds * NSEC_PER_SEC));
dispatch_after(popTime, dispatch_get_main_queue(), ^(void){
     // code to be executed on the main queue after delay
});
```

其中 `NSEC_PER_SEC` 表示一秒钟有多少纳秒。

dispatch_after 是**延迟提交，不是延迟运行，不是在特定的时间后立即运行**：

```
//创建串行队列
dispatch_queue_t queue = dispatch_queue_create("me.tutuge.test.gcd", DISPATCH_QUEUE_SERIAL);

//立即打印一条信息
NSLog(@"Begin add block...");

//提交一个block
dispatch_async(queue, ^{
    //Sleep 10秒
    [NSThread sleepForTimeInterval:10];
    NSLog(@"First block done...");
});

//5 秒以后提交block
dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(5 * NSEC_PER_SEC)), queue, ^{
    NSLog(@"After...");
});
```

五秒之后并不会打印after，因为此时block仅仅被提交进队列，前面还有任务

## dispatch_once

保证在APP运行期间，block中的代码只执行一次

```objc
static dispatch_once_t onceToken;
dispatch_once(&onceToken, ^{
    // code to be executed once
});
```

一定要注意的是 `dispatch_once_t` **必须是全局或 static 变量**。否则使用时会导致非常不好排查的 bug。

dispatch_once 递归调用会产生死锁。

## dispatch_group

一个dispatch group可以用来将多个block组成一组以监测这些Block全部完成或者等待全部完成时发出的消息。

- dispatch_group_create创建一个调度任务组

- dispatch_group_async把一个任务异步提交到任务组里

- dispatch_group_notify用来监听任务组事件的执行完毕，如果都完成触发回调

- dispatch_group_wait 设置等待时间，在等待时间结束后，如果还没有执行完任务组，则返回。返回0代表执行成功，非0则执行失败

  ```objc
  dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
  dispatch_group_t group = dispatch_group_create();
  for(id obj in array)
      dispatch_group_async(group, queue, ^{
          [self doSomethingIntensiveWith:obj];
      });
  dispatch_group_notify(group, queue, ^{
      [self doSomethingWith:array];
  });
  ```

除了 `dispatch_group_async` 直接发起一个异步队列，我们还可以配合使用下面两个方法：

- dispatch_group_enter(group);
- dispatch_group_leave(group);

这两个方法最好配合异步请求使用：

```objc
// 创建 dispatch 组
dispatch_group_t group = dispatch_group_create();

// 第一个请求：
dispatch_group_enter(group);
[self sendGetAddressByPinWithURLs:REQUEST(@"getAddressByPin.json") completionHandler:^(NSDictionary * _Nullable data, NSError * _Nullable error) {
    NSArray *addressList = [TXAddressModel mj_objectArrayWithKeyValuesArray:data[@"addressList"]];
    self.addressList = addressList;
    dispatch_group_leave(group);
}];

// 第二个请求
dispatch_group_enter(group);
[self sendCurrentOrderWithURLs:REQUEST(@"currentOrder.json") completionHandler:^(NSDictionary * _Nullable data, NSError * _Nullable error) {
    TXCurrentOrderModel *currentOrderModel = [TXCurrentOrderModel mj_objectWithKeyValues:data];
    self.currentOrderModel = currentOrderModel;
    dispatch_group_leave(group);
}];

// 当上面两个请求都结束后，回调此 Block
dispatch_group_notify(group, dispatch_get_main_queue(), ^{
    NSLog(@"OVER:%@", [NSThread currentThread]);
    [self setupOrderDataSource];
});
```

这个例子的使用场景是，在两个请求都返回完毕后，才回调一个 block。因为两个请求本生就是异步的，所以不能使用 `dispatch_group_async`。

## dispatch_barrier_async

GCD 提供了 `dispatch_barrier_async` 函数。该函数是为了处理读写操作时如果使用串行队列效率过低，而使用并行队列会导致数据竞争的问题的

它的处理思路是基于并行队列，如果把一个block用`dispatch_barrier_async`提交会在它前面提交的任务都结束后再执行当前block，并且暂时阻止在它之后的block执行，这样做到了在一段时间内只执行当前block，这样既满足了高效性又不会造成数据竞争，如图

![pasted-image.png · 3365](https://img.cdn1.vip/i/6aa67f4792e8b_1789296455.webp)

除了 `dispatch_barrier_async` 还有一个 `dispatch_barrier_sync`，两者有什么区别呢？两者都能达到等待在它前面插入队列的任务先执行完，等待他们自己的任务执行完再执行后面的任务的目的。但是不同的是，`dispatch_barrier_async` 将自己的任务插入到队列之后，不会等待自己的任务结束，它会继续把后面的任务插入到队列，然后等待自己的任务结束后才执行后面任务；`dispatch_barrier_sync` 将自己的任务插入到队列的时候，需要等待自己的任务结束之后才会继续插入被写在它后面的任务，然后执行它们。也就是说前者不会阻塞当前线程，而后者会阻塞当前线程。

## dispatch_semaphore

dispatch_semaphore 主要用来实现锁的操作。可用于请求同步等处理。

- `dispatch_semaphore_t semaphore = dispatch_semaphore_create(value);`
- `dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);`
- `dispatch_semaphore_signal(semaphore);`

value 可以理解为资源数量，以 value = 0 为例，调用 `dispatch_semaphore_wait` 操作成功后，当资源数量 value 等于 0 时，就会阻塞当前线程（反之，value 就会减 1），直到有 `dispatch_semaphore_signal` 通知信号发出，当 value 大于 0 时，当前线程就会被唤醒继续执行其他操作。

##### 例子:模拟同步请求，阻塞当前线程：

```objc
// 1.创建信号量
dispatch_semaphore_t semaphore = dispatch_semaphore_create(0);
NSLog(@"0");
// 开始异步请求操作（部分代码略）
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
    NSLog(@"1");
    // This function returns non-zero if a thread is woken. Otherwise, zero is returned.
    // 2.在网络请求结束后发送通知信号
    dispatch_semaphore_signal(semaphore);
});
// Returns zero on success, or non-zero if the timeout occurred.
// 3.发送等待信号
dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
NSLog(@"2");

// print 0、1、2
```

由于 `dispatch_semaphore_wait`，线程阻塞。只有等待异步请求返回后执行 `dispatch_semaphore_signal`后，才会继续执行下面的代码。所以先打印 1，后打印 2.