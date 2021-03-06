---
title: 《编译原理》速查笔记
date: 2017-08-12
categories: 基础知识
---

> **目录** 


> [1. 第一单元：编译器介绍](#anchor1)  
> *[1.1 编译器的核心功能](#anchor2)*  
> *[1.2 编译器结构](#anchor3)*  
> [2. 第二单元：词法分析(概念)](#anchor4)  
> *[2.1 简介](#anchor5)*  
> *[2.2 手工构造法](#anchor6)*  
> *[2.3 正则表达式](#anchor7)*  
> *[2.4 有限状态自动机](#anchor8)*  
> [3. 第三单元：词法分析(过程)](#anchor9)   
> *[3.1 RE转换成NFA：Thompson算法](#anchor10)*   
> *[3.2 NFA转换成DFA：子集构造算法](#anchor11)*   
> *[3.3 DFA的最小化：Hopcroft算法](#anchor12)*  
> [4. 第四单元：语法分析(概念)](#anchor13)   
> *[4.1 语法分析器的任务](#anchor14)*   
> *[4.2 上下文无关文法](#anchor15)*  
> *[4.3 分析树与二义性](#anchor16)*  
> *[4.4 自顶向下分析](#anchor17)*  
> *[4.5 递归下降分析](#anchor18)*  

<a name="anchor1"></a>
# 第一单元：编译器介绍

<a name="anchor2"></a>
### 1.1 编译器的核心功能

![](http://7xrvqe.com1.z0.glb.clouddn.com/%E7%BC%96%E8%AF%91%E5%99%A8%E7%9A%84%E6%A0%B8%E5%BF%83%E5%8A%9F%E8%83%BD.svg)

<a name="anchor3"></a>
### 1.2 编译器结构

**1.2.1 一种没有优化的编译器结构**

![](http://7xrvqe.com1.z0.glb.clouddn.com/%E6%B2%A1%E6%9C%89%E4%BC%98%E5%8C%96%E7%9A%84%E7%BC%96%E8%AF%91%E5%99%A8%E7%BB%93%E6%9E%84.svg)

**1.2.2 一种更复杂的编译器结构**

![](http://7xrvqe.com1.z0.glb.clouddn.com/%E4%B8%80%E7%A7%8D%E6%9B%B4%E5%A4%8D%E6%9D%82%E7%9A%84%E7%BC%96%E8%AF%91%E5%99%A8.svg)

**1.2.3 编译器例子**

在这一小结中，课程通过一个简单的编译器例子，来讲解。

该编译器实例有如下规则：

- 源语言：加法表达式语言Sum
	- 两种语法形式：
		- 整型表达式： n
		- 加法： e1+e2
- 目标机器：
	- 栈式计算机Stack
		- 一个操作数栈
		- 两条指令
			- 压栈指令：push n
			- 加法指令： add

*任务：编译程序1+2+3到栈式计算机*

**阶段一：**词法语法分析

词法分析，1+2+3这个表达式有5个部分组成，“1”、“+”、“2”、“+”、“3”
语法分析，就是将这个表达式读进去，看是否符合语法规则

**阶段二：**语法树构建

![](http://7xrvqe.com1.z0.glb.clouddn.com/2017-08-12-compile_principle_compiler_structure_3.svg)

**阶段三：**代码的实现

两条规则：
1> 数字n ==> push n
2> 符号+ ==> add

后序遍历语法树：
```
push 1
push 2
add
push 3
add
```

**阶段四：代码优化（常量折叠优化）**

例如：1+2可以直接常量计算成3传入，1+2==>3

<a name="anchor4"></a>
# 第二单元：词法分析（概念）

<a name="anchor5"></a>
### 2.1 简介

**2.1.1 编译器的阶段**

源程序--->前端--->中间表示--->后端--->目标程序

**2.1.2 前端**

![](http://7xrvqe.com1.z0.glb.clouddn.com/%E7%BC%96%E8%AF%91%E5%99%A8-%E5%89%8D%E7%AB%AF.svg)

**2.1.3 词法分析器的任务：字符流到记号流**

- 字符流：
	- 和被编译的语言密切相关（ASCII，Unicode，or...）
- 记号流：
	- 编译器内部定义的数据结构，编码所识别出的词法单元

例子：

```c++
if(x>5)
	y = "hello";
else
	z = 1;
```

词法分析后得到：

```
IF LPAREN IDENT(x) GT INT(5) RPAREN
	IDENT(y) ASSIGN STRING("hello") SEMICOLON
ELSE
	INDET(z) ASSIGN INT(1) SEMICOLON EOF
```

记号的数据结构定义

```c
enum kind {IF, LPAREN, ID, INTLIT, ...}
struct token
{
	enum kind k;
	char *lexeme;
};
```

<a name="anchor6"></a>
### 2.2 手工构造法

**2.2.1 词法分析器的实现方法**

- 手工编码实现法
	- 相对复杂、且容易出错
	- 但是目前非常流行的实现方法
		-GCC, LLVM
- 词法分析器的生成器
	- 可快速原型、代码量较少
	- 但较难控制细节

**2.2.2 转移图**

![](http://7xrvqe.com1.z0.glb.clouddn.com/2017-08-12-compile_principle_compiler_structure_5.svg)

其中“*”表示回退

**2.2.3 关键字表算法**

- 对给定语言中所有的关键字，构建关键字构成的哈希表H
- 对所有的标识符和关键字，先统一按标识符的转移图进行识别
- 识别完成后，进一步查表H看是否是关键字
- 通过合理的构造哈希表H（完美哈希），可以O(1)时间完成

<a name="anchor7"></a>
### 2.3 正则表达式

正则表达式是属于词法分析器的生成器中的声明式的规范

**2.3.1 正则表达式的定义**

> 对于给定的字符集 Σ={c1, c2, ... , cn}
归纳定义：
空串ε是正则表达式
对于任意c∈Σ，c是正则表达式
如果M和N是正则表达式，则以下也是正则表达式

- 选择	M \| N = {M, N}
- 链接	MN = {mn \| m∈M, n∈N}
- 闭包	M* = {ε， M, MM, MMM, ...}

**2.3.2 正则表达式的形式**

```
e --> ε
	| c
	| e | e
	| e e
	| e*
```

**2.3.3 语法糖**

可以引入更多的语法糖，来简化构造


- [c1-cn] == c1\|c2\|...\|cn
- e+      == 一个或多个e
- e?	  == 零个或多个e
- "a*"    == a*自身，不是a的Kleen闭包
- e{i,j}  == i到j个e的连接
- .       == 除 '\n'外的任意字符

<a name="anchor8"></a>
### 2.4 有限状态自动机

**2.4.1 有限状态自动机（FA）**

![](http://7xrvqe.com1.z0.glb.clouddn.com/%E6%9C%89%E9%99%90%E7%8A%B6%E6%80%81%E8%87%AA%E5%8A%A8%E6%9C%BAFA--%E5%85%83%E7%BB%84%E8%A1%A8%E7%A4%BA.svg)

- 第一个例子：

![](http://7xrvqe.com1.z0.glb.clouddn.com/%E6%9C%89%E9%99%90%E7%8A%B6%E6%80%81%E8%87%AA%E5%8A%A8%E6%9C%BA%28FA%29.svg)

根据FA的元组表达式：

> M = (Σ，S, q0, F, δ)

有

```
Σ = {a, b}
S = {0, 1, 2}
q0 = 0
F = {2}
δ = { (q0, a)-->q1, (q0, b)-->q0,
	  (q1, a)-->q2, (q1, b)-->q1,
	  (q2, a)-->q2, (q2, b)-->q2}
```

- 第二个例子：

![](http://7xrvqe.com1.z0.glb.clouddn.com/2017-08-12-compile_principle_compiler_fa_11.svg)

对于这样的例子，有

```
Σ = {a, b}
S = {0, 1}
q0 = 0
F = {1}
δ = { (q0, a)-->{q0, q1}),
	  (q0, b)-->{q1},
	  (q1, b)-->{q0, q1}}
```

- 总结
	- 确定状态有限自动机DFA：对于任意的字符，最多有一个状态可以转移（如例子1）
	- 非确定的有限状态自动机NFA：对于任意的字符，有多于一个状态可以转移（如例子2）

- DFA的实现

最简单的一种实现方式是用有向图的方式来实现，对于例子1中的转移图来说，可以用以下表来表示

状态\字符 | a |b
---------| --|--
0		|	1|0
1		|   2|1
2		|   2|2

<a name="anchor9"></a>
# 第三单元：词法分析（过程）

自动生成所涉及的流程如下：

![](http://7xrvqe.com1.z0.glb.clouddn.com/%E8%87%AA%E5%8A%A8%E7%94%9F%E6%88%90%E8%BF%87%E7%A8%8B.svg)

<a name="anchor10"></a>
### 3.1 RE转换成NFA：Thompson算法

Thompson算法是基于对RE的结构做归纳的算法，对基本的RE直接构造，对复合的RE递归构造。

具体推导的对应关系如下。

![](http://7xrvqe.com1.z0.glb.clouddn.com/Thompson.svg)

根据以上关系，可以根据正则表达式a(b\|c)*推导出NFA

![](http://7xrvqe.com1.z0.glb.clouddn.com/Thompson-example.svg)

<a name="anchor11"></a>
### 3.2 NFA转换成DFA：子集构造算法

算法可以分为两步：

1. 对于输入的一个字符，通过原来的NFA看集合中的节点可以转化成哪些节点，得到一个新的集合。
2. 对新集合中每一个节点求ε闭包


对于3.1中a(b\|c)*推导出的NFA，可以利用自己构造算法转化成DFA，具体过程如下：


```
n0--(a)-->{n1,n2,n3,n4,n6,n9}
将状态n0记作q0，{n1,n2,n3,n4,n6,n9}记作q1
则有
q1--(b)-->{n5,n8,n9,n3,n4,n6}
将{n5,n8,n9,n3,n4,n6}记作q2
q1--(c)-->{n7,n8,n9,n3,n4,n6}
将{n7,n8,n9,n3,n4,n6}记作q3
则有
q2--(b)-->q2
q2--(c)-->q3
q3--(b)-->q2
q3--(c)-->q3
```

最后结果如下：

![](http://7xrvqe.com1.z0.glb.clouddn.com/%E5%AD%90%E9%9B%86%E6%9E%84%E9%80%A0%E7%AE%97%E6%B3%95.svg)

其中q1,q2,q3为接收状态

<a name="anchor12"></a>
### 3.3 DFA的最小化：Hopcroft算法

Hopcroft算法是基于等价类的思想

**例子1：**

根据上一小节最终得到的DFA，可以进行最小化。

1. 首先将所有的状态化为非接收状态N和接收状态A，得到如下结果

![](http://7xrvqe.com1.z0.glb.clouddn.com/Hopcroft%E7%AE%97%E6%B3%95%E4%BE%8B%E5%AD%901.svg)
2. 可以看出，N不能在进行切分，看A能不能进行切分
3. A中的字符b，c都不能对A进行切分，所以，A不能再进行切分了。

因此最终最小化的结果为：

![](http://7xrvqe.com1.z0.glb.clouddn.com/Hopcroft%E7%AE%97%E6%B3%95%E4%BE%8B%E5%AD%901%E7%BB%93%E6%9E%9C.svg)

*注：*所谓切分指的是一个集合中每个元素在接收某个字符后产生转化后的状态不在同一个集合中，就会产生切分。

**例子2：**

![](http://7xrvqe.com1.z0.glb.clouddn.com/Hopcroft%E7%AE%97%E6%B3%95%E4%BE%8B%E5%AD%902.svg)

1. 首先划分N:{q0,q1,q2,q4}和A:{q3,q5}
2. 对于集合A已经不能划分了，对于集合N,字符f不能切分，字符i不能切分，字符e,q1和q2,q4产生了分歧，因此字符e可以把q2,14切分出来，组成集合S。即有N:{q0,q1}，S:{q2,q4},A{q3,q5}
3. 集合S不能再进行切分了，**对于集合N，q0不接受字符e，状态还在集合N中，但是q1接受字符e后，转化到了集合S，产生了分歧，因此字符e可以将q0,q1切分。**

最终结果为：

![](http://7xrvqe.com1.z0.glb.clouddn.com/Hopcroft%E7%AE%97%E6%B3%95%E4%BE%8B%E5%AD%902%E7%BB%93%E6%9E%9C.svg)

<a name="anchor13"></a>
# 第四单元：语法分析（概念）

<a name="anchor14"></a>
### 4.1 语法分析器的任务

语法分析器的任务是将记号流根据语言的语法规则生成语法树。

<a name="anchor15"></a>
### 4.2 上下文无关文法

**4.2.1 乔姆斯基文法体系**

![](http://7xrvqe.com1.z0.glb.clouddn.com/%E4%B9%94%E5%A7%86%E6%96%AF%E5%9F%BA%E6%96%87%E6%B3%95%E4%BD%93%E7%B3%BB.svg)

文法的定义如下：

>文法G是一个四元组：
>G = (T, N, P, S)
>- T是终结符集合
>- N是非终结符集合
>- P是一组产生式规则
>- S是唯一的开始符号

具体分类准则如下:

- **0型文法（任意文法）**
	- \|α\|≠0
	- 其识别系统图灵机。
- **1型文法（上下文有关语言）**
	- \|α\|≤\|β\|，其中\|α\|表示的是α的长度
	- 其识别系统是有限有界自动机。
- **2型文法（上下文无关语言）**
	- α∈N，\|α\|=1
	- 其识别系统是下推自动机。
- **3型文法（正则语言）**：
	- A→a或A→aB: G是右线性文法
	- A→a 或A→Ba: G是左线性文法
	- 其识别系统是有限自动机


**4.2.2 上下文无关文法的例子**

产生式规则如下：

```
E -> num
   | id
   | E + E
   | E * E
```

则有：

```
G = (N, T, P, S)
非终结符： N = {E}
终结符： T = {num, id, +, *}
开始符号： E
产生式规则集合： 已有
```

**4.2.3 推导**

推导的概念是：

>给定文法G，从G的开始符号S开始，用产生式的右部替换左侧的非终结符。子过程不断重复，直到不出现非终结符为止，最终的串称为句子。

推导分为最左推导和最右推导。所谓最左推导指的是从句子的左侧开始推导，最右推导与之相反。

<a name="anchor16"></a>
### 4.3 分析树与二义性

**4.3.1 分析树**

> 推导可以表达成树状结构，和推导所用的顺序无关，其特点有：
> - 树中的每个内部节点代表非终结符
> - 每个叶子节点代表终结符
> - 每一步推导代表如何从双亲节点生成它的直接孩子节点

**例子：**

有如下文法：

```
E -> num
   | id
   | E + E
   | E * E
```

根据上述文法，推导句子*3+4\*5*

推导过程如下：

```
第一种推导过程：
E -> E + E
  -> 3 + E
  -> 3 + E * E
  -> 3 + 4 * E
  -> 3 + 4 * 5
第二种推导过程：
E -> E * E
  -> E + E * E
  -> 3 + E * E
  -> 3 + 4 * E
  -> 3 + 4 * 5
```

第一种推导过程产生的分析树为：

![](http://7xrvqe.com1.z0.glb.clouddn.com/%E5%88%86%E6%9E%90%E6%A0%91_1.svg)

第二种推导过程产生的分析树为：

![](http://7xrvqe.com1.z0.glb.clouddn.com/%E5%88%86%E6%9E%90%E6%A0%91_2.svg)

分析树的含义取决于树的后序遍历。

- 第一种推导过程的含义为3+(4\*5)=23
- 第二种推导过程的含义为(3+4)\*5=35

**4.3.2 二义性文法**

> 给定文法G，如果存在句子s,它有两棵不同的分析树，那么称G是二义性文法
> 从编译器角度，二义性文法存在问题：
> - 同一个程序会有不同的含义
> - 因此程序运行的结果不是唯一的

对于上述的文法，可以通过重写消除二义性

```
E -> E + T
   | T
T -> T * F
   | F
F -> num
   | id
```

<a name="anchor17"></a>
### 4.4 自顶向下分析

> 基本算法思想：从G的开始符号出发，随意推导出某个句子t，比较t和s，如果t==s，则回答“是”，否则回退再次比较。对应于分析树自顶向下的构造顺序。

由于该算法需要用到回溯，给分析效率带来问题，引出了递归下降分析算法和LL(1)分析算法。

<a name="anchor18"></a>
### 4.5 递归下降分析

> 算法基本思想是：每个非终结符构造一个分析函数，用前看符号指导产生式规则的选择。基本思想是利用分治的思想。

例如：

```
S -> N V N
N -> s
   | t
   | g
   | w
V -> e
   | d
```

算法的伪代码为：

```c++
parse_S()
	parse_N()
	parse_V()
	parse_N()

parse_N()
	token = tokens[i++]
	if (token==s || token==t ||
		token==g || token==w)
		return ;
	error("...");

parse_V()
	token = tokens[i++];
	...
```

### 参考

- [编译原理(网易云课堂)](http://mooc.study.163.com/course/USTC-1000002001#/info)
- [乔姆斯基语言体系](http://blog.csdn.net/guguant/article/details/55105786)


