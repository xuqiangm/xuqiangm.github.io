---
title: C++ 内存模型
date: 2016-03-20
categories: c++
---

一直以来对内存模型的认识总是从很局部的，想着还是需要总结梳理一下的，所以阅读了《C++ Prime Plus(第六版)》中的第9章，本文主要是对这一章中内存模型的进行梳理。

C++ 使用三种（C++11中增加到4种）不同的方案来存储数据，这些方案的区别就在于数据保留在内存中的时间：

1. 自动存储持续性
2. 静态存储持续性
3. 动态存储持续性

*注：*C++11 中增加了第4种，线程存储持续性，本文不讨论

接下来将会依次介绍这三种存储方案

不过在介绍这些之前，我们需要了解3种连接性：

- 外部链接性：可在其它文件中访问
- 内部连接性：只能在当前文件中访问
- 无链接性：只能在当前函数或代码块中访问

# 1. 自动存储持续性

在默认的情况下，在函数中声明的函数参数和变量的存储持续性为自动，作用域为局部，没有连接性。可以通过下面这个例子来说明：

```c++
#include <iostream>
using namespace std;

void oil(int x){
	int texas = 5;

	cout<<"在 oil() 中, texas的值为："<<texas<<"，texas的地址为："<<&texas<<endl;
	cout<<"在 oil() 中, x的值为："<<x<<"，x的地址为："<<&x<<endl;

	{
		int texas = 13;
		cout<<"在 block 中, texas的值为："<<texas<<"，texas的地址为："<<&texas<<endl;
		cout<<"在 block 中, x的值为："<<x<<"，x地址为："<<&x<<endl;
	}
}

int main(){
	int texas = 31;
	int year = 2016;
	cout<<"在 main() 中, texas的值为："<<texas<<"，texas的地址为："<<&texas<<endl;
	cout<<"在 main() 中, year的值为："<<year<<"，year的地址为："<<&year<<endl;

	oil(texas);

	return 0;
}
```
![](http://7xrvqe.com1.z0.glb.clouddn.com/16-03-20-memory-model-namespace%E5%B1%80%E9%83%A8%E5%8F%98%E9%87%8F%E8%BF%90%E8%A1%8C%E7%BB%93%E6%9E%9C.png)

由于自动变量的数目随函数的开始和结束而增减，因此程序必须在运行时自动变量进行管理，常见的是使用栈来进行管理，下图截自《C++ Primer Plus（第6版）》308页，该图可以形象生动的说明其中的过程

![](http://7xrvqe.com1.z0.glb.clouddn.com/16-03-20-memory-model-namespace-stack%E4%BD%BF%E7%94%A8%E6%A0%88%E4%BC%A0%E9%80%92%E5%8F%82%E6%95%B0.png)

最后，关于自动存储持续性，还有一点需要说明的是，可以使用 register 显式地指出变量是自动的， register 以前是用来建议编译器使用 CPU 寄存器来存储自动变量的，C++11 后废弃了这个用法，而成为指出变量是自动的关键字。

# 2. 静态持续变量

由于静态变量的数目在程序运行期间是不变的，因此程序不需要使用特殊的装置（如栈）来管理它们。编译器将分配固定的内存块来存储所有的静态变量，这些变量在整个程序执行期间一直存在。这一小节大致要说明的是下面这三种有着不同链接性的静态持续变量



存储描述 | 持续性 | 作用域 | 连接性 | 如何说明
---------|--------|--------|--------|---------
静态，无链接性   | 静态 | 代码块 | 无 | 在代码块中，使用关键字 static
静态，外部链接性 | 静态 | 文件 | 外部 | 不在任何函数内
静态，内部链接性 | 静态 | 文件 | 内部 | 不在任何函数内，使用关键字 static




不过，在介绍着三种之前，关于**静态变量的初始化**，还是有必要说一下的。

首先，所有静态变量都被零初始化，而不管程序员是否显式地初始化了它们，接下来，如果使用常量表达式初始化了变量，且编译器仅根据文件内容（包括被包含的头文件）就可计算表达式，编译器将执行常量表达式初始化，必要时，编译器将执行简单的计算，如果没有足够的信心，变量将被动态初始化。如下面的代码：

```c++
#include <iostream>
int x;		//初始化为零
int y = 5;	//常量表达式初始化
long z = 13*13;	//常量表达式初始化
static double pi = 4.0*atan(1.0);	//动态初始化
```

品尝完“开胃菜”，下面我们进入这一小节的“正菜”

## 2.1 静态持续性、外部连接性

首先我们来看一段代码：

```c++
//file01.cpp
extern int cats = 20;  //extern 可以省略
in dogs = 22;

//file02.cpp
extern int cats;
extern ind dogs;
```

在解释这段代码之前，我们需要引入“单定义规则”

> “单定义规则”（One Definition Rule）：变量只能有一次定义。

为满足这个规则，C++ 提供了两种变量声明，。一种是定义声明或简称定义，它给变量分配存储空间；另一种是引用声明或简称声明，它不给变量分配存储空间，因为它引用已有的变量，引用声明使用关键字 extern。在上面代码中 file01.cpp 中是定义声明， file02.cpp 是引用声明。

## 2.2 静态持续性、内部链接性

### 2.2.1 static

实现静态变量的内部链接性，是通过关键字 static 实现的。

首先我们看一份代码

```c++
//file1
int a = 20;

//file2
int a = 5;
```

上面的做法将失败，因为违反了单定义规则，程序中将包含两个 a 的定义。我们可以在通过 static 关键字来避免这个错误、

```c++
//file1
int a = 20;

//file2
static int a = 5;
```

通过 static 关键字，使变量的链接性设为内部。在多文件程序中，链接性为内部的变量只能在所属文件中使用，连接性为外部的变量可以在其他文件中使用。

### 2.2.2 const

在创建静态且连接性为内部的变量时，除了使用 static 关键字外，还有一个我们熟悉的小精灵也可以达到同样的效果，const 关键字。

默认情况下，const 全局变量的连接性为内部的。为什么 C++ 要这么设定呢？如果我们要做这么一件事：将一组常量放在头文件中，并在同一程序中的多个文件中使用该头文件，那么预处理器将头文件中的内容包含到每个源文件中后，所有的源文件都将包含类似下面这样的定义：

```c++
const int fingers = 10;
const char* warning = "Wak!";
```

如果全局 const 声明的连接性是外部的话，那么会出错，那么就需要提供两份定义，一份类似上面代码，还有一份类似下面的代码：

```c++
extern int fingers;
extern char* warning;
```

而将全局 const 声明的链接性是内部的话，可以在所有文件中使用相同的声明。

如果程序员希望某个变量的连接性为外部的，则可以使用 extern 关键字来覆盖默认的内部链接性：

```c++
extern const int fingers = 10;
```

在这种情况下，必须在所有使用该常量的文件中使用 extern 关键字来声明它，并且只有一个文件可对其进行初始化。

## 2.3 静态存储持续性、无链接性

这种变量是这么创建的，将 static 限定符用于在代码快中定义的变量，这样将会使局部变量的存储持续性为静态的。这意味着虽然该变量只在该代码块中可用，但它在该代码块中不处于活动状态时仍然存在。另外，如果初始化了静态局部变量，则程序只在启动时进行一次初始化。

```c++
#include <iostream>
using namespace std;

void test(){
	static int val = 0;
	val++;
	cout<<val<<endl;
}

int main(){
    test();
    test();
    return 0;
}

//结果为： 
//1 2
```

# 3. 动态存储持续性

动态存储变量主要是通过 new （或C 中的 malloc()）创建，虽然通常情况下，在程序结束时，分配的内存会被系统释放，但安全的做法是用 delete 释放。

下面主要介绍 new 运算符

## 3.1 new 运算符

```c++
int* pi = new int{6};   //会被转化成 int* pi = new(sizeof(int));
int* arr = new int[4]{1,2,3,4};   //会被转化成 int* arr = new(4*sizeof(int));
```

new 如果创建失败会返回 std::bad_alloc（C++11之前的做法是返回空指针）。

## 3.2 定位 new 运算符

通常,new 负责在堆中找到一个足够满足要求的内存块，而定位 new 运算符能够在使用指定位置的空间，如静态存储区。使用定位 new 运算符需要包含头文件 new。

```c++
#include <iostream>
#include <new>
using namespace std;

struct chaff{
	char dross[20];
	int slag;
};

char buffer1[50];	//50字节的静态存储区
char buffer2[500];	////500字节的静态存储区

int main(){
	cout<<"buffer1[50] address:"<<reinterpret_cast<void *>(buffer1)<<endl;
	cout<<"buffer1[500] address:"<<reinterpret_cast<void *>(buffer2)<<endl;

    chaff* p1,* p2;
    int* p3,* p4;

    p1 = new chaff;
    p3 = new int[20];

	cout<<"p1 address:"<<p1<<endl;
	cout<<"p3 address:"<<p3<<endl;

    p2 = new (buffer1) chaff;	//调用的是 new(sizeof(chaff),buffer)
    p4 = new (buffer2) int[20]; ////调用的是 new(20*sizeof(int),buffer)

    cout<<"p2 address:"<<p2<<endl;
	cout<<"p4 address:"<<p4<<endl;

	p2 = new (buffer1) chaff;
    p4 = new (buffer2) int;

    cout<<"reapply,p2 address:"<<p2<<endl;
	cout<<"reapply,p4 address:"<<p4<<endl;

	delete p1;
	delete p3;

    return 0;
}
```

![](http://7xrvqe.com1.z0.glb.clouddn.com/16-03-20-memory-model-namespacenew.png)

通过示例可以看出 p2, p4 使用静态存储区中的内存，并且在再次使用该区域内存是时，起始的位置相同。并且，由于 p2,p4 指向的是静态存储区，所以不能使用 delete 释放。

# 4. 补充说明

## 4.1 volatile 关键字

关键字 volatile 表明，即使程序代码没有对内存单元进行修改，其值也可能发生变化。例如，将一个指针指向某个硬件位置，在这种情况下，硬件（而不是程序）可能修改其中的内容。编译器默认的优化方式是，如果编译器发现程序在几条程序中两次使用了某个变量的值，则编译器可能不是让程序查找这个值两次，而是将这个值缓存到寄存器中。这种优化假设变量的值在这两次使用之间不会变化，如果不将变量声明为 volatile，则编译器将进行这种优化，而设置为 volatile 则相当于告诉编译器，不要进行这种优化。

## 4.2 函数和链接性

函数的链接性默认为外部的，即可通过使用 extern 使用另一个文件的函数。也可以是用 static 将函数的连接性设置为内部的。

## 4.3 语言链接性

由于在 C 语言中，一个名称只能对应一个函数，而在 C++ 中一个名称可以对应多个函数（重载函数）。因此 C 语言编译器和 C++ 编译器执行名称修饰时生成的符号名称不同，因此在 C++ 中连接 C 编译的函数，需要使用 extern 关键字。

```c++
extern "C" void func(int);
```

### 参考文献
《C++ Prime Plus(第六版)》