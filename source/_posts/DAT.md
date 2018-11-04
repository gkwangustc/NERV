---
title: 论文笔记 - "Deep Attentive Tracking via Reciprocative Learning
date: 2018-11-04 21:50:04
tags: paper, visual tracking
mathjax: true
---

### 前言

这是Ma Chao大神发表在NIPS 2018上的一篇论文，从代码来看是MDNet的改进版，与CVPR 2018和ECCV 2018的趋势一样，这篇论文也在跟踪框架中使用了Attention，不过与其他论文不同的是，本篇论文的Attention并不是通过在网络加入一个module来学习得到的，而是将BP得到的梯度作为Attention，并将其作为一个Loss函数的正则项来调整网络的训练过程，比较有新意。

论文：https://arxiv.org/pdf/1810.03851.pdf

源码：https://github.com/shipubupt/NIPS2018

<!--more-->

### 背景

一般来说，tracking和detection类似，可以分为one-stage和two-stage两种。代表性的one-stage方法一般基于DCF方法，通过相关滤波器在特征上卷积得到heat map，最高点即为目标所在位置。two-stage方法一般为判别方法，即先在图像中产生proposal，然后通过分类器来给proposal打分，最高的proposal即为目标所在位置，如SVM，CNN-SVM等。

这两种方法都可以通过Attention提高tracker的性能，在one-stage方法中，在输入特征上加Gaussian或者Cosine窗函数来抑制目标周围的特征，让DCF专注于图片中心的目标，可以视为一种简单的Attention，但是当物体有较大位移时，这样的Attention就没有太大用处，而且抑制边缘的背景特征，也会丢掉一些有用的信息；在two-stage方法中，Attention更常用于特征选择，在end-to-end的训练框架中，通过Attention模块的选择作用，可以增强特征的表达能力，提高tracker性能，但是在单帧上学习得到的attention在持续的跟踪过程中并不鲁棒，由于无法自适应的调整，一些微小的错误就会导致累积效应，从而限制了tracker的性能。

### 创新点

本文提出了一种交互学习算法来探索attention在two-stage的跟踪算法中的应用。与现有的two-stage算法不同，本文的attention并不是通过一个额外的attention module计算得到的，作者直接从分类网络中得到attention信息。

首先作者将图像输入网络进行前传，得到classification score，然后进行反传，得到每一层的梯度，但是在反传的过程中，网络的参数并没有更新，仅仅是为了计算得到第一个卷积层的梯度，并将其作为attention map，加入到loss函数中。然后使用这个loss函数来训练网络。跟踪时，直接使用前传得到的分数来判断每个proposal是否为目标。

本文的创新点可以概括为

- 提出了一个交互学习的算法，来计算tracking-by-detection框架中的attention
- 将attention map作为Loss函数中的正则项来训练分类器，让其聚焦于时序鲁邦的特征
- 作者在各大benchmark上进行测试，性能和state-of-the-art的方法性能相当。

### 本文提出的方法

本文的框架图如图所示：

![Pipeline](/images/DAT/Pipeline.png)

#### Attention

对于一个基于CNN的track-by-detection框架，我们假设输入为$I$，输出为score向量，每个元素代表I输入属于$c$类的概率。对于一个给定的输入图片$I_0$，使用一阶的泰勒在点$z_0$处对映射函数$f_c(I)$进行泰勒展开，可以得到：

$$
f_c(I) \approx A_{c}^{\top} I + B
$$

点$z_0$是$I_0$的去心邻域中的一点，对于$I_0$去心邻域中的任何一点，上述等式都成立。而由于$I_0$和$z_0$足够近，因此两点的梯度相同，在上式中$A_c$为偏导数，即函数在$I_0$处的梯度

$$
A_c = \frac{\partial f_c(I)}{\partial I}\mid_{I=I_0}
$$

此外，从等式1中可以看出，映射函数求出的score收到$A_c$的影响，所以$A_c$也反映了$I_0$对score的重要性有多大，因此$A_c$可以被视为一种attention map

求解attention map时，需要按照等式2来得到输入$I_0$的梯度，所以过程分两步进行，首先将$I_0$输入网络中，并进行反传，取第一层的梯度，并且只保留正值来作为attention map，因为正值可以清晰的反应$I_0$中每一部分对分数的贡献。在这个BP过程中，网络参数是不更新的

#### Attention的正则化

在tracking-by-detection框架中，一般通过分类器将目标标记为正类，将背景标记为负类，如果将目标样本求得的attention map即为$A_p$，将背景样本求得的attention map记为$A_n$的话，我们希望$A_p$越大越好，$A_n$越小越好。因此，我们定义了如下的正则项，对于正样本：

$$
R_{(y=1)} = \frac {\sigma_{A_p}} {\mu_{A_p}} + \frac {\mu_{A_n}} {\sigma_{A_n}}
$$

对于负样本：

$$
R_{(y=0)} = \frac {\sigma_{A_n}} {\mu_{A_n}} + \frac {\mu_{A_p}} {\sigma_{A_p}}
$$

相应的分类Loss函数调整为：

$$
\mathcal{L} = \mathcal{L}_{CE} + \lambda \cdot \left [ y \cdot R_{(y=1)} + (1 - y) \cdot R_{(y=0)} \right ]
$$

上式用来说明attention map是如何影响网络的训练的。对于正样本，我们从两个方面来加强目标的attention，首先就是增加$A_p$的均值，并减小方差，这样使得$A_p$的密度增大，同时变化减小（类似高原），其次就是减小$A_n$的均值，并增加方差，这样使得$A_n$的密度减小，变化增大（类似丘陵），分类器通过这两个约束可以提高true-positive，降低false-negative。同样的对于等式4，分类器可以提高true-negative，降低false-postive。通过attention map的约束，分类器能够提高准确率

#### 交互学习

![Attention map的对比](/images/DAT/reciprocative_learning.png)

将attention map加入Loss函数之后，我们就可以进行交互学习了。在每次迭代过程中，我们都会计算出输入的sample的attention map，它反映了分类器当前的注意力在哪儿。没有attention约束的分类器，往往只会聚焦于某些具有判别能力的区域，当物体产生很多大形变时，这些点并不能让我们有效的跟踪。而有了attention的约束，判别器会聚焦于所有能将目标和背景区分开的区域。如图所示，随着交互学习的进行，attention区域逐渐扩展到整个目标。

### 跟踪过程

#### 模型初始化

在第一帧时，我们在目标周围随机采集$N_1$个样本，根据与目标的IoU是否大于0.5划分为正负样本，按照之前的公式，训练$H_1$轮，来更新全卷积层的权重

#### 在线检测

在上一帧预测的目标周围采集$N_2$个样本输入网络，选择得分最高的样本进行bounding box regression，作为在当前帧的检测结果

#### 模型更新

 在每一帧，我们在预测的位置附近采集$N_2$个样本，同样根据与预测位置的IoU是否大于0.5划分为正负样本。然后，每隔$T$帧，按照公式训练$H_2$轮，来更新卷积层的权重

 ![可视化](/images/DAT/contrast.png)

作者将跟踪过程进行了可视化，如图所示，左右两栏分别代表baseline(没有交互学习)，以及本文方法（交互学习），第一栏为attention map，第二栏为score map，第三栏为跟踪结果，红框为预测框，绿框为groundtruth，可以看出，在一开始，两种方法相差不大，但是随着跟踪过程的进行，交互学习可以帮助分类器聚焦到整个目标区域，增强分类器的判断能力。最终，当与目标类似的干扰物体出现时，没有交互学习的方法就产生了漂移，而本文的方法则正确的检测到了结果。


### 实验

#### 实验细节

特征提取网络为VGG-M，在整个跟踪过程中参数固定。全连接层是随机初始化并在跟踪过程中逐步更新的。作者使用GTX 1080时的速度为1 FPS。

#### 评测指标

作者在OTB上使用的指标为distance precision (DP) 和 overlap success (OS)，DP使用的是20 pixel对应的准确度，OS使用的曲线下面积，此外还计算了CLE（中心平均误差）和 IoU为0.5时的比例（$OS_{0.5}$）。在VOT上，作者使用的指标为EAO，Ar，Rr

#### Ablation Studies

![表1](/images/DAT/table1.png)

作者首先在OTB2013上做了关了$\lambda$的实验，根据表1可以看出，$\lambda$在3到5之间的表现都很好，最终作者选择了5

![表2](/images/DAT/table2.png)

由于attention map作为特征权重也可以提高分类器的表现，为了证明attention map约束的分类器更有效果，作者进行了对比实验，最终发现，attention map作为权重虽然可以提高分类器表现，但是提升幅度不如attention map作为约束

#### 与其他方法的对比

作者在OTB-2013，OTB-2015，VOT-2016上与目前state-of-the-art的一些算法进行了对比，结果自然是比其他算法要高。

![OTB结果1](/images/DAT/table3.png)

![OTB结果2](/images/DAT/OTB.png)

![VOT结果](/images/DAT/VOT.png)

#### 总结

与现有的一些使用attention map作为feature weight的做法不同，本文从另一个角度使用attention map，将其作为一个约束项加入Loss函数中，从而让分类器在跟踪的过程中学会如何去attention，可以说是比较有新意的视角，本文的源码也已经放在了github上，有兴趣的同学可以自己尝试用一下。
