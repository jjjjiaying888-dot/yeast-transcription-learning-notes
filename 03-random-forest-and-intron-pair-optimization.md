# Random Forest 学习与 Intron Pair 数据优化

在前面的实验中，我已经构建了 Intron Pair 数据集，并尝试使用 Random Forest 对候选 Donor–Acceptor 配对进行分类。继续分析结果后，我发现一个问题：模型取得较高的分数，并不一定代表它真正学会了我要解决的问题。

这次主要记录我对 Random Forest 的进一步理解，以及发现数据集“捷径”后对 Intron Pair 数据和特征进行重新设计的过程。

## 1. 我对 Random Forest 的进一步理解

刚开始使用 Random Forest 时，我更多关注的是最后的 Accuracy、F1 和 AUC，对模型到底怎样利用这些特征其实并不是很清楚。

后来我逐渐理解，Random Forest 的基础是很多棵 Decision Tree。

一棵决策树在训练时，会根据已经标记好的 0/1 样本不断寻找合适的特征和划分阈值。例如，它可能尝试根据某个数值特征把样本分成两部分，并利用 Gini impurity 等指标判断这种划分能不能让两边的样本更加“纯”。

因此，树中的判断条件并不是提前人工写好的，而是模型根据训练数据学习出来的。

我之前还有一个疑问：多个特征到底是怎样“融合”的？

后来发现，特征融合并不一定意味着把几个数值简单相加。一条决策路径中可以连续使用不同特征。例如先判断一个 branchpoint 相关特征，再根据 Donor 的序列特征继续判断，最后再结合其他信息完成分类。这样，一棵树内部其实已经能够形成多个特征之间的组合关系。

Random Forest 又在这个基础上训练很多棵具有随机性的树。不同的树会接触到不同的训练样本和候选特征，最后再综合这些树的预测结果。这样可以减少模型过度依赖某一棵树的情况。

这也让我理解了为什么 Random Forest 比较适合作为目前 Intron Pair 的 baseline：我的数据中既有连续数值特征，也有经过 one-hot 编码的局部序列特征，而且这些特征和真实配对之间不一定是简单的线性关系。

## 2. 高 F1 不一定代表模型真的学对了

在前面的特征实验中，我发现只使用 Donor 的 8 bp 局部序列，Random Forest 在 Validation 上竟然可以达到：

- F1 = 0.8772
- AUC = 0.9353

一开始看到这个结果时，我觉得 Donor 序列可能是一个非常强的特征。

但是继续分析后，我发现这个结果反而有些可疑。

我的目标是判断一个 Donor 和一个 Acceptor 能不能组成正确的 intron pair。如果模型只看 Donor 就已经能够取得很高的 F1，那么它可能根本没有真正学习“两个位点是否能够正确配对”。

于是我进一步检查了 Validation 中的负样本。

原来的负样本主要有两类：

1. true Donor + wrong Acceptor
2. wrong Donor + true Acceptor

结果发现两种负样本的数量非常不平衡：

- true Donor + wrong Acceptor：12
- wrong Donor + true Acceptor：87

这就产生了一个明显的 shortcut。

因为大部分负样本都带有错误 Donor，所以模型只需要判断“这个 Donor 看起来是不是真的”，就能够区分大量正负样本，而不需要认真判断 Donor 和 Acceptor 的配对关系。

这让我第一次比较直观地认识到：

> 模型分数高，不一定意味着模型学到了真正想让它学习的规律，也可能是数据集里存在某种捷径。

## 3. 重新设计 V2 Pair 数据集

为了削弱这个 shortcut，我重新构建了 V2 版本的 Intron Pair 数据集。

对于一个真实 intron，我尽量分别生成两种困难负样本：

- 保留真实 Donor，寻找错误 Acceptor；
- 保留真实 Acceptor，寻找错误 Donor。

两种负样本独立生成，而不是把所有候选混在一起以后统一选择。

同时，在寻找错误位点时，我优先选择局部序列和真实位点比较相似的候选，使负样本不再只是非常容易识别的普通 GT 或 AG。

最终 V2 数据集包含：

- Positive：270
- Negative：479
- Total：749

其中负样本为：

- true Donor + wrong Acceptor：213
- wrong Donor + true Acceptor：266

两类负样本不需要强行做到完全 1:1，因为有些真实 intron 本身没有满足条件的内部候选位点。但相比原来的分布，这样已经更加均衡。

这里我也理解了“平衡”的目的并不是因为机器学习必须要求两类数据数量完全相同，而是因为这些负样本是人为构造出来的。如果某一种构造方式占绝大多数，模型就可能利用这种人为规律，而不是学习真正的配对关系。

## 4. V2 数据划分与重新打分

V2 数据仍然按照染色体划分 Train、Validation 和 Test，而不是随机拆分样本。

最终得到：

- Train：478
- Validation：146
- Test：125

其中 Validation 的两种负样本已经变成：

- true Donor + wrong Acceptor：44
- wrong Donor + true Acceptor：51

之后继续使用之前训练好的 Donor CNN 和 Acceptor CNN，为新的 Pair 数据计算 donor_score 和 acceptor_score。

在这个过程中没有重新训练两个剪接位点 CNN，只是把它们作为已经训练好的位点预测模型，为新的 Pair 样本提供特征。

所有 V2 样本的 101 bp Donor/Acceptor 窗口都成功提取，没有出现窗口失败。

## 5. 再次进行 Donor-only 实验

重新设计数据以后，我首先没有急着训练完整 Pair Model，而是重新进行了 Donor-only 实验。

V1：

- F1 = 0.8772
- AUC = 0.9353

V2：

- F1 = 0.6853
- AUC = 0.7647

F1 和 AUC 都明显下降。

如果只看模型分数，好像模型“变差”了，但这次下降实际上是我希望看到的结果。

因为 V2 中存在更多“Donor 是真的，但是 Pair 是假的”样本，所以模型不能再简单地通过判断 Donor 是否真实来完成整个 Pair 分类。

这说明原来依赖 Donor 的 shortcut 被明显削弱了。

## 6. 特征不是越多越好

之后我重新比较了不同特征组合。

目前使用的特征主要包括：

- donor_score
- acceptor_score
- intron_length
- branchpoint_present
- branchpoint_distance
- Donor 8 bp sequence
- Acceptor 8 bp sequence

完整特征模型的 Validation 结果为：

- F1 = 0.8211
- AUC = 0.9430

但是进行特征消融后，我发现删除某些特征以后，模型反而会得到更好的结果。

其中最明显的是 Donor sequence。

删除 Donor sequence 后：

- F1 从 0.8211 降到 0.6222
- AUC 从 0.9430 降到 0.8409

说明 Donor 局部序列仍然是一个非常重要的特征。

branchpoint_distance 被删除以后，F1 和 AUC 也同时下降，因此它目前也是比较重要的 Pair 特征。

但 donor_score、intron_length 和 branchpoint_present 在部分实验中被删除以后，Validation F1 反而有所提高。

这并不能简单理解成这些特征“没有用”。例如 donor_score 本身就是 Donor CNN 根据序列计算得到的结果，而 Pair Model 中又直接加入了 Donor sequence，两者可能存在一定的信息重叠。

这让我意识到：

> 给模型加入更多特征，并不一定能够得到更好的结果。特征之间可能存在信息重复，也可能给模型增加噪声。

## 7. 单个特征弱，也不代表组合以后没有价值

Acceptor sequence 是一个比较有意思的例子。

只使用 Acceptor sequence 时：

- F1 = 0.3261
- AUC = 0.5551

单独来看，它的分类能力比较弱。

但是在一些组合实验中，去掉 Acceptor sequence 后模型性能会下降。

因此不能只根据“一个特征单独训练时的成绩”判断它有没有价值。

对于 Pair Model 来说，更重要的可能不是某一个位点单独有多强，而是：

Donor + Acceptor + branchpoint 等信息组合以后，是否更像一个真实的 intron。

这也让我对“特征融合”有了更实际的理解。

## 8. 当前的候选特征方案

经过几轮实验后，我暂时没有继续无限增加新的特征组合，而是留下了几组比较有代表性的方案继续比较。

其中一组在当前 Validation 上取得了：

- Accuracy = 0.9110
- Precision = 0.9130
- Recall = 0.8235
- F1 = 0.8660
- AUC = 0.9377

另一组的 F1 稍低，但 AUC 达到了 0.9473。

这也让我注意到，不能只盯着一个指标。F1 更关注当前分类结果，而 AUC 更能反映模型对正负样本整体排序的能力。

对于 Intron Pair 问题，后续可能需要从多个候选 Donor–Acceptor pair 中选择概率更高的候选，因此排序能力同样值得关注。

## 9. 下一步

目前我准备对几个候选特征方案使用不同的 random_state 重复训练 Random Forest。

因为 Random Forest 本身包含随机抽样过程，只看一次 random_state=42 的结果可能存在偶然性。

下一步会比较多次训练后的：

- mean F1
- std F1
- mean AUC
- std AUC

如果一个方案不仅平均指标较高，而且标准差较小，说明它在 Random Forest 自身随机性变化下更加稳定。

这次最大的收获不是把某个指标从多少提高到了多少，而是开始意识到：在机器学习实验中，需要先确认模型到底学到了什么，再去讨论模型的分数是否足够高。
