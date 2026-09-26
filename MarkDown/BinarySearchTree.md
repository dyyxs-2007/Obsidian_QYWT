###### <font face="楷体" color="green"><center> 二叉查找树的基本操作实现<center></font>
# 二叉树的基本介绍
## 二叉树的结构
- 二叉树是一种二分的数据结构，像一颗树一样自上到下发散开来， 这种数据结构在查找上提供了巨大便利
> 这是一个普通二叉树的例子

[![屏幕截图 2025-11-07 192453.png](https://youke1.picui.cn/s1/2025/11/07/690dd6f5c87d8.png)](https://youke1.picui.cn/s1/2025/11/07/690dd6f5c87d8.png)
- 如图， 最上方的节点我们称其为根（root），这一点与现实的树相反，而从上往下数的层数就是树的深
- 除根节点外，每一个节点都有它们的父节点和子节点，比如这里的5，就有父节点为根节点4，子节点6，3；而节点2和节点6，3，8的子节点为空节点，这些在二叉树最下层，子节点均为空姐点的节点，被称为叶子节点
## 二叉树的分类
- 二叉树根据不同功能被分为了几个类型：
### 普通二叉树：
> 如上面介绍的就是一个普通二叉树，能实现数据的存储，但在查找方面性能差

### 满二叉树：
> 满二叉树是二叉树中的一种形式，它指的是所有叶子节点都在同一层，并且非叶子节点都有两个子节点，就如同树上挂满了节点一样

### 完全二叉树：
> 完全二叉树与满二叉树类似，但完全二叉树要求最底层的叶子节点上从左到右排布节点，不能出现相隔节点的情况，如图就是一个完全二叉树的例子
![屏幕截图 2025-11-07 194235.png](https://youke1.picui.cn/s1/2025/11/07/690ddb1436cfb.png)

### 二叉查找树(BST)
- 二叉查找树(Binary Search Tree)作为我们今天的重点，我们对它详细介绍，所谓二叉查找树就是一颗用于快速插入，查找的二叉树
    1. 我们在前面已经或多或少学习过一个有序数组的二分查找，时间复杂度是很优秀的$log(n)$，但是想要用数组实现就需要每插入一个元素进行排序，这样就拉低了我们整体的时间复杂度
    2. 那么使用链表如何呢？链表对删除插入数据有着不错的性能，很容易维护数据的有序，但是链表这样的数据结构却不适合二分查找
    3. 有没有一种数据结构同时具备优秀的插入查找性能，又能维护数据的有序性呢？有的，有的
- 二叉查找树闪亮登场：
- 二叉查找树与普通二叉树相似，但在其基础上，增加了一条规则，那就是对一个非叶子节点而言，其左子节点储存的值小于其自身储存的值，自身储存的值又小于其右子节点的值

对上方普通二叉树给出的例子进行了重排，使得所有节点满足了二叉搜索树的性质
![屏幕截图 2025-11-07 200932.png](https://youke1.picui.cn/s1/2025/11/07/690de15eef547.png)
> 这一个规则既保证了插入的时候只需要二分地比较就能插入到正确的位置，并且在搜索的时候也使用的二分思想，这恰恰满足了我们幻想中的两种优良性质

- 如果对所说的插入搜索操作不了解，看完下面具体实现并动手练习也是了解二叉树优势的好方法
## 二叉树在实现上与链表等线性数据结构的区别
- 由于二叉树二分的特点以及根节点不允许有父节点的性质，我们就不能使用我们~~熟悉~~的虚拟头节点，而是需要老老实实判断根节点是否为空
- 在前面链表，队列等数据结构的实现中，我们多是用到了迭代遍历的思想，而在二叉树中，我们则更偏向于递归地实现
> 当然这并不是说不能使用非递归方式实现，但多比较复杂且需要栈的辅助
# 二叉查找树的具体实现
## 需要实现的具体功能

    对于实现一个数据结构，我们首先要分析它应该具有的功能
    对于二叉树我们需要它实现以下操作：
- 基础数据结构的操作，如初始化，销毁，判空，判长等操作，但对于二叉树而言，因为与链表的线性结构不同，二分的结构不便于我们构建哨兵节点来统一操作 （不了解这一点的可以学习一下链表的删除插入操作中的虚拟头节点dummyhead），所以常常要单独判断根节点是否为空，这一点会在下面的实现中展示
- 对于二叉树的具体操作：增（插入），删，改，查

## 在实现前的几个准备步骤：
- 头文件包含
- 定义好节点的结构
- 主函数创建出根节点
```c
#include<stdio.h>
#include<stdio.h>
typedef struct TreeNode {
	int val;
	struct TNOde* lchild;
	struct TNOde* rchild;
}TNode;

int main() {
	TNode* root = NULL;

	return 0;
}
```
- 用于分配新节点的函数，在插入等操作中可以直接调用：
```c
TNode* Alloc(void) {
	TNode* newnode = (TNode*)malloc(sizeof(TNode));
	if (NULL == newnode) {
		printf("fail to alloc\n");
		exit(1);
	}
	newnode->lchild = NULL;//直接初始化子节点
	newnode->rchild = NULL;
	return newnode;
}
```
## 基础的几个操作
基础数据结构如上面所介绍的有
1. 二叉树的初始化
> 二叉树的初始化是初始化主函数中已经创建出来的根节点，让根节点的值为空
- 代码实现如下：
```c
typedef struct TreeNode {
	int val;
	struct TNOde* lchild;
	struct TNOde* rchild;
}TNode;

void Init_BST(TNode** root) {
	*root = NULL;
}

int main() {
	TNode* root = NULL;
	Init_BST(&root);
	return 0;
}
```
  > 注意这其中传入的是根节点的地址，这是一个经典的问题，为什么不直接传入根节点呢？      原因是直接传入根节点进函数，此时进行了值传递，生成了一个root的副本，对这个副本进行我们想进行的赋值操作时，改变的也只有副本，对主函数的root没有任何影响，所以要传入root的地址来直接修改root的值

2. 二叉树的销毁：
> 二叉树的销毁涉及到遍历的知识点，也就是说，要先找到叶子节点并将其释放，之后要将其置空防止操作野指针，这里递归的思路需要多加熟练

- 代码示例：
```c
void Destroy_BST(TNode** adsroot) {
	if (NULL == (*adsroot)) {
		return;
	} else {
		if (NULL != (*adsroot)->lchild) {
			Destroy_BST(&(*adsroot)->lchild);
		}
		if (NULL != (*adsroot)->rchild) {
			Destroy_BST(&(*adsroot)->rchild);
		}
        free(*adsroot);
		*adsroot = NULL;
	}
}
```
3. 二叉树的判长
- 二叉树的判长其实就是数究竟有几个叶子节点，分析后不难得出当前节点有以下几种情况：
    1. 节点为空
    2. 非空且无子节点
    3. 非空至少有一个子节点
- 对于情况1， 直接返回0
- 情况2说明是叶子节点，返回1
- 情况3返回时先加上自身的1，再加上进入左孩子右孩子递归得出的值
- 代码示例如下：
```c
int Size_BST(TNode* adsroot) {
	if (NULL == adsroot) {
		return 0;
	}
	if (NULL == adsroot->lchild && NULL == adsroot->rchild) {
		return 1;
	}
	return 1 + Size_BST(adsroot->lchild) + Size_BST(adsroot->rchild);
}
```
4. 二叉树的判空：
- 只需判断传入的根节点是否为空即可，这个函数相对简单，有两种写法,使用时根据自己喜好
- 写法一：
```c
int Empty_BST(TNode* root) {
	return NULL == root;
}
```
- 写法二：
```c
int Empty_BST(TNode* root) {
	if (NULL == root) {
		return 1;
	} else {
		return 0;
	}
}
```
## 二叉查找树的插入节点
二叉树的插入，我总结了以下几条规则：
1. 当前节点为空时，插入当前节点
2. 当前节点不为空时，若待存值大于当前节点储存的值，更新当前节点至其右子节点
3. 当前节点不为空时，若待存值大于当前节点储存的值，更新当前节点至其右子节点
![屏幕截图 2025-11-08 161910.png](https://youke1.picui.cn/s1/2025/11/08/690efd482cd84.png)

二叉树的插入节点就需要这样比较，直到找到空节点插入
- 下面展示递归插入节点的代码：
- 由于头节点可能为空所以一样要在调用时传入地址
```c
void Insert_BST(TNode** adsroot, int tar) {
	if (NULL == *adsroot) {
		*adsroot = Alloc();
		(*adsroot)->val = tar;
		return;
	} else {
		if (tar > (*adsroot)->val) {//比当前值大，进入右节点
			Insert_BST(&(*adsroot)->rchild, tar);
		} else if (tar < (*adsroot)->val) {//比当前值小，进入左节点
			Insert_BST(&(*adsroot)->lchild, tar);
		} else {//如果等于当前节点直接返回
			printf("you can not insert tow identical val\n");
			return;
		}
	}
}
```
## 二叉查找树的查找节点
- 查找节点与插入节点类似，但查找节点应该返回当前节点的指针，所以在递归调用和判断上和插入不太一样，但同样都需要与当前节点的值进行比较

- 代码如下：
```c
TNode* Search_BST(TNode* adsroot, int tar) {
	if (NULL == adsroot) {
		printf("fail to search\n");
		return NULL;
	} else {
		if (tar > adsroot->val) {
			return Search_BST(adsroot->rchild, tar);
		} else if (tar < adsroot->val) {
			return Search_BST(adsroot->lchild, tar);
		} else {
			return adsroot;
		}
	}
}
```
## 二叉查找树的删除节点
- 二叉查找树的删除节点相对比较复杂，因为涉及到删除节点后对其排序性质的维护
- 我们选择将删除节点抽象出三种情况来处理
1. 对叶子节点的删除
2. 对只有单个子节点的删除
3. 对有两个子节点的删除
   1. 找到右子树中最小值节点
   2. 将该节点右子树中的最小值覆盖当前节点的val
   3. 删除右侧子树中的最小值节点
- 依旧使用递归的思想，代码如下：
```c
TNode* Min_BST(TNode* adsroot) {
	if (NULL == adsroot) {
		return NULL;
	} else if (NULL == adsroot->lchild) {
		return adsroot;
	} else {
		return Min_BST(adsroot->lchild);
	}
}//以上为辅助函数，找到右子树中的最小值
void Delete_BST(TNode** adsroot, int tar) {
	if (NULL == *adsroot) {
		return;
	}//以下为寻找节点过程
	if (tar > (*adsroot)->val) {
		Delete_BST(&(*adsroot)->rchild, tar);
	} else if (tar < (*adsroot)->val) {
		Delete_BST(&(*adsroot)->lchild, tar);
	} else {//进入待删除节点
		if (NULL == (*adsroot)->rchild && NULL == (*adsroot)->lchild) {//情况1
			free(*adsroot);
			*adsroot = NULL;
		} else if (NULL == (*adsroot)->lchild) {//情况2
			TNode* delptr = *adsroot;
			*adsroot = (*adsroot)->rchild;
			free(delptr);
		} else if (NULL == (*adsroot)->rchild) {//情况2
			TNode* delptr = *adsroot;
			*adsroot = (*adsroot)->lchild;
			free(delptr);
		} else {//情况3
			TNode* minnode = Min_BST((*adsroot)->rchild);
			(*adsroot)->val = minnode->val;
			Delete_BST(&(*adsroot)->rchild, minnode->val);
		}
	}
}
```
## 二叉查找树的修改节点
- 注意，二叉树节点上的值一经修改，有可能破坏其本身的排序性，这时候就需要我们先删除指定节点，再重新插入新节点，以此做到调整结构的修改
- 若直接修改，二叉搜索树就可能会退化为普通二叉树
### 普通的修改节点函数（数值直接替换）
- 代码如下：
```c
void Modify_BST(TNode* adsroot, int tar, int newval) {
	if (NULL == adsroot) {
		return;
	}
	else {
		if (tar > adsroot->val) {
			Modify_BST(adsroot->rchild, tar, newval);
		}
		else if (tar < adsroot->val) {
			Modify_BST(adsroot->lchild, tar, newval);
		}
		else {
			adsroot->val = newval;
		}
	}
}
```
### 调整结构的修改节点函数
- 其底层实现其实是删除节点和插入节点的结合，即删除原有节点，插入新值的节点
- 代码如下：
```c
void Modify_BST(TNode* adsroot, int tar, int newval) {
	Delete_BST(&adsroot, tar);
	Insert_BST(&adsroot, newval);
}
```
# 二叉查找树的遍历
- 区分二叉树的遍历方式我总结了以下两个要点：
1. 无论何种方式，大体上从左至右
2. 前中后三种方式，以第二个遍历的节点作为命名标准
- 例如前序遍历就是先检查本节点，再检查左孩子，最后右孩子第二个检查的是左所以叫前序；而中序是先左再本节点，再右节点，第二个检查的是中所以叫中序，后序遍历以此类推
> 其实理解命名的含义后，代码写起来就十分的简单
### 前序遍历
先检查当前节点，再遍历左右
- 代码如下：
```c
void Ergodic_Front_BST(TNode* adsroot) {
	if (NULL == adsroot) {
		return;
	}
	else {
		printf("%d", adsroot->val);
		Ergodic_Front_BST(adsroot->lchild);
		Ergodic_Front_BST(adsroot->rchild);
	}
}
```
### 中序遍历
- 代码如下：
```c
void Ergodic_Middle_BST(TNode* adsroot) {
	if (NULL == adsroot) {
		return;
	}
	else {
		Ergodic_Middle_BST(adsroot->lchild);
		printf("%d", adsroot->val);
		Ergodic_Middle_BST(adsroot->rchild);
	}
}
```
### 后序遍历
```c
void Ergodic_Back_BST(TNode* adsroot) {
	if (NULL == adsroot) {
		return;
	}
	else {
		Ergodic_Back_BST(adsroot->lchild);
		Ergodic_Back_BST(adsroot->rchild);
		printf("%d", adsroot->val);
	}
}
```
### 层序遍历（思路）
- 层序遍历指的是在一个二叉树中按照每一层的层内顺序遍历，直到当前层所有元素被遍历完再进入下一层
- 如果要进行层序遍历，需要借用队列这一数据结构，其主要思想就是将当前层的所有元素入队后记录，再弹出直到一层遍历完
- 由于我们是c语言的二叉树章节，这里不给出代码的具体实现，但是会给出每一步骤的思路，读者可以自行设计队列的接口进行实操
- 思路如下：
1. 从根节点开始，先将根节点入队，记录此时队中元素个数size 1
2. 此时根节点相当于已经遍历，再弹出队首元素前，将其两个子节点入队，注意这两个子节点排在root后，此时弹出队首元素
3. 从入队开始到出队共弹出一个元素，对应size的数目，说明第一层遍历完，并把第一层的所有子节点都入队了
4. 更新size为此时队中的值，按例子讲是2，则再将第一个节点的子节点入队并弹出节点，进行两次达到size，此时队中全是第三层从左到右的节点
5. 更新size为当前队中节点数4
6. 以此类推
# 完整代码：
```c
#include<stdio.h>
#include<stdio.h>
typedef struct TreeNode {
	int val;
	struct TreeNode* lchild;
	struct TreeNode* rchild;
}TNode;

void Init_BST(TNode** root) {
	*root = NULL;
}
//分配节点函数
TNode* Alloc(void) {
	TNode* newnode = (TNode*)malloc(sizeof(TNode));
	if (NULL == newnode) {
		printf("fail to alloc\n");
		exit(1);
	}
	newnode->lchild = NULL;
	newnode->rchild = NULL;
	return newnode;
}
//销毁树
void Destroy_BST(TNode** adsroot) {
	if (NULL == (*adsroot)) {
		return;
	} else {
		if (NULL != (*adsroot)->lchild) {
			Destroy_BST(&(*adsroot)->lchild);
		}
		if (NULL != (*adsroot)->rchild) {
			Destroy_BST(&(*adsroot)->rchild);
		}
		free(*adsroot);
		*adsroot = NULL;
	}
}
//判空树
int Empty_BST(TNode* root) {
	return NULL == root;
}
//取树大小
int Size_BST(TNode* adsroot) {
	if (NULL == adsroot) {
		return 0;
	}
	if (NULL == adsroot->lchild && NULL == adsroot->rchild) {
		return 1;
	}
	return 1 + Size_BST(adsroot->lchild) + Size_BST(adsroot->rchild);
}
//插入节点函数
void Insert_BST(TNode** adsroot, int tar) {
	if (NULL == *adsroot) {
		*adsroot = Alloc();
		(*adsroot)->val = tar;
		return;
	} else {
		if (tar > (*adsroot)->val) {
			Insert_BST(&(*adsroot)->rchild, tar);
		} else if (tar < (*adsroot)->val) {
			Insert_BST(&(*adsroot)->lchild, tar);
		} else {
			printf("you can not insert tow identical val\n");
			return;
		}
	}
}
//查找节点函数
TNode* Search_BST(TNode* adsroot, int tar) {
	if (NULL == adsroot) {
		printf("fail to search\n");
		return NULL;
	} else {
		if (tar > adsroot->val) {
			return Search_BST(adsroot->rchild, tar);
		} else if (tar < adsroot->val) {
			return Search_BST(adsroot->lchild, tar);
		} else {
			return adsroot;
		}
	}
}
//合并起来删除节点函数
TNode* Min_BST(TNode* adsroot) {
	if (NULL == adsroot) {
		return NULL;
	} else if (NULL == adsroot->lchild) {
		return adsroot;
	} else {
		return Min_BST(adsroot->lchild);
	}
}
void Delete_BST(TNode** adsroot, int tar) {
	if (NULL == *adsroot) {
		return;
	}
	if (tar > (*adsroot)->val) {
		Delete_BST(&(*adsroot)->rchild, tar);
	} else if (tar < (*adsroot)->val) {
		Delete_BST(&(*adsroot)->lchild, tar);
	} else {
		if (NULL == (*adsroot)->rchild && NULL == (*adsroot)->lchild) {
			free(*adsroot);
			*adsroot = NULL;
		} else if (NULL == (*adsroot)->lchild) {
			TNode* delptr = *adsroot;
			*adsroot = (*adsroot)->rchild;
			free(delptr);
		} else if (NULL == (*adsroot)->rchild) {
			TNode* delptr = *adsroot;
			*adsroot = (*adsroot)->lchild;
			free(delptr);
		} else {
			TNode* minnode = Min_BST((*adsroot)->rchild);
			(*adsroot)->val = minnode->val;
			Delete_BST(&(*adsroot)->rchild, minnode->val);
		}
	}
}
//修改节点函数
void Modify_BST(TNode* adsroot, int tar, int newval) {
	Delete_BST(&adsroot, tar);
	Insert_BST(&adsroot, newval);
}
//前序遍历
void Ergodic_Front_BST(TNode* adsroot) {
	if (NULL == adsroot) {
		return;
	}
	else {
		printf("%d ", adsroot->val);
		Ergodic_Front_BST(adsroot->lchild);
		Ergodic_Front_BST(adsroot->rchild);
	}
}
//中序遍历
void Ergodic_Middle_BST(TNode* adsroot) {
	if (NULL == adsroot) {
		return;
	}
	else {
		Ergodic_Middle_BST(adsroot->lchild);
		printf("%d ", adsroot->val);
		Ergodic_Middle_BST(adsroot->rchild);
	}
}
//后序遍历
void Ergodic_Back_BST(TNode* adsroot) {
	if (NULL == adsroot) {
		return;
	}
	else {
		Ergodic_Back_BST(adsroot->lchild);
		Ergodic_Back_BST(adsroot->rchild);
		printf("%d ", adsroot->val);
	}
}

int main() {
	TNode* root = NULL;
	Init_BST(&root);
	Insert_BST(&root, 5);
	Insert_BST(&root, 2);
	Insert_BST(&root, 3);
	Insert_BST(&root, 8);
	Insert_BST(&root, 10);
	Insert_BST(&root, 1);

	Ergodic_Back_BST(root);
	printf("\n");
	printf("%d", Size_BST(root));
	printf("\n");
	Modify_BST(root, 8, 4);
	TNode* temp = Search_BST(root, 5);
	printf("%d", temp->rchild->val);
	printf("\n");
	printf("%d", temp->lchild->rchild->rchild->val);
	
	Destroy_BST(&root);
	printf("\n");
	printf("%d", Size_BST(root));

	return 0;
}
```
![屏幕截图 2025-11-08 193107.png](https://youke1.picui.cn/s1/2025/11/08/690f29f3becfb.png)
# 二叉查找树的其他常用表现形式与高级应用（概念）
这里给出两种较为常用的二叉搜索树的改进，感兴趣的读者可以查找资料继续了解
## 平衡二叉树(AVL)
- 平衡二叉树指的是对一个二叉排序树而言，左右两颗子树的高度差不超过1，并且两个子树本身也是平衡二叉树
- 普通二叉搜索树由于当数据按递增或递减方式排列输入就会退化成一条链表，降低了其优秀的$logn$的查找效率，如果能在插入时不断调整树的结构，始终将较为中间的值放在跟节点上，于是就出现了平衡二叉树
- 平衡二叉树根据节点的平衡因子来调整树形，使用旋转等手段维护其查找性质
## 红黑树(自平衡BST)
- 红黑树顾名思义，其节点除了红色就是黑色，而颜色的出现遵循以下五条规则：
1. 根节点必须是黑色
2. 所有节点不是黑色就是红色
3. 所有叶子节点（这里指的是NULL节点）都是黑色
4. 红色不在父子节点中连续出现
5. 每一条从根节点到叶子节点的路径，所经过的黑色节点数相同
- 这五条保持颜色的性质使得只要维护这样的颜色性质就可以保证其查找效率，由于对高度的要求不严格，所以较AVL树有着更优秀的维护效率

# 小结
二叉树是一种十分重要的数据结构，它以二分的形式存储数据，在插入与查找上展现出了较链表和数组不同时具备的优良性质，实现二叉树的各种操作不仅是熟悉这种二分的数据结构的方式，更能锻炼递归的能力
- 至于二叉树的遍历中也可以用到栈的迭代法而不需要使用递归，这种方法较为麻烦，留给有兴趣的读者详细了解，这里不做过多赘述。
