---
title: C++11 新特性：右值引用
date: 2016-04-04
categories: C++
---

本文是对于《深入理解C++11：C++11新特性解析与应用》第三章第三小节“右值引用：移动语义和完美转发”的梳理。

# 1. 什么是右值引用

## 左值、纯右值、将亡值

左值（lvalue）、右值（rvalue）对于我们来说，并不陌生。常见的有两种分辨方法：

1. 在赋值表达式中，出现在星号左边的就是“左值”，而在等号右边的，则称为“右值”。
2. 可以取地址的、有名字的就是左值，反之，不能取地址的、没有名字的就是右值。

在 C++11 中，右值由两个概念构成，即，将亡值（xvalue）、纯右值（prvalue）。因此在 C++11 的程序中，所有的值必属于左值、将亡值、纯右值三者之一。下面举例说明下着三种值的例子。

- 纯右值就是 C++98 标准中右值的概念。如，非引用返回的函数返回的临时变量值，一些运算表达式产生的临时变量值，不跟对象关联的字面常量值（如，2、'c'、true），类型转换函数的返回值、lambda 表达式等。
- 将亡值则是 C++11 新增的跟右值引用相关的表达式，这样的表达式通常是将要被移动的对象（移为他用）。如，返回右值引用 T&& 的函数返回值，std::move 的返回值，转换为 T&& 的类型转换函数的返回值。
- 出去以上剩余的，可以表示函数、对象的值都属于左值。

## 左值引用和右值引用

右值引用和左值引用都是属于引用类型。无论是声明一个左值引用还是右值引用，都必须立即进行初始化。而其原因可以理解为是引用类型本身自己并不拥有所绑定对象的内存，只是改对象的一个别名。左值引用是具名变量值的别名，而右值引用是不具名（匿名）变量的别名。

## C++11中引用类型及其可以引用的值类型

![](http://7xrvqe.com1.z0.glb.clouddn.com/16-4-4/22188018.jpg)

从表中我们可以看出，常量左值引用是一个“万能引用”，但是引用的是右值，那么就变成常量右值，而常量右值通常是没有什么用途的。

在 <type_traits> 头文件中提供了3个模板类：is_rvalue_reference、is_lvalue_reference、is_reference，分别用来判断右值引用、左值引用、引用。

```c++
#include <iostream>
#include <type_traits>
using namespace std;

int main(){
	//cout<<boolalpha;
    string&& a= "我是右值引用";
    cout<< is_rvalue_reference<decltype(a)>::value<<endl;
    return 0;
}

//输出是：1
```
右值引用最大的两个用处是：转移语义和完美转发。

## std::move：强制转化为右值

在 C++11 中，标准库在 <utility> 中了一个有用的函数 std::move，它唯一的功能是将一个左值强制转化为右值引用，从实现上讲，move 基本等同于一个类型转化：

```c++
static_cast<T&&>(lvalue);
```

值得注意的是，被转化的值，其生命期并没有随着左右值得转化为改变。

```c++
#include <iostream>
using namespace std;

class Moveable{
public:
	Moveable():i(new int(3)){}
	~Moveable(){ delete i;}
	Moveable(const Moveable& m):i(new int(*m.i)) {}
	Moveable(Moveable&& m):i(m.i){
		m.i = nullptr;
	}

	int* i;
};

int main(){
	Moveable a;
	Moveable c(move(a));		//会调用移动构造函数
	cout<<*a.i<<endl;	//运行时错误
    return 0;
}
```

# 2. 转移语义

```c++
#include <iostream>
using namespace std;

class HasPtrMem{
public:
	HasPtrMem():d(new int(0)){
		cout<<"Construct"<<++n_cstr<<endl;
	}
	HasPtrMem(const HasPtrMem& h):d(new int(*h.d)){
		cout<<"Copy construct:"<<++n_cptr<<endl;
	}
	~HasPtrMem(){
		cout<<"Destruct:"<<++n_dstr<<endl;
	}

private:
	int* d;
	static int n_cstr;
	static int n_cptr;
	static int n_dstr;
};

int HasPtrMem::n_cstr = 0;
int HasPtrMem::n_cptr = 0;
int HasPtrMem::n_dstr = 0;

HasPtrMem GetTemp(){
	return HasPtrMem();
}

int main(){
	HasPtrMem h = GetTemp();
    return 0;
}
```

运行结果：

![](http://7xrvqe.com1.z0.glb.clouddn.com/16-4-4/2099072.jpg)

上面是没有使用移动语义优化的版本，在实际运行时，如果没有关闭编译器的返回值优化（RVO，Return Value Optimization，或，NRVO,Named Return Value Optimization）的话，得到的结果是：

![](http://7xrvqe.com1.z0.glb.clouddn.com/16-4-4/30053813.jpg)

因为编译器将会对返回值进行优化，h 直接使用 HasPtrMem() 时生成的对象地址。为了达到实验效果，编译的时候加上 -fno-elide-constructors 关闭该优化功能（虽然这个功能很棒）。

首先调用 GetTemp() 函数时，首先是调用 HasPtrMem 的构造函数创建一个临时变量，之后返回该临时变量，调用 HasPtrMem 的拷贝构造函数创建另一个临时变量，返回到 main 函数中，之后调用第一个临时变量的析构函数，销毁创建的第一个变量。之后再调用一次拷贝构造函数，传递的是第二个临时变量，创建出对象 h，之后将第二个临时变量析构，在程序结束之后，析构对象 h。

下面我们使用移动语义来对这段代码进行优化

```c++
#include <iostream>
using namespace std;

class HasPtrMem{
public:
	HasPtrMem():d(new int(3)){
		cout<<"Construct"<<++n_cstr<<endl;
	}
	HasPtrMem(const HasPtrMem& h):d(new int(*h.d)){
		cout<<"Copy construct:"<<++n_cptr<<endl;
	}
	HasPtrMem(HasPtrMem&& h):d(h.d){	//移动构造函数
		h.d = nullptr;					//将临时值得指针成员置空
		cout<<"Move Construct:"<<++n_mstr<<endl;
	}
	~HasPtrMem(){
		delete d;
		cout<<"Destruct:"<<++n_dstr<<endl;
	}

	int* d;
	static int n_cstr;
	static int n_cptr;
	static int n_dstr;
	static int n_mstr;
};

int HasPtrMem::n_cstr = 0;
int HasPtrMem::n_cptr = 0;
int HasPtrMem::n_dstr = 0;
int HasPtrMem::n_mstr = 0;

HasPtrMem GetTemp(){
	HasPtrMem h;
	cout<<"Resource from GetTemp()"<<":"<<hex<<h.d<<endl;
	return h;
}

int main(){
	HasPtrMem h = GetTemp();
	cout<<"Resource from main()"<<":"<<hex<<h.d<<endl;
    return 0;
}
```

结果为：

![](http://7xrvqe.com1.z0.glb.clouddn.com/16-4-4/53942185.jpg)

从结果上看，首先调用 GetTemp() 函数，会构造一个局部变量，首地址为 Ox581728 ，之后返回时，调用的是移动构造函数生成一个临时变量，不过该临时变量使用的是之前创建的堆空间，这就避免了一次堆空间的释放和申请；之后析构局部变量。之后再 main() 函数中，使用刚才创建的临时变量去构造对象 h，调用的是移动构造函数，因此，h 使用的是临时对象的堆空间，即首地址仍是 Ox581728，又避免了一次堆空间的释放和申请。因此，提高了效率。

下图很好的说明两个版本的区别：

![](http://7xrvqe.com1.z0.glb.clouddn.com/16-4-4/90655673.jpg)

## 补充一些

需要要补充的是,默认情况下，编译器会为程序员隐式地生成一个（隐式表示如果不被使用则不生成）移动个构造函数。并且，在 C++11 中，拷贝构造/赋值和移动构造/赋值函数必须同时提供，或者同时不提供，程序员才能保证类同时具有拷贝和移动语义。只声明一种的话，类都仅能实现一种语义。只有移动语义构造的类型往往都是“资源型”的类型，比如说智能指针，文件流等。

在标准库的头文件 <type_traits> 里，我们还可以通过一些辅助的模板类来判断一个类型是不是可以移动的。比如 is_move_constructible、is_trivially_move_constructible、is_nothrow_move_constructible，使用方法仍然是使用其成员 value。比如：

```c++
cout<<is_move_constructible<UnknownType>::value;
```

## 关于异常

对于移动构造函数来说，抛出异常有时是一件危险的事情。因为可能移动语义还没完成，一个异常却抛出来了，这就会导致一些指针就成为悬挂指针，因此程序员应该尽量编写不抛出异常的移动构造函数，通过为其添加一个 noexcept 关键字，可以保证移动构造中抛出来的异常会直接调用 terminate 程序终止运行，而不是造成悬挂指针的状态。在标准库中，我们还可以用一个 std::move_if_noexcept 的模板函数替代 move 函数。该函数在类的移动构造函数没有 noexcept 关键字修饰时返回一个左值引用从而使变量可以使用拷贝语义，而在类的移动构造函数有 noexcept 关键字时，返回一个右值引用，从而使变量可以使用移动语义。

```c++
#include <iostream>
#include <utility>
using namespace std;

struct  Maythrow{
	Maythrow() {}
	Maythrow(const Maythrow&){
		cout<<"Maythrow copy constructor."<<endl;
	}
	Maythrow(Maythrow&&){
		cout<<"Maythrow move constructor."<<endl;
	}
};

struct Nothrow{
	Nothrow() {}
	Nothrow(Nothrow&& ) noexcept{
		cout<<"Nothrow move constructor."<<endl;
	}
	Nothrow(const Nothrow&){
		cout<<"Nothrow copy constructor."<<endl;
	}
};

int main(){
	Maythrow m;
	Nothrow n;

	Maythrow mt = move_if_noexcept(m);	//Maythorow copy constructor
	Nothrow nt = move_if_noexcept(n);   //Nothorow move constructor
	return 0;
}

//运行结果
//Maythrow copy constructor
//Nothrow move constructor
```

# 3. 完美转发



### 参考资料

- 《深入理解C++11：C++11新特性解析与应用》




