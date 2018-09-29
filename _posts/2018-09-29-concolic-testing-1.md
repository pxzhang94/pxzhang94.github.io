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
- 对于高可能性的错误，随机测试的代价很低，反之很高（比如说👇的第3行判断语句，对于随机测试来说，几乎不可能覆盖到其True分支）
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
- 符号执行的代价很高，但是可以“保证”到达给定节点
同样是上面那个程序，我们可以通过求解👇的路径约束：
```
y == 42342531 && (x<0 || x >= 10)
```
生成一个特定的测试输入来覆盖我们指定的路径，比如：
```
x == 10 and y == 42342531
```
但是目前符号执行面临两个问题，第一：由于需要利用符号变量来构建路径约束，所以显然其比随机测试需要更多的时间；第二：目前最常见的约束求解器（如Z3等）的求解能力仍十分有限，对于稍复杂的路径约束就束手无策。
所以自动化测试领域提出了concolic testing，我翻译为动态符号执行测试，也可以翻译为具体符号执行，因为concolic是concrete和symbolic的合成词。

## 动态符号执行测试（Concolic Testing）
- 集成了具体执行和符号执行两种策略
- 决定选择随机测试还是符号执行
- 如果选择符号执行，决定选择的程序执行路径
- 常见的动态符号执行测试工具：Microsoft SAGE and JDart
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
![抽象语法树](http://pxzhang94.github.io/img/posts/concolic_testing_1.png)
如👆图所示，黑色箭头表示随机测试，蓝色箭头表示符号执行

## 常见动态符号执行策略
事实上，绝大多数动态符号执行测试策略仅执行一遍随机测试，更多的是关注随后符号执行路径的调度，这导致相关算法性能距离[最优调度策略](http://pxzhang94.github.io/2018/10/01/concolic_testing_2)有极大的差距。
### Directed Automated Random Testing
### Coverage-Optimized Search
### Generational Search
### Context-Guided Search




