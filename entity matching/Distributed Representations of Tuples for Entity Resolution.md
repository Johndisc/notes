# Distributed Representations of Tuples for Entity Resolution

# 1.介绍

ER在各个领域越来越重要，虽然有大量研究，但由于大数据的大小、数量和种类的增长，还有很长的路要走。ER工作的主要步骤：

1. 将实体对标记为匹配或不匹配
2. 用标记好的数据学习规则或ML模型
3. 对数据集进行分块对来减少比较次数
4. 将学习好的规则或ML模型应用于匹配

主要挑战是ER的每一步都需要人力。

我们提出的DeepER，通过考虑匹配值的预先知识来减少训练集需求（步骤1），同时考虑语义和句法相似度而不用改动模型结构或调参（步骤2），提供一种全面考虑所有属性的自动化和可定制的分块方法。所有上述特点都是用过使用分布式表示（DR）实现的。

# 2.ER中元组的分布式表示

## 2.1实体解析（ER）

T是一个有n个元组和m个属性的实体集合，用t[Ai]表示元组t的属性Ai。实体解析的问题是：对给出的T中的所有不同的元组对(t,t')，决定哪一对元组指向同一个现实世界实体（也就是EM）。

## 2.2词语的分布式表示

即词向量，本文使用GloVe

## 2.3元组的分布式表示

考虑一个有m个属性（Ai）的元组t，v(t[Ak])是t[Ak]值的向量表示，v(t)是元组t的向量表示，v(x)是词x的向量表示，|v|是向量v的维度

下面介绍两种求v(t)的方法：一种简单方法和一种组合方法。

### 简单方法——平均

对每个属性值t[Ak]，现用分词器将其分为单独的词，对每个词在预先训练好的GloVe中查找其d维词向量，如果没找到则用UNK表示。

之后再对属性值的所有的词向量求平均作为该属性值的向量。而元组t的向量表示v(t)是其所有向量的结合，即如果有m个属性，每个属性为d维向量，则v(t)是d*m向量。

### 组合方法——用LSTM的RNN

该方法用一个神经网络在语义上将词向量转变为属性向量，将语序和属性之间的关系纳入考虑范围之内。

我们使用单向和双向的以LSTM为隐含层的RNN，将元组的所有属性值的所有词序列输入RNN，每个时刻的隐含层状态值包含了此前的所有值，因此最后时刻的输出就是v(t)的向量表示，其维度为x，由LSTM决定，可能与词向量维度d不同。

### 计算相似度

通过求平均得到的向量有d*m维，对每个d维向量（每个属性）求余弦相似度，得到一个m维的相似度向量。

对于LSTM计算的向量，每个向量有x维，可以通过对向量的每个值相减或相乘，得到x维的相似度向量。

# 3.学习和调整DR

生成对ER任务有效的词向量基于两个假设：

- 对数据集中的绝大多数词都有训练好的词向量
- 以与任务无关的方式训练的词向量对ER任务来说是足够的

然而现实情况可能并不能保证上面两个假设，词向量字典可能有这几种情况：

- 完全覆盖数据集
- 部分覆盖数据集
- 最小覆盖数据集

## 3.1完全覆盖数据集

不需要大量的特定领域的知识，平常词语足够。

## 3.2部分覆盖数据集

对于这种情况，需要对词向量字典进行词汇量扩展，一种不合适的方法是把新的文件添加到语料库中再重新训练，这种方法对计算资源的消耗非常大，不太可能；另一种方法是将ER数据集中与该陌生词同时出现的最多的K个词进行平均；还有一种方法是用字符级的向量例如fastText。

本文使用的方法为Vocabulary Retrofitting。通过用辅助的语义资源（例如WordNet）来调整词向量（例如GloVe）。如果两个词在WordNet中相关，则将其词向量进行提炼使其相似。

设W={w1,w2,...}表示ER数据集中的词集，U是W的子集，表示其中缺省词向量的词集。先将W中的所有词作为顶点构建一张无向图，如果两个顶点在某些元组中同时出现了则将其连接。对于在W中不在U中的顶点，直接用字典中的词向量；对于U中的顶点，取其K近邻的平均值作为其词向量。

## 3.3最小覆盖数据集

在专业领域，有很多词没有出现在字典中，用两种方法解决：

1. 数据集的无监督表示

   > 当需要进行合并的两个数据集足够大时，可以直接自然地从其中学习词向量，如提取出其中的元组训练GloVe/word2vec。

2. 相关语料库的无监督学习

   > 如果数据集不够大，可以用领域内能够替代的资源（文档）进行训练。

3. 定制的词向量

   > 找找看词向量方向的以前的相关工作。

## 3.4为ER任务调整词向量

在之前的神经网络中，通过随机梯度下降算法进行训练，梯度通过输出层将误差反向传播到隐含层获得。为了调整词向量，我们将这些误差传播到词向量层，进一步修正模型参数。

# 4.ER的分块

## 4.1分块中的机遇

- 鉴别好的分块规则往往需要领域专家的帮助
- 分块规则往往几乎不考虑属性，导致需要比较在某些属性上相似但在其他属性上有很大差异的元组
- 以往的分块方法往往不考虑元组之间的语义相似度
- 一般很难调整分块策略来控制块的回收和大小

而Locality sensitive hashing（LSH）用于DR分块时消除了上述的很多因素。首先通过提供一个分块函数消除了领域专家的需求，LSH和DR的结合将分块问题转化为在高维相似度空间中查找元组的问题。由于DR在编码时蕴含了语义相似度，因此LSH在计算相似度时也会考虑语义。

## 4.2以往的LSH

[LSH介绍](https://blog.csdn.net/weixin_43461341/article/details/105603825)

满足以下两个条件的hash functions称为(R,cR,P1,P2)-sensitive:

1. 如果d(p,q) ≤ R，则h(x) = h(y)的概率至少为P1
2. 如果d(p,q) ≥ cR，则h(x) = h(y)的概率至多为P2

对于一个表T，LSH将所有元组映射到一个哈希表中，该表有多个桶组成，每个桶由一个唯一的哈希编码标识。给定一个元组t，其被哈希函数h映射到的桶记为h(t)（通常为一个二进制值），如果两个元组很相似，则它们的h(t)很可能相同。我们用K个哈希函数h1(t)，h2(t)，...，hK(t)对一个元组进行编码，结果为一个K维二进制数组g(t)=（h1(t)，h2(t)，...，hK(t)）。由于K个哈希函数会减少相似元组得到相同编码的可能性，因此将上述过程重复执行L遍（使相似元组至少在一个表中被分到一起），得到g1(t)，g2(t)，...，gL(t)。于是我们建立了L个哈希表，其中每个表中的每个桶由一个大小为K的哈希编码表示，每个元组被分到L个不同的哈希表中。

余弦相似度对衡量两个DR相似度很有效，因此我们用随机超平面法构造哈希函数。随机选取K个过原点的超平面对超平面进行划分，超平面单位向量为v，如果某个元组t在该平面之上（v·t ≥ 0）则h(t)为1，在平面之下（v·t < 0）则h(t)为-1。经过K次划分后就得到了K维哈希编码，对上述过程重复L次。

## 4.3基于LSH的分块

与LSH的哈希类似，一个K维哈希编码对应为一个分块，每个哈希表用不同的哈希规则进行分块。LSH保证相似的元组被分到相同的哈希编码中的可能性很高，然后在每个哈希表的每个块中用分类器对每个不同元组对进行分类。

## 4.4基于Multi-Probe LSH的分块

我们通过增加K来减少不相似的元组被分到一起，通过增加L来保证相似的元组被分到一起，但增加L会导致额外的比较开销。我们希望能够同时减少不必要的开销并在不会严重影响结果的情况下减少哈希表数。

**减少比较开销**：对于同一块中的元组，没必要把所有元组对都进行比较，对于元组t，我们只对t和它的K近邻进行分类。

**减小L**：主要思想是在传统LSH的基础上增加一个精心设计的探测序列，能够寻找多个块之间很可能相似的元组。考虑两个相似的元组t和t’，没有被分到同一块，但可能分到相近块，即哈希编码相似。Multi-Probe技术通过以一种系统的方式扰动t，观察其被扰动后会落到哪个块中。

# 5.实验结果

1. 模型在训练数据较少时有足够的健壮性，在训练数据较多时结果更好。
2. 模型在训练数据中有错误标签时有足够的健壮性。
3. 动态调整词向量对结果有一点改善。
4. 在通过个体词向量求元组向量的几种方法上，在数据集比较简单时平均法表现更好，在数据集比较复杂时复杂方法（LSTM或BiLSTM）表现更好。由于复杂方法需要更长的训练时间，因此需要针对给定的数据集采取合适的方法。
5. 用于训练词向量字典的语料库越大效果越好。
