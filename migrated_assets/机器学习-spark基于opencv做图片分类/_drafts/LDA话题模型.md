---
title: LDA话题模型
tags:
---


在阅读完 Rickjin(靳志辉)老师的<LDA数学八卦>，对这一算法有了更深刻的理解, 并结合其它网络资料，组织成了这篇文章 可以让初学者入门 也便于已近掌握次模型的人得以回顾


## 那些年你错过的函数


### Gamma函数

$\Gamma(x)=\int_0^\infty t^{x-1}e^{-t}dt$
$\Gamma(n) = (n-1)!$



Gamma函数的积分形式告诉我们一个实数的阶层是多少

对于这个函数一般人可能会有两个疑问
1. 这个函数科学家是如何找到
2. 这个函数为什么定义为$\Gamma(n) = (n-1)!$

Gamma分布的更一般表达式
$Gamma(t|\alpha, \beta) = \frac{\beta^\alpha t^{\alpha-1}e^{-\beta t}}{\Gamma(\alpha)}$
其中 $\alpha$ 称为shape parameter, 主要决定了分布曲线的形状，$\beta$ 称为rate parameter, 或者 inverse scale parameter 主要决定了曲线有多陡


1. Gamma函数可以拓展到复平面上
2. Gamma函数使得1/2次导数的计算变得有意义, 所以对于一般的函数f(x)通过Taylor级数表达为幂级数
可以尝试定义出任意函数的分数阶导数 由此产生分析数学的一个研究课题 Fractional Calculus


Gamma分布可以作为先验分布
Gamma分布和泊松过程联系紧密
二项式分布的极限是泊松分布

iid条件 independent and identically distributed IID 条件 一组随机变量中每个变量的概率分布都相同 且这些随机变量相互独立

