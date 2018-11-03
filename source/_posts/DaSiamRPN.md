---
title: 论文笔记 - Distractor-aware Siamese Networks for Visual Object Tracking
date: 2018-09-23 21:05:04
tags: paper, visual tracking
mathjax: true
---
### 概要

DaSiamRPN是SiamRPN的后续作品，使用的backbone为SiamRPN，本篇论文主要是在数据集扩展、训练方法、loss函数以及local-to-global方面对SiamRPN进行了改进。

<!--more-->

### 创新点

#### Distractor-Aware
- 传统Siamese的缺点
- 创新点一：训练方法
- 创新点二：Loss函数

#### Long-Term
- 传统Siamese的缺点
- 创新点三：local-to-global

### Distractor-Aware

#### 传统siamese的缺点

- 现象：除了目标得分较高，其他类似的物体得分也很高

  ![Heatmap](/images/DaSiamRPN/Heatmap.png)

- 原因一：无语义信息的目标数量要远远大于有语义信息的目标和数量

  在训练过程中，训练的图片对中，大部分区域都是没有语义信息的背景，有语义信息的很少，因此，网络只学习了区分背景和前景的能力。

- 原因而：有意义的目标中，大部分为干扰目标，而不是要跟踪的目标

  在测试过程中，Siamese只使用了第一帧的部分图片，忽略了背景信息，此外Siamese只是将搜索区域附近得分最高的物体标记为目标，但是有可能周围的物体只是跟目标很像，并不是物体

#### 创新点一：训练方法

![](/images/DaSiamRPN/Imagepair.png)

- 通过多种类的正图片对来增加模型的生成能力

  作者扩展了训练用的数据集，除了使用VID以及YouTube-BB之外（物体种类较少，分别只包含20和30个类），还通过数据增强的方式，使用ImageNet DET和COCO作为训练集，极大的增加了物体的种类

- 通过包含语义信息的负图片对来增加模型的判别能力

  作者在训练的过程中，有意的使用相同种类但不是目标的负图片对来训练网络，使得网络可以对同种类的不同物体进行有效的区分，增加了鲁棒性

#### 创新点二：更改了Loss函数

- 在函数中增加了Distractor项

  作者首先在每帧的检测结果中，使用NMS筛选出可能的Distractor，方法如下：

  $$
  D = \left [ \forall d_{i} \in D , f(z, d_{i}) > h \cap d_{i} \neq z_{t} \right ]
  $$

  其中，$z$为当前帧的目标，$h$为给定阈值，$d_i$为可能的Distractors。

  然后将Loss函数更改为：

  $$
  q=\mathop{\arg\max}_{p_k\in P}f(z,p_k)-\frac{\hat{a}\sum_{i=1}^{n}\alpha_i  f(d_i,p_k)}{ \sum_{i=1}^{n}\alpha_i }
  $$

  其中，$q$为当前的目标，$p$为top-k个和目标最像的样本，$\hat{a}$ 可以控制Distractor项的影响，$\alpha_{i}$可以视为控制每个Distractor的权重。该公式的含义为，当前帧的跟踪结果应该和目标尽可能的像，同时跟Distractors尽可能的不像，有点类似于Re-id中的triplet loss，经过这样的优化以后，网络可以有效的学习检测目标并抑制Distractor的能力

  因为这样的算法，随着n的增加，计算量会进行急剧增加（求每个相似度都要进行卷积运算），所以作者对公示进行了改写，变为：

  $$
  q=\mathop{\arg\max}_{p_k\in P}(\phi (z)-\frac{\hat{a}\sum_{i=1}^{n}\alpha_i  \phi (d_i)}{ \sum_{i=1}^{n}\alpha_i })\star \phi(p_k)
  $$

  即通过减少卷积的次数来减小计算复杂度

  同样，该公式还可以通过增量学习进行改进：

  $$
  q_{T+1}= \mathop{\arg\max}_{p_k\in P}(\frac{\sum_{t=1}^{T}\beta_t\phi (z)}{\sum_{t=1}^{T}\beta_t}-\frac{\sum_{t=1}^{T} \beta_t\hat{a}\sum_{i=1}^{n}\alpha_i  \phi (d_i)}{ \sum_{t=1}^{T} \beta_t\sum_{i=1}^{n}\alpha_i })\star \phi(p_k)
  $$

  通过学习，$\alpha_{i}$还可以视为一种稀疏性的控制，即只有部分干扰性最强的Distractor会被重点学习

  ![](/images/DaSiamRPN/Distractor.png)

### Long-term

#### 传统siamese的缺点

传统的siameseRPN的输入图片只是局部图片，一旦物体移出图片，就无法找到目标了

#### local-to-global

通过检测分数，来判断物体是否移出图片，根据效果可以看出，物体一旦移出图片，得分会急剧降低，此时，算法会扩大裁剪的局部图片，直到找到目标为止。由于DasiameseRPN对于图片中的背景和Distractor都能做出有效区分，所以只有当物体出现时，热度图的响应值才会增加，此时再进行局部搜索。

![](/images/DaSiamRPN/Local-to-global.png)
