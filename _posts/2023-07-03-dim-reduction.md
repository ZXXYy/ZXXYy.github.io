---
layout: post
title:      "线形/非线形降维方法概述"
date:       2023-07-03 09:00:00
author:     zxy
math: true
categories: ["Coding", "AI"]
tags: ["AI", "ML"]
post: true
---

## 绪论

### 什么是降维

降维（dimensionality reduction）是指通过寻找数据的低维表示，将高维数据转化为更紧凑的形式。降维的目标是减少数据的维度，同时尽可能保留原始数据中的重要信息。降维的有效性基于一个假设：高维数据中存在冗余信息或重复表达的信息。

在许多高维数据集中，存在大量的冗余信息或者可以被简化的重复表达。这意味着数据中的许多特征之间存在相关性或依赖关系，可以通过更低维的表示形式来捕捉这些关系，而不会丢失太多关键信息。通过降维，我们可以将数据映射到更低维的空间，从而消除或减少这些冗余信息，获得更紧凑的数据表示。

我们希望通过一个函数f把一个d维的向量映射到k维上且不丢失任何重要的信息（k<d），即

$$\boldsymbol{X}=\begin{bmatrix}x_1\\ x_2\\ \vdots\\ x_d\end{bmatrix}\rightarrow\boldsymbol{f}\left(\begin{bmatrix}x_1\\ x_2\\ \vdots\\ x_d\end{bmatrix}\right)=\begin{bmatrix}y_1\\ \vdots\\ y_k\end{bmatrix}=y\quad\text{with}\ k<d\quad (1)$$



### 为什么需要降维

在原始数据中各个特征之间存在着一定的信息冗余，随着特征的不断增加就容易出现“维数灾难”的问题。比如高维数据会导致计算上的挑战，很多传统的算法和方法在高维空间中需要更多的计算资源和时间，导致计算复杂度的急剧增加。降维可以帮助减少存储需求、提高计算效率，方便可视化和理解数据，去除冗余和噪声，防止过拟合，以及提高模型性能。

### 各种降维算法

一般情况下降维方法分为：线性降维方法和非线性降维方法。

线性降维方法的典型算法有：

- 主成份分析 (PCA, Principal Component Analysis) 
- 线性判别分析 (LDA, Linear Discriminant Analysis）
- 多尺度变换 (MDS, Multi-Dimensional Scaling)

非线性降维方法有：

- 保距特征映射 (ISOMAP) [2]
- 局部线性嵌入 (LLE, Locally Linear Embedding) [1]
- 拉普拉斯特征映射 (LE, Laplacian Eigenmap) [3]
- Maximum Variance Unfolding (MVU) [5]

在接下来的章节中，我们将逐一介绍这些算法并对比各种方法。

## 线性降维方法

线性降维方法的基本思想是通过线性变换将高维数据映射到低维空间。线性降维方法假设低维表示可以通过原始数据的线性组合来表示，从而在保留重要信息的同时减少数据的维度。

线形映射函数通常更容易找到，即$(1)$中的函数$f$是一个线性映射，在此处用$\mathbf{w}$表示，则线性降维方法可以表示为：

$$\begin{bmatrix}x_1\\ x_2\\ \vdots\\ x_d\end{bmatrix}\Rightarrow\mathbf{w}\begin{bmatrix}x_1\\ x_2\\ \vdots\\ x_d\end{bmatrix}=\begin{bmatrix}\mathbf{w}_{11}&\cdots&\mathbf{w}_{1d}\\ \vdots&&\vdots\\ \mathbf{w}_{k_1}&\cdots&\mathbf{w}_{k_d}\end{bmatrix}\begin{bmatrix}x_1\\ x_2\\ \vdots\\ x_d\end{bmatrix}=\begin{bmatrix}\mathbf{y}_1\\ \vdots\\ \mathbf{y}_k\end{bmatrix}\quad\text{with k<d}$$

### 主成份分析

主成份分析 (PCA, Principal Component Analysis) 是一种最常用的**无监督**线性降维方法。它通过找到原始数据中方差最大的方向（主成分）来实现降维。这是一个强假设，因为可能存在在方差较小的方向上有原始数据相关的重要信息，但是通常来说，方差小的方向会是噪声~\cite{kPCA}。

对于PCA主要步骤如下：

对一组数据$X=\{x_1,x_2,..., x_n\}$，其中$x_i$是一个$d$维的向量

1. 计算出样本均值 $\hat{\mu}=\frac{1}{n}\sum_{i=1}^{n}x_i$
2. 去中心化，$z_i=x_i-\hat{\mu}$
3. 计算协方差矩阵 $S=\sum_{i=1}^{n}z_i z_i^t$
4. 计算$S$的特征向量和特征值，得到最大的$k$个特征值对应的特征向量 $e_1, e_2, ...,e_k$
5. $y = [e_1 \cdots e_k]^Tz$，如果数据$x$已被中心化，则$\mathbf{w}=[e_1 \cdots e_k]^T$

### 线性判别分析

线性判别分析（Linear Discriminant Analysis，LDA）是一种经典的**有监督**线性降维方法，最开始是作为解决二分类问题由Fisher在1936年提出。LDA的主要目标是通过最大化不同类别之间的类间距离和最小化同类别内的类内距离，找到一个低维空间的投影，使得不同类别之间更易于区分。

对于LDA主要步骤如下：

对一组数据$X=\{(x_1,y_1),(x_2,y_2),..., (x_n,y_n)\}$，其中$x_i$是一个$d$维的向量，$y_i \in \{C_1, C_2\}$ ，$C_1,C_2$为常数

1. 计算类内散度矩阵$S_w=S_1+S_2=\sum_{y_i=C_1}(x_i-\mu_1)(x_i-\mu_1)^t+\sum_{y_i=C_2}(x_i-\mu_2)(x_i-\mu_2)^t$

   其中$\mu_j=\frac{1}{n_j}\sum_{y_i=C_j}x$

2. 计算类间散度矩阵$S_b=(\mu_{1}-\mu_{2})(\mu_{1}-\mu_{2})^{t}$

3. 计算矩阵$S_w^{-1}S_b$

4. 计算$S_w^{-1}S_b$的最大的$k$个特征值和对应的$k$个特征向量$e_1, e_2, ...,e_k$

5. 得到投影矩阵$\mathbf{w}=[e_1 \cdots e_k]^T$

### 多维尺度变换

最初的多尺度变换（Multi-Dimensional Scaling，MDS）是一种**无监督**线性降维方法，其目标是保持原始数据点之间的相对距离关系，尽可能地在低维空间中还原数据的结构。其中MDS又分为classical MDS和non-classical MDS：

- Classical MDS(经典多维尺度变换):经典多维尺度变换的距离标准采用欧式距离。

- Non-classical MDS(非经典多维度尺度变换):非经典多维度尺度变换的距离标准采用非欧式距离s

对一组数据$X=\{x_1,x_2,..., x_n\}$，定义一个距离矩阵$D=[d_{ij}:i,j=1,…,n]$，其中$d_{ij}$是两个样本点间的距离。多维尺度变换的优化目标是在低维欧氏空间$R^r$（通常是$R$或$R^2$）中找到一组点，使得它们之间的距离（或不相似度）尽可能接近于$d_{ij}$， MDS不针对$X$做映射，更关心的是距离矩阵$D$。

对MDS来说其优化目标函数为：

$$\sum_{i=1}^n\sum_{j=1}^n(d_{ij}-d(\mathbf{y}_i,\mathbf{y}_j))^2.$$

对一组数据$X=\{x_1,x_2,..., x_n\}$，其中$x_i$是一个$d$维的向量，经典多维尺度变换的主要步骤如下：

1. 计算样本的距离矩阵$D=[d_{ij}:i,j=1,…,n]$
2. 构造矩阵$A=[a_{i,j}]=[-\frac{1}{2}d^2_{i,j}]$

3. 构造中心化矩阵$B=HAH, H=I-\frac{1}{n}O$, 其中$I$为$n*n$的单位阵，$O$为$n*n$值均为1的矩阵
4. 对中心化矩阵 $B$ 进行特征值分解，得到最大的$k$个特征值对应的特征向量 $e_1, e_2, ...,e_k$
5. $y = [e_1 \cdots e_k]\Lambda_k^{\frac{1}{2}}$ ， 其中$\Lambda_k$为大的$k$个特征值构成的对角矩阵

## 非线形降维方法

与线性降维方法不同，非线性降维方法允许数据在降维过程中发生非线性变换，即$(1)$中的函数$f$非线性的，以更好地捕捉数据的内在结构和特征。常见的非线性降维方法有核主成分分析（Kernel Principal Component Analysis，Kernel PCA）、流形学习（Manifold Learning）、自编码器（Autoencoder）等。

在本小节中主要介绍核主成分分析以及流形学习中的Isomap，LLE，LE算法并对比他们之间的关系。

### kernel PCA

核主成分分析（Kernel Principal Component Analysis，Kernel PCA）[8]是一种非线性降维方法，是主成分分析（PCA）在高维特征空间中的扩展。它通过应用核函数将数据映射到高维特征空间，从而在非线性情况下实现降维。

对一组数据$X=\{x_1,x_2,..., x_n\}$，其中$x_i$是一个$d$维的向量，现在用一个非线性映射$\phi$ 将$X$ 中的向量$x$ 映射到高维空间(记为$d$维)，即$\phi\left(\mathbf{x}\right):R^d\to R^t,t\gg d$, 映射$\phi$通常不显示给出，最后对该高维度空间的向量进行PCA分析。kPCA的主要步骤如下：

1. 计算核矩阵$K=[k_{ij}: \phi(x_i)^T\phi(x_j)]$，对给定的数据集计算核矩阵，该矩阵描述了数据样本之间的相似度或内积。常用的核函数包括高斯核、多项式核、Sigmoid核等。

2. 构造中心化核矩阵$B=HKH, H=I-\frac{1}{n}O$, 其中$I$为$n*n$的单位阵，$O$为$n*n$值均为1的矩阵
3. 对中心化矩阵 $B$ 进行特征值分解，得到最大的$k$个特征值$\lambda$对应的特征向量$u=[e_1, e_2, ...,e_k]$
4. $Y=\frac{1}{\sqrt{\lambda}}Ku$

### kernel LDA

核线性判别分析（Kernel Linear Discriminant Analysis）[6] 是一种非线性降维和分类方法，是线性判别分析（Linear Discriminant Analysis，LDA）在高维特征空间中的扩展。和kernel PCA类似，它通过应用核函数将数据映射到高维特征空间，从而在非线性情况下实现判别分析。总的来说，核LDA就是对中心化的核矩阵执行LAD。

### Isomap

Isomap（Isometric Mapping）[2]是一种流形学习算法，基于MDS，用于非线性降维和数据可视化。它基于流形假设，认为高维数据分布在一个低维流形上，并试图在降维过程中保持样本之间的测地距离（geodesic distance）。总的来说，Isomap就是改变了MDS中距离的度量，利用最短路径算法（如Dijkstra算法）来估计数据点之间的测地距离。

Isomap的主要步骤如下：

1. 构建邻接图，有两种方法：一种是指定半径阈值，半径内的点为邻近点；一种是使用K近邻算法，在邻近点之间基于欧式距离构建一个邻接图。
2. 计算最短路径距离：利用最短路径算法（如Dijkstra算法）计算邻接图中任意两个节点之间的最短路径距离。这些距离被认为是数据点之间的测地距离的近似。根据计算得到的最短路径距离，构建一个距离矩阵$D$，其中每个元素表示两个数据点之间的测地距离。
3. 构建距离矩阵，根据MDS线性降维算法对测地距离进行降维

### 局部线性嵌入

局部线性嵌入 (LLE, Locally Linear Embedding)[1]算法由 Sam T.Roweis等人于2000年提出并发表在《Science》杂志上。该算法基于流形学习的思想，假设高维数据分布在一个低维流形上，并试图在降维过程中保持数据点之间的局部线性关系。LLE通过局部线性拟合来恢复全局的非线性结构，希望每个点与其邻近点的相对关系得以保持，即原来距离近的样本在新空间继续保持近的距离，而原来非常远的样本现在在哪并不关注。

LLE的主要步骤如下：

对一组数据$X=\{x_1,x_2,..., x_n\}$，其中$x_i$是一个$d$维的向量，需要将$x_i$映射到$k$维的$y_i$向量上

1. 寻找$x_i$的邻接点，可使用K近邻算法

2. 重构权重计算：对于每个数据点，通过最小化其与邻域点之间的重构误差，计算其与邻域点的重构权重。重构误差可通过最小二乘法求解。

   即最小化
   $\varepsilon(W)=\sum\limits_{\mathrm{i}}\left|x_i-\sum_\mathrm{j}W_{\mathrm{ij}}x_j\right|^2$

   如果$x_j$为非近邻点，则$W_{ij}=0$，得到权重矩阵$W$

3. 构建了保持邻域关系的映射，高维度的$x_i$被映射到低维度的$y_i$，表示流形上的全局内部坐标，$y_i$的计算通过最小化嵌入代价函数来实现，即最小化
$\Phi(Y)=\sum_i\bigg|y_i-\sum_j W_{ij}y_j\bigg|^2$

​		经过一系列数学推导，求解$y_i$即为求解矩阵$M$的最小$k$个非0特征值对应的特征向量$y_i$,

​		其中$M=(I-W)(I-W)^T$, $I$为单位矩阵

### 拉普拉斯特征映射 

拉普拉斯特征映射 (LE, Laplacian Eigenmap) [3]基于图论和谱图理论的思想，通过构建数据的拉普拉斯矩阵和特征值分解，实现对数据的降维和特征提取。LE的主要思想是利用数据的局部邻域结构来构建一个图，并通过图的拉普拉斯矩阵来描述数据点之间的关系。LE和LLE非常相似，目标都是降维后尽量保持数据的局部分布而忽略全局分布。

LE的主要步骤如下：

1. 构建邻接图，有两种方法：一种是指定半径阈值，半径内的点为邻近点；一种是使用K近邻算法，在邻近点之间基于欧式距离构建一个邻接图。

2. 重构权重计算：如果点$x_i$和点 $x_j$相连，那么它们关系的权重可设定为热核函数$W_{ij}=e^{-\frac{\|x_i-x_j\|^2}{t}}$或简化为$W_{ij}=1$，否则$W_{ij}=0$

3. 进行特征映射，计算拉普拉斯矩阵$L=D-W$ 的特征向量与特征值，其中$D$是对角阵，$\begin{aligned}
   D_{ii}& =\sum_j W_{ji}  
   \end{aligned}$

   即$Ly=\lambda Dy$，使用最小的 $k$个非零特征值对应的特征向量作为降维后的结果输出

### **最大方差展开**

最大方差展开（MVU, Maximum Variance Unfolding）[5]也叫做semidefinite embedding，基于数据的局部邻域结构和方差最大化的思想，通过优化一个目标函数来实现降维和数据展开。

对一组数据$X=\{x_1,x_2,..., x_n\}$，其中$x_i$是一个$d$维的向量，需要将$x_i$映射到$k$维的$y_i$向量上$k\ll d$ 

 MVU的优化目标是，给定一组边的集合$E$，使得$\|\mathbf{y}_i-\mathbf{y}_j\|\approx\|\mathbf{x}_i-\mathbf{x}_j\|\ \text{for}(i,j)\in E$

MVU的主要步骤如下：

1. 使用K近邻算法构建邻接图，在邻近点之间基于欧式距离构建一个邻接图。
2. 使用半正定“展开”邻接图，与直接学习输出向量不同，半定法旨在找到一个内积矩阵，最大化邻域图中未连接的任意两个输入之间的距离，同时保持最近邻之间的距离。
3. 对内积矩阵应用MDS算法，获得降维后的向量。

## 总结

我们通过一张表格总结上述提到的各种降维算法，并且比较他们之间的异同。

| 算法名称 | 线性/非线性 | 有监督/无监督 | 优化目标                                                     | 优点                                                         | 缺点                                                         |
| -------- | ----------- | ------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| PCA      | 线性        | 无监督        | 降维后的低维样本之间每一维度方差尽可能大                     | 能够找到数据中最重要的特征或主成分，具有较高信息量           | 计算复杂度高，低方差的特征可能会被忽略，导致部分信息损失。   |
| LDA      | 线性        | 有监督        | 降维后同一类样本间协方差尽可能小，不同类中心距离尽可能大     | 可以进行降维又可以进行分类，对于数据噪声和异常值具有一定的鲁棒性 | 计算复杂度高，需要有标签的数据，LDA假设数据符合多元正态分布  |
| 经典MDS  | 线性        | 无监督        | 降维的同时保持数据点之间的相对距离关系                       | 通过改变距离函数MDS可以处理非线性关系的数据                  | 计算复杂度高，受限于距离度量，对噪声敏感                     |
| kPCA     | 非线性      | 无监督        | 核函数将数据映射到高维空间，再对高维空间降维，降维后的低维样本之间每一维度方差尽可能大 | PCA的非线性拓展                                              | 计算复杂度高，需要调整超参数                                 |
| kLDA     | 非线性      | 有监督        | 核函数将数据映射到高维空间，再对高维空间降维，降维后同一类样本间协方差尽可能小，不同类中心距离尽可能大 | LDA的非线性拓展                                              | 计算复杂度高，需要调整超参数                                 |
| Isomap   | 非线性      | 无监督        | 降维的同时保证高维数据的流型不变，即降维过程中保持样本之间的测地距离 | 保持流形的全局几何结构，适用于学习内部平坦的低维流形         | 计算复杂度较高，不适于学习有较大内在曲率的流形               |
| LLE      | 非线性      | 无监督        | 降维后尽量保持数据的局部分布                                 | 可以学习任意维的局部线性的低维流形，计算复杂度较低           | 所学习的流形只能是不闭合的；要求样本在流形上是稠密采样的     |
| LE       | 非线性      | 无监督        | 降维后尽量保持数据的局部分布                                 | 效率高，使原空间中离得很近的点在低维空间也离得很近，可以用于聚类 | 对算法参数和数据采样密度较敏感，不能有效保持流形的全局几何结构 |
| MVU      | 非线性      | 无监督        | 降维后尽量保持数据的局部分布，且最大化数据点之间的方差       | 尽可能保持数据的局部和全局结构，有噪声的数据时具有一定的鲁棒性 | 性能受超参数影响大                                           |

总的来说，kPCA、kLDA、 Isomap分别是PCA、LDA、经典MDS在非线性空间上的拓展。kPCA和kLDA采用核函数的方式将数据映射到高维空间，进而把问题转化为线性问题。而lsomap采用测地距离的方式来保证高维数据的流型不变。

对于非线形降维，Isomap、LE、LLE均利用了局部近邻的信息来构建针对流型的全局嵌入。但相对Isomap关注整个流型的全局特征，LLE和LE算法仅仅关注样本的局部分布。从计算的角度来说，Isomap和MVU在降维时会一个密集矩阵，而LE和LLE算法构建了一个稀疏矩阵。

为了更好的理解这些非线形降维算法之间的关系，Ham等人[4]指出，Isomap、LE、LLE这三种算法均可以转化为kPCA。Xiao [7]等人用统一的对偶形式表示了Isomap、LE、LLE以及MVU这四种算法，并说明了这些算法间的联系。

## 参考文献

[1] Roweis, Sam T., and Lawrence K. Saul. "Nonlinear dimensionality reduction by locally linear embedding." *science* 290.5500 (2000): 2323-2326.

[2] Tenenbaum, Joshua B., Vin de Silva, and John C. Langford. "A global geometric framework for nonlinear dimensionality reduction." *science* 290.5500 (2000): 2319-2323.

[3] Belkin, Mikhail, and Partha Niyogi. "Laplacian eigenmaps for dimensionality reduction and data representation." *Neural computation* 15.6 (2003): 1373-1396.

[4] Ham, Jihun, et al. "A kernel view of the dimensionality reduction of manifolds." *Proceedings of the twenty-first international conference on Machine learning*. 2004.

[5] Weinberger, Kilian Q., and Lawrence K. Saul. "An introduction to nonlinear dimensionality reduction by maximum variance unfolding." *AAAI*. Vol. 6. 2006.

[6] Mika, Sebastian, et al. "Fisher discriminant analysis with kernels." *Neural networks for signal processing IX: Proceedings of the 1999 IEEE signal processing society workshop (cat. no. 98th8468)*. Ieee, 1999.

[7] Xiao, Lin, Jun Sun, and Stephen Boyd. "A duality view of spectral methods for dimensionality reduction." *Proceedings of the 23rd international conference on Machine learning*. 2006.

[8] Schölkopf, Bernhard, Alexander Smola, and Klaus-Robert Müller. "Kernel principal component analysis." *International conference on artificial neural networks*. Berlin, Heidelberg: Springer Berlin Heidelberg, 1997.

### 参考博客

https://www.cnblogs.com/pinard/p/6239403.html

https://www.showmeai.tech/article-detail/198

https://chenrudan.github.io/blog/2016/04/01/dimensionalityreduction.html#1

http://blog.codinglabs.org/articles/pca-tutorial.html

https://leovan.me/cn/2018/03/manifold-learning/#fn:1

https://www.cnblogs.com/pinard/p/6244265.html

https://rich-d-wilkinson.github.io/MATH3030/6.1-classical-mds.html#non-euclidean-distance-matrices

https://0809zheng.github.io/2021/07/27/kpca.html

https://zhuanlan.zhihu.com/p/59775730

https://zhuanlan.zhihu.com/p/434098745

https://tianchi.aliyun.com/forum/post/79400

https://www.cs.cmu.edu/~ggordon/10725-F12/scribes/10725_Lecture21.pdf

https://en.wikipedia.org/wiki/Semidefinite_embedding