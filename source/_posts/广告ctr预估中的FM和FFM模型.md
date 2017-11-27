---
title: 广告ctr预估中的FM和FFM模型
date: 2017-11-26 18:04:41
tags: 机器学习、互联网广告
mathjax: true
---

##1. 在线广告广告ctr预估

> 点击率(click-through rate)和转化率(conversion rate)是衡量广告流量的两个重要指标。点击率与点击价值的乘积决定了广告的排列顺序。在大规模广告系统中，常用的ctr预估模型是LR(Logistic Regression)，LR易于并行化，配合大量的人工特征工程，可以产生很好的效果。但LR本质上是一种广义线性模型，分类能力有限。因此，近年来出现了以GBDT(Gradient Boosting Decision Tree)、FM(actorization Machine)及其改进FFM(Field-aware Factorization Machine)为代表的非线性模型，在一些CTR预估竞赛中表现出优异的效果。本文将对FM和FFM进行简短概要的介绍。

##2. FM模型
> **FM(Factorization Machine)由Steffen Randle于2010年提出，起初是用于推荐系统中稀疏数据场景下的评分预测问题。**它属于一种在多项式模型上衍生出来的模型。

- 二阶FM的预测公式：
$$
\phi_{FM}(w, x) = w_{0}+\sum\limits_{i=1}^{n}w_{i}x_{i}+\sum\limits_{i=1}^{n-1}\sum\limits_{j=i+1}^{n}w_{ij}x_{i}x_{j}
 = w_{0}+\sum\limits_{i=1}^{n}w_{i}x_{i}+\sum\limits_{i=1}^{n-1}\sum\limits_{j=i+1}^{n}(v_{i}^{T}v_{j})x_{i}x_{j}
$$
注意这里直接计算$\phi_{FM}(w, x)$的时间复杂度是$O(kn^{2})$。经过变换二次项，时间复杂度可降为$O(kn)$。即：
$$\sum\limits_{i=1}^{n}\sum\limits_{j=i+1}^{n}<v_{i}, v_{j}>x_{i}x_{j}=\frac{1}{2}\sum\limits_{f=1}^{k}((\sum\limits_{i=1}^{n}v_{i,f}x_{i})^{2}-\sum\limits_{i=1}^{n}v_{i,f}^{2}x_{i}^{2})$$
FM通过将特征变量交叉项的系数分解到隐变量空间中，可以更好的在稀疏数据中估计二次项的参数。FM模型以其通用的表达形式可以直接适用于回归、分类和排序任务。
- 模型学习
针对FM模型，可以直接使用SGD进行模型参数更新，公式如下：
\begin{equation}
\frac{\partial \phi_{FM}(w, x)}{\partial w}=
\left\{
\begin{aligned}
1,  \theta = w_{0} \\
x_{i}, \theta = w_{i} \\
x_{i}\sum\limits_{j=1}^{n}v_{j,f}x_{j}-v_{i,f}x_{i}^{2}, \theta = v_{i,f}
\end{aligned}
\right.
\end{equation}

##3. FM模型的改进: FFM模型
> **FFM(Field-aware Factorization Machine)由Yuchin Juan等在2014年参加kaggle criteo展示广告点击率预测比赛时提出。在FM模型基础之上引入了特征域的概念。**

- 预测公式：
$$
\phi_{FFM}(w, x) = w_{0}+\sum\limits_{i=1}^{n}w_{i}x_{i}+\sum\limits_{i=1}^{n-1}\sum\limits_{j=i+1}^{n}(v_{i,f_{1}} \cdot v_{j,f_{2}})x_{i}x_{j}
$$
这里，$f_{1}, f_{2}$是特征$x_{i}$和$x_{j}$所属的域，每一个特征所属的域是唯一的，$x_{i}$和$x_{j}$被分解到了对应域的隐空间中。二次项的总共有$\frac{n(n-1)}{2}$个。
- 模型学习：
 1. 计算子梯度
$g_{i, f_{1}} = \triangledown_{w_{i,f_{1}}}f(w)=\lambda w_{i,f_{1}} + \kappa w_{j, f_{2}}$
$g_{j, f_{2}} = \triangledown_{w_{j, f_{2}}}f(w) = \lambda w_{j,f_{2}} + \kappa w_{i, f_{1}}$
这里，$\kappa = \frac{-y}{1+exp(y\phi_{FFM}(w,x))}$
 2. 计算总梯度
$(G_{i, f_{1}})_{d} = (G_{i, f_{1}})_{d} + (g_{i, f_{1}})_{d}^{2}$
$(G_{j, f_{2}})_{d} = (G_{j, f_{2}})_{d} + (g_{j, f_{2}})_{d}^{2}$
 3. 参数更新
$(w_{i, f_{1}})_{d} = (w_{i, f_{1}})_{d} - \frac{\eta}{\sqrt{(G_{i, f_{1}})_{d}}}(g_{i,f_{1}})_{d}$
$(w_{j, f_{2}})_{d} = (w_{j, f_{2}})_{d} - \frac{\eta}{\sqrt{(G_{j, f_{2}})_{d}}}(g_{j,f_{2}})_{d}$

    这里，参数更新时，参考了Adagrad算法的自适应学习率方法。FFM模型整体的训练时间复杂度是$O(kn^{2})$，相比FM更高。FFM在实际使用时容易过拟合，需要使用Early Stopping策略。并且需要将原始数据转换成"field_id:feature_id:value"的格式。实际使用时，数值型特征分配单独的field编号，类别特征ohe后的所有特征属于同一个域。
- FFM的适用场景
  - 数据集包含大量类别特征，并进行了ohe编码
  - 转换后的数据集比较稀疏

##4. 其他杂项
针对展示广告中的点击率或转化率预估模型，通常需要求解如下无约束优化问题：
$$
min_{w} \frac{\lambda}{2}||w||_{2}^{2}+\sum\limits_{i=1}^{m}log(1+exp(-y_{i}\phi(w, x_{i})))
$$

- 这里的特征一般含用户侧(性别、年龄、教育背景、收入、偏好等)、广告侧(创意、素材、标题等)、上下文(时间、位置等)等特征，实际使用时需要针对模型的特性做不同的特征工程。
- 优化算法一般有一阶优化方法SGD(Stochastic Gradient Descent)、二阶拟牛顿优化方法L-BFGS(Limited memory Broyden–Fletcher–Goldfarb–Shanno)、Trust-Region。这里就暂时不展开讲述，留在后续文章中介绍。

##5. 参考资料
[1] [深入FFM原理与实践-美团点评技术博客](https://tech.meituan.com/deep-understanding-of-ffm-principles-and-practices.html)
[2] [Factorization Machines-steffen randle](http://www.csie.ntu.edu.tw/~b97053/paper/Rendle2010FM.pdf)
[3] [Field-aware Factorization Machines for CTR Prediction](https://www.andrew.cmu.edu/user/yongzhua/conferences/ffm.pdf)
[4] [kaggle: Display Advertising Challenge](https://www.kaggle.com/c/criteo-display-ad-challenge)
