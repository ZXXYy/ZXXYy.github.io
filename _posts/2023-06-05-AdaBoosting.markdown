---
layout: post
title:      "AdaBoosting Learning notes"
date:       2023-06-05 14:00:00
author:     zxy
math: true
categories: ["Coding", "AI"]
tags: ["AI"]
post: true
---

## High-Level ideas

1. "Wisdom of the Crowds"
- commonly use decision stump as crowd
2. "Wisdom of the Weighted Crowds"
3. sequential building of stumps

## My understanding of AdaBoost

**Motivation**：Adaboost算法后的思想其实就是“三个臭皮匠顶个诸葛亮”，用英语来说就是"Wisdom of the Crowds": 单独使用时表现不佳的模型在组合起来之后就可以形成强大的模型。

**Terms：**

- Decision stump: 最简单的Decision Tree，由一个decision node和两个叶子结点组成。

- Weak learner/Week classifier：也就是臭皮匠，对二分类问题来说，这些分类器的效果可能就比抛硬币要好一些。在AdaBoost中通常会使用Decision stump来作为weak classifier

**Three general ideas of AdaBoost**:

1. "Wisdom of the Crowds": Use decision stumps as crowds to vote.
2. "Wisdom of the weighted Crowds"：some stumps get more say in the final classificatin than others.
3. sequential building of stumps: Each stump is made by taking the previous stump's mistakes into account.

**Boosting vs. Bagging**:

- 在Bagging中, 各个模型是独立的，彼此不同，使用训练数据集的不同随机子集进行训练的。Random forest就是基于这个原则。
- 在Boosting中，各个模型的模型构建过程一个接一个地进行，模型预测的准确性影响后续模型的训练过程。

## Algorithmic description of the AdaBoost

The training set is denoted as $(x_1,y_1),\dots,(x_m,y_m)$ 

The weight of this distribution on training example $i$ on round $t$ is denoted ,$D_t(i)$

**Given**: $(x_1,y_1),\dots,(x_m,y_m)$ where $x_i\in X,y_i\in Y=\{-1,+1\}$

Initialize $D_1(i)= 1/m$.

For $t = 1, ..., T$:

- Train weak learner using distribution $D_t$

- Get weak hypothes $h_{t}:X\rightarrow\{-1,+1\}$ with error

  $$\epsilon_t=\Pr_{i\sim D_t}\left[h_t(x_i)\neq y_i\right].$$

- Choose $\alpha_{t}=\frac{1}{2}\ln\Big(\frac{1-\epsilon_{t}}{\epsilon_{t}}\Big).$

- Update:

  $D_{t+1}(i)=\frac{D_t(i)}{Z_t}\times\left\{\begin{array}{ll}e^{-\alpha_t}&\textrm{if}h_t(x_i)=y_i\\ e^{\alpha_t}&\textrm{if}h_t(x_i)\neq y_i\end{array}\right. \\=\frac{D_{t}(i)\exp(-\alpha_{t}y_{i}h_{t}(x_{i}))}{Z_{t}}$

​		where $Z_{t}$  a normalization factor (chosen so that $D_t+1$ will be a distribution).

**Output** the final hypothesis:

$H(x)=\operatorname{sign}\left(\sum\limits_{t=1}^T\alpha_t h_t(x)\right).$


## Reference
1. https://www.cs.cmu.edu/~aarti/Class/10701/slides/Lecture10.pdf