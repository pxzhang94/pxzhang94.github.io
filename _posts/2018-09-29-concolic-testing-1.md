---
layout: post
title: "Concolic Testing 1"
subtitle: "Introduction"
author: "pxzhang"
header-img: "img/post-bg-halting.jpg"
header-mask: 0.3
tags:
  - Software Engineering
  - Concolic Testing
---

## 随机测试（Random Testing）
- 使用随机生成的测试输入执行程序，然后观察是否会出现错误状态
- 对于高可能性的错误，随机测试的代价很低，反之很高<br/>
比如说下面的第3行判断语句，对于随机测试来说，几乎不可能覆盖到其True分支
```
public static void example(int x, int y) {
	int[] array = new int[10];
	if (y == 42342531) {
		array[x] = y; 
	}
}
```

## 符号执行（Symbolic Execution）
- 利用符号变量来代替输入执行程序，然后使用约束求解器求解路径约束从而获得测试输入
- 符号执行的代价很高，但是可以“保证”到达给定节点<br/>
同样是上面那个程序，我们可以通过求解下面的路径约束：
```
y == 42342531 && (x<0 || x >= 10)
```
生成一个特定的测试输入来覆盖我们指定的路径，比如：
```
x == 10 and y == 42342531
```
但是目前符号执行面临两个问题，第一：由于需要利用符号变量来构建路径约束，所以显然其比随机测试需要更多的时间；第二：目前最常见的约束求解器（如Z3等）的求解能力仍十分有限，对于稍复杂的路径约束就束手无策。<br/>
常见的符号执行策略有：
- Random State Search
- Random Path Selection
- Depth First Search
- Subpath-Guided Search

所以自动化测试领域提出了concolic testing，我翻译为动态符号执行测试，也可以翻译为具体符号执行，因为concolic是concrete和symbolic的合成词。

## 动态符号执行测试（Concolic Testing）
- 集成了具体执行和符号执行两种策略
- 决定选择随机测试还是符号执行
- 如果选择符号执行，决定选择的程序执行路径
- 常见的动态符号执行测试工具：Microsoft SAGE和JDart等<br/>
比如说如下程序：
```
int h(int x, int y) {
	if (x != y) { 
		if (2*x == x + 10) { 
			abort(); /*error*/
		}
		else {
			return 2x+y; 
		}
	}
	else{
		return 2x;
	} 
}
```
其抽象语法树为：
![抽象语法树](http://pxzhang94.github.io/img/posts/concolic_testing/1.png)
如上图所示，黑色箭头表示随机测试，蓝色箭头表示符号执行.<br/>
常见的动态符号执行策略有：
- Directed Automated Random Testing
- Coverage-Optimized Search
- Generational Search
- Context-Guided Search

## 部分常见测试策略
事实上，绝大多数动态符号执行测试策略仅执行一遍随机测试，更多的是关注随后符号执行路径的调度，这导致相关算法性能距离[最优调度策略](http://pxzhang94.github.io/2018/10/08/concolic-testing-2)有极大的差距。我将使用以下函数作为例子简单阐述几种常见动态符号执行以及符号执行策略。
```
void myfunc(int x, int y){
1.    if(x==y){
2.        x++;
       }
3.    if ((x*x)%10==9){
4.        return;
       }
5.    if (x*y+3*y-5*x==15){  
6.        if (x%2==1 ||y%2==1){
7.           x = x-y;
           } 
       }
8.    return;
}
```
其抽象语法树为：
![抽象语法树](http://pxzhang94.github.io/img/posts/concolic_testing/2.png)

### [Subpath-Guided Search](http://pxzhang94.github.io/paper/concolic_testing/oopsla13-pgse.pdf)
该方法是符号执行测试调度策略，首先给出两个关键的定义：
- Length-n子路径：给定一条路径<s1, s2,…, sk>，那么其length-n 子路径就是<sk-n+1, sk-n+2, …, sk>
- 统计分析结构：结构e = (π, f) 用于统计分析，其中π是length-n 子路径，f是π被探索过的频率<br/>
基于以上定义，该调度策略可以总结为以下步骤：
```
初始化 
	结构e的优先队列
重复 
	1.随机选择一个具有最低子路径计数的节点继续
	2.如果下一个节点是终节点，求解路径条件获得测试用例
	3.用下一个节点来更新优先队列
直到 所有节点被覆盖 
```
用一个案例来进一步说明该算法（优先队列中带\*结构表示有包含相应子路径的未终结状态）：

| 步骤 | 候选路径 | 优先队列 | 生成用例 |
| --- | --- | --- | --- |
| 1 | 1:1t<br/>2:1f | (\*1t, 0), (\*1f, 0) | |
| 2 | 1:1t<br/>2:1f3t<br/>3:1f3f | (\*1t, 0), (\*3t, 0), (\*3f, 0), (1f, 1) | 1f3t4 |
| 3 | 1:1t<br/>2:1f3f | (\*1t, 0), (\*3f, 0), (3t, 1), (1f, 1) | |
| ... | ... | ... | ... |

### [Directed Automated Random Testing](http://pxzhang94.github.io/paper/concolic_testing/pldi2005.pdf)
该方法是动态符号执行测试调度策略，也是目前广泛使用的JAVA自动化测试工具JDart的算法基础。该调度策略可以总结为以下步骤：
```
初始化 
	一个具体执行的测试用例
重复 
	1.执行该测试用例
	2.深度优先搜索获取未覆盖的分支
	3.求解访问未覆盖分支的路径条件获得测试用例
直到 所有节点被覆盖 
```
用一个案例来进一步说明该算法：
- 假设第一个测试用例是x = 0, y = 100,其访问了节点1, 3, 5, 8
- 对路径<1, 3, 5, 6>通过求解下面的条件 x!=y && (x*x)%10!=9 && x*y+3y-5x==15执行符号执行 
- 假设测试用例是x=4, y = 5 ,其访问了节点1, 3, 5, 6, 8
- 对路径<1, 3, 5, 6, 7>执行符号执行
- ..... 

### [Coverage-Optimized Search](http://pxzhang94.github.io/paper/concolic_testing/klee-osdi-08.pdf)
该方法是动态符号执行测试调度策略,也是目前广泛使用的C/C++自动化测试工具KLEE的算法基础。该调度策略可以总结为以下步骤：
```
初始化
	一个具体执行的测试用例
重复
	1.执行该测试用例
	2.计算未覆盖节点与已覆盖节点间的最小距离
	3.从最小距离中随机选择一条路径
	4.求解该路径条件获得一个测试用例
直到 所有节点被覆盖
```
用一个案例来进一步说明该算法：
- 假设第一个测试用例是x = 0, y = 100,其访问了节点1, 3, 5, 8
- 对于 <1, 2>, <1, 3, 4>和<1, 3, 5, 6>最小距离是1, <1, 3, 5, 6, 7>是2
- 从<1, 2>, <1, 3, 4>和<1, 3, 5, 6>随机选择一条路径,假设是<1, 3, 4>
- 对路径<1, 3, 4>执行符号执行
- ..... 

### [Generational Search](http://pxzhang94.github.io/paper/concolic_testing/FuzzTesting.pdf)
该方法是动态符号执行测试调度策略。该调度策略可以总结为以下步骤：
```
初始化
	带有一个测试用例的测试套件
重复
	1.执行该测试用例
	2.给定该测试用例的路径，其上所有分支条件系统地用未被访问的另一分支替换
	3.求解每个约束获得相应的测试用例
直到 所有节点被覆盖
```
用一个案例来进一步说明该算法：
- 假设第一个测试用例是x = 0, y = 100,其访问了节点1, 3, 5, 8
- 对路径<1, 2>, <1,3,4>和<1,3,5,6>的路径条件求解并将相应的测试用例添加到测试套件中
- 选择路径<1, 2>的测试用例 
- 选择路径<1, 3, 4>的测试用例 
- 选择路径<1, 3, 5, 6>的测试用例 
- ..... 
下图来源于原论文，较为清晰的阐述了该算法调度顺序：
![Generational Search](http://pxzhang94.github.io/img/posts/concolic_testing/3.png)

### [Context-Guided Search](http://pxzhang94.github.io/paper/concolic_testing/seo-fse2014.pdf)
该方法是动态符号执行测试调度策略。首先给出一个关键的定义
- K-Context：定义分支b的k-context为一条执行路径上之前k个分支的序列
比如路径<1,2,3,5,8>包含分支(1,3,5)，那么：
- 5的1-context为(5)
- 5的2-context为(3,5)
- 5的3-context为(1,3,5)
该调度策略可以总结为以下步骤：
```
初始化
	一个具体执行的测试用例和k = 1
重复
	1.重复广度优先搜索
		a.检查在执行树上给定深度的每个节点的k-context是否是新的
		b.求解访问未覆盖分支的路径条件获得测试用例
		c.用测试用例更新执行树
	2.递增k
直到 所有节点被覆盖
```
用一个案例来进一步说明该算法：
- 假设第一个测试用例是x = 0, y = 100,其访问了节点1, 3, 5, 8
- 对路径<1, 2>执行符号测试
- 假设测试用例是x=0, y=0,其访问了节点1, 2, 3, 5, 8
- 对路径<1, 3, 4>执行符号测试
- ..... 




