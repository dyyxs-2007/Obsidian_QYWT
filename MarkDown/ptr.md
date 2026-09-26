###### <center><font color=green face="楷体">数据结构中的二级指针问题</font><center>
# 对指针的简单回顾
- 我们知道，内存在设备中以字节为单位被编号，也就是说每一个被存储的数据对应内存中都有自己的地址编号，而c语言开放了指针，方便我们直接对内存地址上的数据进行操作

| 地址| 0x00001 | 0x00002 | 0x00003 |
|-|---------|---------|---------|
| 数据|1|12|14|
- 下方是数据，上方是对应的地址
> 本例中 &1 = 0x00001, &12 = 0x00002, &14 = 0x00003

- 一个指向数据的指针，指向的是数据的地址，可以通过该地址修改数据的值
- 而一个指针的指针，可以通过该指针修改它指向的指针，也可以修改它指向的指针指向的值

# 一个错误示范
### 一个简单的代码来展示常见的错误用法
考虑下面这段程序
```c
#include<stdio.h>
#include<stdlib.h>

typedef struct ListNode {
	int date;
	struct ListNode* next;
}Node;
//初始化函数
void List_Init(Node*phead) {
	if (NULL == phead) {
		Node* ptr = (Node*)malloc(sizeof(Node));
		if (NULL == ptr) {              //判断是否成功分配
			printf("fail to alloc\n");
			exit(1);
		}
		phead = ptr;         //核心初始化内容
		phead->next = NULL;
		phead->date = 0;  //此行有无均可
	}
}
int main() {
	Node* head = NULL;
	List_Init(head);
	//判断head是否被初始化，如果初始化了，head会成为指向一个结构体的指针而不是NULL
    if (NULL == head) {
		printf("fail to return\n");
	} else {
		printf("init successfully\n");
	}
	return 0;
}
```
### 程序的运行结果：

![屏幕截图 2025-10-27 143700.png](https://youke1.picui.cn/s1/2025/10/27/68ff13db1360e.png)
- 如你所见，我们确实没有改变head的值，那么问题出在了哪里？

# 为什么要传入二级指针
- 简单来说，问题出在了传参上，在上面的程序中，我们想要改变head的值，想把它**从**一个指向NULL的指针**更改为**指向新的结构体的指针，<font color=red>但是我们传入init函数的却只是head的值也就是NULL</font>,所以在init函数内部进行的一切操作都不会波及主函数的head！
```c
void List_Init(Node*head)
head=NULL;
List_Init(head);
```
- 如上方的代码切片，传入的只是一个值所以无法改变它本身，所以我们需要传入一个二级指针，这个二级指针储存的是head的地址，这样一来，我们就可以传入head的地址进入init函数，***通过解引用改变head的值了***！
# 正确示范
- 考虑下面的程序
```c
#include<stdio.h>
#include<stdlib.h>

typedef struct ListNode {
	int date;
	struct ListNode* next;
}Node;
//初始化函数
void List_Init(Node**phead) {
	if (NULL == *phead) {
		Node* ptr = (Node*)malloc(sizeof(Node));
		if (NULL == ptr) {              //判断是否成功分配
			printf("fail to alloc\n");
			exit(1);
		}
		(*phead) = ptr;         //核心初始化内容
		(*phead)->next = NULL;
		(*phead)->date = 0;  //此行有无均可
	}
}
int main() {
	Node* head = NULL;
	List_Init(&head);
	//判断head是否被初始化，如果初始化了，head会成为指向一个结构体的指针而不是NULL
    if (NULL == head) {
		printf("fail to return\n");
	} else {
		printf("init successfully\n");
	}
	return 0;
}
```
- 注意看，在上方的程序中主要有以下几点发生了改变
  1. 主函数中传参形式从head变成了&head
  2. LIst_Init函数的声明中传参形式从Node*变成了Node**
  3. List_Init函数中的所有phead被*phead替代
- 如我们上面所言，为了改变head的值使其能被更改，我们需要传入head的地址，但由于<font color=red>head本身就是一个指向结构体的指针</font>，这样一来，传入的就是一个二级指针，即Node**
- 而此时函数体内的phead是外面head的地址，为了更改head的值，我们需要对phead进行解引用来修改head，注意->运算符优先级高于*，需要我们主动为*phead带括号
- 修改后运行结果如下
  ![屏幕截图 2025-10-27 145446.png](https://youke1.picui.cn/s1/2025/10/27/68ff196261081.png)
# 避免二级指针的其他手段
-   在函数中使用return，主函数内接收更新：
    如上文所分析，head没改变的原因是init函数内的phead更新没引起head的改变，那我们<font color=red>主动返回这个已经被初始化的phead并且赋值给主函数的head</font>不就可以更新head了吗

    - 核心代码如下，读者感兴趣可以自行补充完整
```c
List_Init (Node*phead) {
    ......
    return phead;
}
int main() {
    Node head = NULL;
    head = List_Init(head);
}
```