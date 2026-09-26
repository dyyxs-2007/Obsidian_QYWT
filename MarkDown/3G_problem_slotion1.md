###### <center><font color=green face="仿宋">3G移动应用开发实验室一面试题题解</font><center>

---
# 题目一

- ***题目代码如下，现分析程序输出***
```c {linenos=true}
#include<stdio.h>
#include<string.h>
int main() {
	int a = sizeof(int) - sizeof(char);
	char str[] = "Good";
	char G = str[strlen(str) - 4];
	int b = strlen("Hello world and hello 3G!");
	int c = sizeof("Welcome to 3G lab!!");
	printf("Hello %d%c %d%d\n", a, G, c, b);
	return 0;
}
```
- 分析如下
> 代码前几行声明了a,数组str，字符G，b和c
> >1. a是int类型内存减去char类型内存，即4-1=3；
> >2. strlen(str) -4指的是字符数组str所占内存大小减去对应四个字节，即4-4=0，则G应该为str数组中的第一个值，即G；
>>3. 再看bc，这两个的区别在于其测量字符串长度的函数不同，与strlen直接测量字符串所占内存不同，sizeof则是完全测量该字符串数组所占空间，所以会相较于	strlen多一个终止符\0的内存
>
>经过上方分析，最终a=3，G='G',c=20,b=25;
输出为Hello 3G 2025
---
# 题目二
- ***题目代码如下，求问程序输出***
```c
#include<stdio.h>
const int MAX_VALUE = 100;
void test_function() {
	static int call_count = 0;
	int local_var = 10;
	call_count++;
	local_var++;
	printf("static=%d,local= %d\n", call_count, local_var);

	if (1 == call_count) {
		int local_var = 50;
		printf("scope1:local_var= %d\n", local_var);
		printf("Hello 3g\n");
	} else {
		printf("scope2:local_var= %d\n", local_var);
	}
}

int main() {
	const int MAX_VALLUE = 200;
	printf("MAX_VALLUE: %d\n", MAX_VALLUE);
	test_function(); 
	test_function(); 
	return 0; 
}
```
- 分析如下
> 进入主函数首先声明了一个不可修改的局部变量MAX_VALLUE,注意到最上方还声明了一个同名的全局变量，但在主函数输出时，所采用的是当前函数创建的局部变量而不是同名的全局变量，所以在第二行输出时应该是主函数中声明的200；
> 接下来进入test_function函数，程序跳转入函数内，先声明一个静态储存区的变量1 call_count，然后声明了一个自动变量2 local_var,下一步使这两个变量自增，此时变量1为1，变量2为11，判断if语句，成立，创建新的同名变量local_var为50，但实际不改变变量2的值，输出50。退回主函数，该函数再次被调用，刚刚创建的静态储存区变量仍储存着上次操作的值1，而local_var却被更新为10，进行if条件判断时，call_count值为2，进入else语句，将10输出。
---
# 题目三
- ***题目代码如下，求问程序输出***
```c
#include<stdio.h>
int main() {
	float f1 = 2011.0f / 3.0f;
	double d1 = 2011.0 / 3.0;
	double d2 = 2011.0f / 3.0;
	printf("f1=%.10f\n", f1);
	printf("d1=%.20lf\n", d1);
	if (d1 == d2) {
		printf("d1 == d2\n");
	} else {
		printf("d1 != d2\n");
	}
	return 0;
}
```
- 分析如下
f1,d1,d2的区别：
> f1是double型变量，赋值语句也将转换为float的两个数值的运算结果赋值给f1，而d1被两个double类型数据运算后的double数据赋值，精度较f1更高，至于d2，为d2赋值的数据中既有float又有double，在进行运算时会先将float升级为double型，再进行运算时精度与d1相同。

综上最终会进入if语句输出d1==d2

---
# 题目四
- ***题目代码如下，求问程序输出***
```c
#include<stdio.h>
int main() {
	int i = 0, j = 0;
	if (i++ && ++j) {
		j = j >> 2;
		printf("if branch: i=%d,j=%d\n", i, j);
	} else {
		i = i << 2;
		printf("else branch: i=%d,j=%d\n", i, j);
	}
	return 0;
}
```
- 分析如下
> i与j一开始被初始化为0，在下面的判断语句里从左到右进行运算，i自增为1，此时判断使用的仍是未自增时的0，导致&&运算符被短路[^1]，程序进入else语句，i进行位运算--即储存i的数据整体在字节中向左移动一位，可视作乘以2的一次幂，最终输出i=4，j=0
---
# 题目五
- ***题目代码如下，求问程序输出***
```c
#include<stdio.h>
	int process(int num) {
		while (1) {
			switch (num) {
			case 0:
				num = (num ^ 123 ^ 87) | 16;
			case 1:
				num = num ^ 15;
				break;
			case 2:
				--num;
			case 3:
				num = num + 1;
				break;
			case 4:
				num = num + 1;
			case 5:
				return (1 << num);
			case 6:
				num = num - 3;
				break;
			case 10:
				num = num - 6;
				break;
			case 15:
				num = num / 3;
				break;
			default:
				num = (num % 2) ? (num - 1) : (num / 2);
			}
		}
	}
int main() {	
	int start = 0;
	int answer = process(start);
	printf("answer=%d\n", answer);
	return 0;
}
```
- 分析如下：
首先主函数传入start=0作为被调函数的初始值进入while循环中的switch语句，我们来看看它会怎么运行
1. num=0进入case 0，num依次与123，87的二进制数进行异或运算[^2]然后与16的二进制数进行或运算[^3]，结果为111100（2）
2. 注意到case 0后面没有break；也就是说，程序会继续向下执行case 1，即将上一步得到的num再次与15的二进制数进行异或运算为51（10）
3. 此时遇到break；跳出switch语句，进行下一轮循环，此时输入的值为上一步得到的51，进入switch语句的default语句，并反复进行三目运算符操作直到num=6
4. num=6再循环进入switch语句的case 6，num被更新为3，跳出
5. 再运行，num更新为4，进入case 4，num被更新为5
6. 最后运行case 5，将1按位向高位移动五个比特位，即给1乘以2的五次方。
---
# 题目六
- ***题目代码如下，求问程序输出***
```c
#include<stdio.h>
int strange(int n, int m) {
	if (0 == n) {
		return m;
	} else if (0 == m) {
		return n;
	}
	return strange(n - 1, m) + strange(n, m - 1);
}
int main() {
	int result = strange(3, 2);
	printf("%d\n", result);
	return 0;
}
```
- 分析如下：
> 本题考查递归，初始传入n=3，m=2，一次运行后再次传入n=2，m=2和n=3，m=1，反复如此运行，直到有n或m等于0时开始逐级返回，最重加和为15，本题思路方面没什么较大问题，运算无误即可。
---
# 题目七
- ***题目代码如下，求问程序输出***
```c
#include<stdio.h>
int main() {
	int arr[2][3] = {{1, 2, 3}, {4, 5, 6}}; 
	int* ptr = arr[0]; 
	int (*arrPtr)[3] = arr; 
	int* ptrArr[2]; 
	ptrArr[0] = arr[0]; 
	ptrArr[1] = arr[1]; 
	printf("*(arr[1]+2)=%d\n", *(arr[1] + 2));
	printf("arrPtr[1][1]=%d\n", arrPtr[1][1]);
	printf("*(*arrPtr+3)=%d\n", *(*arrPtr + 3));
	printf("*(ptr+2)=%d\n", *(ptr + 2));
	printf("arr[0][1]=%d\n", arr[0][1]);
	printf("ptrArr[0][0]=%d\n", ptrArr[0][0]);
	return 0;
}
```
- 分析如下：
1. 指针ptr被赋予arr[0]上储存的值，即arr[0][0]的地址
2. arrptr为数组指针，指向一个有三个元素的数组，arr储存的是arr[0]的地址，即arr[0][0]的地址的地址，将这个地址的的地址赋给arrptr，此时arrptr指向{1,2,3}
3. ptrarr为指针数组，数组内第一个指针指向arr[0][0]，第二个指向arr[1][0]
> 1. 输出时第一行为arr[1][0]的地址向后偏移两位再解引用，就是arr[1][2],即6
> 2. 第二行arrptr指向{1,2,3}的地址，arrptr[1][1]即先对arrptr解引用一次并向后偏移一个对应长度，但因为此时二级数组储存的是各个小元素集合的地址，所以i偏移量为3，再进行解引用一次，并向后偏移一个单位，即arr[1][1]的5
> 3. 对arrptr先解引用得到arr[1][1]的地址，再加三偏移到arr[1][1]即4
> 4. ptr储存的是a[0][0]的地址，先加2偏移到arr[0][2],即3
> 5. arr[0][1]即2
> 6. ptrarr[0]储存的是一个指向arr[0][0]的指针，所以ptrarr[0][0]就是arr[0][0],即为1
---
# 题目八
- ***题目代码如下，求问这样写对比直接声明调用函数有什么好处***
```c
#include<stdio.h>
typedef void (*log)(const char* msg);
void error_log(const char* msg) {
	printf("[ERROR] %s\n", msg);
}
void debug_log(const char* msg) {
	printf("[DENUG] %s\n", msg);
}
void print_log(log func_log, const char* msg) {
	func_log(msg);
}
int main() {
	const char* msg = "hello 3G!";
	print_log(debug_log, msg);
	print_log(error_log, "YOLO");
	return 0;
}
```
- 分析如下
1. typedef用来打标签，为一个输入类型为const char*的函数打标签为log
2. print_log的调用中，传入值有两个，一个是待打印的字符串，一个是待传入的函数，这个函数将会为传入的log类型的函数传入待打印的字符串
> print_log起到一个中转站的作用，可以根据不同要求传入不同函数并且在形式上达到统一。
---
# 题目九
- ***题目代码如下，求问程序输出***
```c
#include<stdio.h>
#define SQUARE(x) x * x
#define SWAP(a, b, t) do {t = a; a = b; b = t; }while(0)
#define N 4
#define TO_STR(x) #x
#define MAGIC_CAL(a, b) (SQUARE(a) + SQUARE(b))
#define WELCOM(x) printf("Welcom to" TO_STR(3G) "%d!\n",x)
int main() {
	int i = 2, j = 4, tmp = 0;
	if (SQUARE(++i) == SQUARE(--j)) {
		SWAP(i, j, tmp);
	}
	printf("i=%d,j=%d\n", i--, j);
	WELCOM(SQUARE(i * i) * MAGIC_CAL(i, N));
	return 0;
}
```
- 分析如下
进入主函数，声明i和j的值，再进入if的判断语句，语句中是一个宏函数，替换为两个++i和两个--j，这在c语言标准里虽然是未定义的，但是本题我们将其视为先执行所有++或--之后再相乘，所以左边是16，右边是4，两者不相等，所以直接进行下一句，此时i为4，j为2，在输出这两个数值之后i再自减，最后一句WELCOM的SQU为3\*3\*3\*3=81，MAGIC_CAL为3\*3+4*4=25，相乘为2025，最后输出。
---
# 题目十
- ***题目代码如下，求问程序输出***
```c
#include<stdio.h>
struct SonStruct {
	char ch;
	int val;
};
struct MyStruct {
	unsigned int bit : 3;
	int* ptr;
	union {
		char arr[9];
		int val; 
	}unionSon; 
	SonStruct son; 
}; 
int main() {
	MyStruct s1 = { 0 }, s2 = { 0 };
	int number = 8;
	s1.bit = number;
	s1.ptr = &number;
	s1.unionSon.arr[1] = 1;
	s2 = s1;
	printf("Size of MyStruct:%zu\n\n", sizeof(s1));
	printf("bit (3-bit unsigned): %d\n", s1.bit);
	printf("unionSon.val (int): %d\n\n", s1.unionSon.val);
	printf("After ++(*s1.ptr) & (*s2.ptr)++ : %d %d\n\n", ++(*s1.ptr), (*s2.ptr)++);
	printf("Final number value: %d\n", number);
	return 0;
}
```
- 分析如下：
1. 先计算s1的大小：
	1. 计算Sonstruct：char大小为1，int大小为4，由于成员要储存在自身内存的整数位上，所以char补全为4字节，最终大小为8
	2. 计算mystruct：int在前面与8位的指针对齐，union联合体中由于共用内存共占9字节，与8字节对齐后多出一个，下面是以4字节对齐的sonstruct，由于成员要储存在自身内存的整数位上，给余出的一个字节补全到四个再接上一个sonstruct的int，最后新开一行填入int，由于结构体需要整体大小是最大成员的整数倍，所以将最后一行补全到8
	3. 综上所述一共开了五行，每行八字节对齐，共占用40字节内存，输出为40
2. s1的bit被赋值为8，但mystruct中的bit只能使用三个二进制位，无法表示8，所以输出是0
3. 由于union的数组第二个数被赋值为1，这里我们采用小端存储[^4]的方式来计算，由于联合体成员共用内存的特性，存入arr[1]中的1被存在了第二个地址的第一位上，而int读取也是按照小端顺序读取，即从低地址开始读取，于是它从第一个地址的八个字节第一位开始读，按照二进制转换规则转换，前面八个比特都是0（因为初始化为0）直到第二个字节的第一个比特读取到了arr[1]存入的1，它将其解释为第九位储存了1，于是转换为十进制时，把这里的1翻译为2的八次幂，即256
4. 第四个printf语句中，s2.ptr先解引用为8输出，然后自增为9，再前面自增为9输出，这里涉及到未定义行为，我们选择从右至左进行变量参数计算顺序
5. 最后由于上一步两次增值，number来到10
---
# 题目十一
- ***寻找x水壶和y水壶互相倒腾能得出的target升液体量***
```c
int Euclid(int x, int y) {
	if (0 == x % y) {
		return y; 
	} else {
		return Euclid(y, x % y);
	}
}
int main() {
	int x, y;
	scanf("%d %d", &x, &y);
	int target = Euclid(x, y);
	printf("%d", target); 
}
```
- 思路如下：
本题需要使用辗转相除法判断target是否为x和y的最大公约数的整数倍，这里我给出辗转相除法的递归实现如上。
---
# 题目十二
我们知道c语言库中的排序函数为快速排序，那么我给出一种快速排序的实现方法：
```c
int Sort(int* num, int left, int right) {
	int standard = num[left];
	while (left < right) {
		while (standard <= num[right] && left < right) {
			right--;
		}
		num[left] = num[right];
		while (standard >= num[left] && left < right) {
			left++;
		}
		num[right] = num[left];
	}
	num[left] = standard;
	return left;
}
void Quick_Sort(int* num, int left, int right) {
	if (left < right) {
		int standard = Sort(num, left, right);
		Quick_Sort(num, left, standard - 1);
		Quick_Sort(num, standard + 1, right);
	}
}
int main() {
	int num[] = { 4, 6, 8, 1, 0, 9, 2, 3, 7, 5 };
	Quick_Sort(num, 0, 9);
	int i;
	for (i = 0; i < 10; i++) {
		printf("%d", num[i]);
	}
	return 0;
}
```
> 排序代码如上，主要实现方法为递归，每次调用将小于基准值的数据列于基准值左侧，大于的列于右侧，再对每个小数组使用分治的思想，直到数组不可再分最终排序完成。

[^1]:如果进行逻辑判定时进行一部分后已经可以判断出结果一定为某个状态时程序将直接跳出逻辑判断，不进行后方语句执行，如`a&&b`，如果已经判定a为假，则b将不被判断，因为&&运算符需要两侧同时为真时才输出为真。
[^2]:异或运算指的是两个二进制数按位对应运算，如果两个中有且仅有一个为1时运算结果为1，其他为0
[^3]:或运算与异或相似，但或运算当两个运算位上的数值有一个为1时结果就为1
[^4]: 大/小端存储：计算机存储数据的两种方式，大端为高位数据存于低地址，小端则是低位数据存于低地址