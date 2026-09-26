###### <font color=green face="楷体"><center>单链表实现中的基本操作<center></font>
# 介绍链表
- 链表是一种线性的数据结构，与数组在内存中的连续储存不同，数据在内存中是分散的，被一个个指针像穿手链一样串起来

|0x0001|...|0x0A23|...|0x0F05|...|0xAF76|
|-|-|-|-|-|-|-|
|节点1|->|节点3|->|节点5|->|节点七|
### 链表的基本单元：
- 从上面的简介可以知道，链表需要有指针将其一个个串起来，同时也需要一些地方来存储本身需要存储的数据，于是我们***可以将一个单链表的单个节点分为两个区域，其中一个用来存储数据，另外一个用来存储下一个节点的地址***
![屏幕截图 2025-10-28 084044.png](https://youke1.picui.cn/s1/2025/10/28/69001105519d5.png)
- 图中x1，x2，x3分别代表了三个不同的数据，由于三个节点地址在内存中并不相邻，所以需要有next指针指向下一个节点，注意最后一个节点的next指针为空

# 明确目的
- 实现一个能使用的链表，能完成链表应具有的各项操作
- 用函数封装的思想将其各种操作封装起来
# 需要实现的功能
1. 对一个空的头节点初始化
2. 销毁一个链表
3. 判断链表是否为空
4. 求链表长
5. 链表的添项
    1. 从头节点插入
    2. 从尾节点插入
    3. 从指定节点插入
 6. 链表的删项
 7. 链表的改项
 8. 链表的查找项

- 我们会在下面对这些操作逐一进行讲解分析
# 具体实现
#### 1. 前提准备：
 1. 头文件包含 dtsio.h, stdlib.h
 2. 声明结构体：
```c
typedef struct ListNode{
    int date;
    struct ListNode*next;
}Node;
```
结构体单元中包含一个数据域和指针域
 3. 主函数声明头节点（哨兵节点）
```c
int main() {
    Node*head = NULL;
    return 0;
}
```
#### 2. 对一个空的头节点初始化
> 注意从此后我们的操作均在外置函数中封装，最后会由main函数驱动运行

```c
Node* List_Init(Node*head) {
    if (NULL == head) {
        Node*tempptr = (Node*)malloc(sizeof(Node));
        if (NULL == tempptr) {
            printf("fail to alloc\n");
            exit(1);
        }
        head = tempptr;
        head->next = NULL;
        head->date = 0;
    }
    return head;
}
```
以上就是初始化函数，主函数使用```head = ListInit(head)"```来更新head的值（当然也可用二级指针，详情请看我另一篇博客二级指针问题）
[二级指针问题](https://blog.csdn.net/2505_93363502/article/details/153971164?fromshare=blogdetail&sharetype=blogdetail&sharerId=153971164&sharerefer=PC&sharesource=2505_93363502&sharefrom=from_link)

- 对空的头节点分配空间并将它的next初始化为NULL，以便于我们以后能在头节点后面添加新项
- 注意这个头节点并不储存数据，只是为了将插入删除等操作的统一手段，这一带你在后面将会为你展示

#### 3. 销毁一个链表
- 销毁链表不单单要重新设置头指针，也要释放分配的那些空间
- 设置一个新指针从头指针的下一个开始移动，直到NULL之前释放所有节点内存
- 类似初始化地重置head的值

原理很简单，上代码实践：
```c
void List_Destroy(Node* head) {
	Node* ptr = head;
	while (NULL != ptr) {
		Node* delptr = ptr;
		ptr = ptr->next;
		free(delptr);
	}
}
```
- 声明指针从head开始遍历，每在一个节点就定义一个临时指针储存当前结构体地址
- 由于删除后就找不到下一个节点，所以必须要先让ptr进入下一个节点再释放当前节点内存

#### 5. 判断链表是否为空
- 判空链表有两种方式如下，自行选择使用
```c
int List_Empty(Node* head) {
	return NULL == head->next;
}
```
和
```c
int List_Empty(Node* head) {
	if (NULL == head->next) {
        return 1;
    } else {
        return 0;
    }
}
```
原理就是判断头节点head后面还有没有项
#### 6. 求链表长
> 我们所说的求链表长指的是求出链表中的有效项数
```c
int List_Length(Node* head) {
	int count = 0;
	Node* ptr = head->next;
	while (NULL != ptr) {
		ptr = ptr->next;
		count++;
	}
	return count;
}
```
- 仍然先声明一个指针从头节点的后一个节点开始移动，直到NULL结束
- 在这期间每移动一次计数器自增来记录经过的节点数，最后返回计数器

#### 7. 链表的添项
##### 1. 从头节点插入：
- 所谓从头结点插入，就是从我们一开始分配的head后面插入一项
- 先分配新节点需要的空间，再填入该储存的数值
- 将原本head的下一个节点更改为新节点的下一个节点，让head的next指向新的节点
> 这里一定要先让新节点的next为head的next，否则先更新head的next的话，会丢失head原本下一个的地址

- 代码如下：
```c
void List_Front_Create(Node* head, int tar) {
	Node* tempptr = (Node*)malloc(sizeof(Node));
	if (NULL == tempptr) {
		printf("fail to alloc\n");
		exit(1);
	}
	tempptr->date = tar;
	tempptr->next = head->next;
	head->next = tempptr;
}
```
- 由于原本的head的next不论是空链表时的NULL还是指向结点，都能有一个值赋给新节点的next，这也恰恰说明了创造头结点的好处---形式上统一，在后面的查找中你会有更深的理解
##### 2. 从尾节点插入：
- 和头插法相似，先分配节点需要的空间
- 从传入的head头节点开始，遍历来到最后一个节点，再在最后一个结点的后面连接上新节点
- 不要忘记将新节点的next更改为NULL，因为它此时已经为最后一项
- 代码如下
```c
void List_Rear_Create(Node* head,int target) {
	Node* tempptr = (Node*)malloc(sizeof(Node));
	if (NULL == tempptr) {
		printf("fail to alloc\n");
		exit(1);
	}
	Node* ptr = head;
	while (NULL != ptr->next) {
		ptr = ptr->next;
	}
	ptr->next = tempptr;
	tempptr->next = NULL;
	tempptr->date = target;
}
```
> 注意看，这里的遍历环节，终止条件是NULL == ptr->next，假若链表为空，那么从头节点出发前直接跳出循环，如果至少有一项，那么就会遍历到最后一项的next前停下，这样不论如何，我们都可以找到我们要添加位置的前一项

- 倘若不制造头节点就需要单独的判断链表是否为空的情况
##### 3. 从指定节点插入：
- 我们需要指定一个节点插入，就需要在遍历的时候加装一个计数器，来判定我们是否到达指定添加节点
- 同时为了确定指定节点数值合法（不超出最大节点个数），当遍历到最后一项的NULL时说明节点数非法，直接返回
- 其他步骤与头尾插入类似
- 代码如下：
```c
void List_Add(Node* head, int count,int tar) {
	Node* tempptr = (Node*)malloc(sizeof(Node));
	if (NULL == tempptr) {
		printf("fail to alloc\n");
		exit(1);
	}
	Node* ptr = head;
	int cons = 0;
	while (NULL != head && cons < count - 1) {
		cons++;
		ptr = ptr->next;
	}
	if (NULL == head) {
		return;
	}
	tempptr->next = ptr->next;
	ptr->next = tempptr;
}
```
- 若想插入节点，必须遍历到待插入节点的前一个节点，才能维护链表不断，当需要到达第n项时，由于起始点是head，所以需要移动n次，于是到达前一项需要移动n-1次
#### 8. 链表的删项
- 对于删掉链表中指定项的问题，我们的策略与前面从指定节点插入一样
- 注意删除时需要一个临时指针保存待删除节点的地址以便后续释放该节点内存
- 删除时先将删除节点的next给删除节点前一个节点的next
- 代码如下：
```c
void List_Delete(Node* head,int count) {
	Node* ptr = head;
	int cons = 0;
	while (NULL != ptr && cons < count - 1) {
		cons++;
		ptr = ptr->next;
	}
	if (NULL == ptr) {
		return;
	}
	Node* delptr = ptr->next;
    //将删除项下一个节点地址给删除项前一个结点的next
	ptr->next = ptr->next->next;
	free(delptr);
}
```
#### 9. 链表的改项
- 链表的改项思想与指定位置添项相似，但是改项时可以直接遍历到待改节点
- 代码如下：
```c
void List_Modify(Node* head, int count, int tar) {
	Node* ptr = head;
	int cons = 0;
	while (NULL != ptr && cons < count) {
		cons++;
		ptr = ptr->next;
	}
	if (NULL == ptr) {
		return;
	}
    //更改语句
	ptr->date = tar;
}
```
> 注意此时的循环跳出条件为count，目的是直接遍历到待改节点，遍历到NULL的特殊情况处理与添项时类似
#### 10. 链表的查找项
##### 1. 已知数值查找指针：
- 核心思想就是从头节点出发在遍历到NULL前逐个比对date和tar的值
- 如果查找到了项，就返回指向该节点的指针，如果不存在该项，就返回NULL
- 代码如下：
```c
Node* List_Node_Seek(Node* head, int tar) {
	Node* ptr = head->next;
	while (NULL != ptr) {
		if (tar == ptr->date) {
			return ptr;
		}
		ptr = ptr->next;
	}
	return NULL;
}
```
- 在while循环中，ptr遍历每个节点，发现date等于tar就返回当前节点指针，如果一直到链表尾部都没找到说明不存在该项，返回空指针

##### 2. 已知项数查找数值
- 还是以计数器的形式从head遍历抵达目的项，并处理项数非法的情况
- 返回指定的节点的date，如果没找到返回-1
- 代码如下：
```c
int List_Date_Seek(Node* head, int count) {
	Node* ptr = head->next;
	int cons = 0;
	while (NULL != ptr && cons < count) {
		ptr = ptr->next;
	}
	if (NULL == ptr) {
		return - 1;
	}
	return ptr->date;
}
```
# 小结：
### 链表的基本操作围绕指针展开，分析好每个节点指针的指向，处理好边界条件如无节点，单节点，遍历到尾节点的情况就行
### 在构建一种数据结构前明确其需要实现的功能，比如***通用的判空，初始化，销毁，求长***；以及链表的增删改查，栈和队列的push和pop，二叉树的遍历和搜索，<font  color=red>对应模块化地实现部分功能，就能找到实现数据结构的通解
### 数据结构的实现刚开始确实比较困难，但是找到方法以后进步就会很快，一起进步，共勉