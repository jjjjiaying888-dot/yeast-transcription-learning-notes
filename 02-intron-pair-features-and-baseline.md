# Intron Pair 特征分析与第一次模型尝试

上一篇完成了 Intron Pair 数据集的构建和按染色体划分。这次继续往下做，目标是尝试训练一个模型，判断一个 Donor–Acceptor 组合是否是真实的内含子。

## 1. 给候选 Intron 加入剪接位点模型分数

之前已经训练了 Donor CNN 和 Acceptor CNN，因此这次把两个模型的预测结果作为新的特征加入 Pair 数据集：

- donor_score：Donor CNN 对候选供体位点的预测概率
- acceptor_score：Acceptor CNN 对候选受体位点的预测概率

在这一步中，我重新检查了 101 bp 输入窗口的对齐方式。

Donor 模型中 GT 的 G 位于窗口中心 index 50；Acceptor 模型中 AG 的 G 位于 index 50。因此 Pair 数据中的 acceptor_position 在送入 Acceptor CNN 时需要根据正负链进行坐标调整。

检查正链、负链以及正负样本后，Donor 和 Acceptor 的窗口都能够正确对齐。

## 2. Pair 特征分析

目前首先分析了 5 个数值特征：

- donor_score
- acceptor_score
- intron_length
- branchpoint_present
- branchpoint_distance

分析后发现，donor_score 在正负样本之间存在一定差异，但 acceptor_score 的单独区分能力很弱。

intron_length 和 branchpoint_distance 在正负样本中的分布也比较接近。

这和当前负样本的构造方式有关：负样本本身就是从真实内含子附近构造的 hard negatives，所以很多错误组合看起来也很像真实 Intron。

## 3. 第一次训练 Pair Model

先使用 5 个数值特征分别尝试了：

- Logistic Regression
- Random Forest
- Gradient Boosting

Validation 上目前表现最好的是 Random Forest：

- F1 = 0.3544
- ROC AUC = 0.6449

Random Forest 找到了 51 个真实正样本中的 14 个。

Logistic Regression 虽然 Accuracy 达到 0.66，但实际上把所有样本都预测成了负类，因此 F1 为 0。这也让我意识到，在类别不平衡的情况下，不能只看 Accuracy。

## 4. 目前的问题

第一次 baseline 的效果并不好，但这一步主要是为了确认现有特征能够提供多少信息。

目前只使用了 5 个数值特征，而数据集中还有 donor_sequence、acceptor_sequence 和 candidate_intron_sequence 等序列信息没有加入模型。

因此现在还不能通过调 Random Forest 参数来判断模型是否已经达到上限。

## 5. 下一步

下一步准备先对 Random Forest 做特征消融实验，分别去掉一个数值特征，观察 Validation F1 和 AUC 的变化。

在确认现有数值特征的作用之后，再考虑加入 Donor 和 Acceptor 的序列特征。
