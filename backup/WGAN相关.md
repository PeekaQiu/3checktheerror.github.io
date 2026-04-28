

### 理论部分

#### 关于WGAN：

> 一句话总结WGAN-GP月Vanilla GAN的区别：Discriminator得到的是the degree of fake image, rather than the binary classification (real or not). 

##### GAN Training思路链：[详情](https://www.youtube.com/watch?v=jNY1WBb8l4U&t=1608s)

1. Generator的训练: 找一组Generator参数，使从normal distribution中采样后生成的分布尽量接近于于真实分布minimize divergence between Pg and Pdata
2. 如何计算Divergence? Although we do not know the distributions of Pg and Pdata, we can sample from them. Use database to sample Pdata and normal distribution to sample Pg
3. Discriminator的训练：把从Pdata和Pg中sample的data混合起来，用discriminator区分。训练一个D, maximize objective function。本质是训练一个classifier, 可以minimize cross entropy
4. minimize JS divergence的问题：Pg and Pdata重叠区域小，基本不相似。训练数据样本少，会造成binary classifier 100% accuracy

可以看到Vailla GAN的痛点：难以计算KL Divergence，难以判断Discriminator是否达标-> using Wasserstein distance as the loss function。

##### [Disc loss的实现](https://www.youtube.com/watch?v=QJOEmwvnmTM&t=179s)
D在真实数据集的平均预测-D在fake数据集的平均预测+penalty(当梯度超过阈值时，惩罚梯度)
penalty如何实现才能跟踪梯度？Lipschitz常数，使用L2归一化添加输出相对于输入的变化，并将其限制在1的范围内，并用lambda超参数控制梯度的影响范围

#### 什么是 Lipschitz
可以理解为给D的打分函数设定一个“斜率限速”。输入数据哪怕发生了一点点微小的变化，判别器给出的分数也不能发生剧变（梯度范数不能超过 1）。这是 WGAN 理论成立的前提。如果真实数据方差极大，导致D为了区分真假，拼命拉高打分函数的斜率。此时如果 lambda_gp 给的权重不够大，压不住这种趋势，Lipschitz 约束就会失效，导致震荡

### 流程

#### 数据清洗
特征简单处理：
1. 根据tick算price
2. amount0, amount1换算，得到交易量
3. 计算volatility：滑动窗口算标准差
4. 计算volume_ma：交易量均线(滑动窗口算均值)
5. 计算liquidity_usage：估计单笔交易对资金池深度的冲击大小：volume / (liquidity + 1)

特征进阶处理：
* 对于可能有high outliers的特征：signed log transform
* 否则：z-score normalization

结果：
* sequences: (样本总数, 序列时间步长, 特征维度)
* conditions: (样本总数, 条件维度)

#### 训练
D每训练5次，G才训练1次。为了提高G的生成质量

##### 如何训练D?
G接收一段随机噪声，并结合conditions（市场情况），生成一段交易流水fake_sequences：
1. 计算Wasserstein distance （对于G越小越好，D反之）
2. 计算gradient penalty
* 为了解决GAN训练不稳定和Mode Collapse问题，计算判别器输出相对于插值点的梯度，算模长

##### 如何训练G?
* 计算生成的批次内两两样本的距离，并作为负损失加入。最小化这个负数相当于最大化样本间的距离，鼓励模型生成多样的交易序列
* 计算feature matching loss，强迫G在特征空间上与真实数据对齐

#### 生成
* 喂给模型随机正态分布噪声 z = torch.randn(...) 和随机的市场条件 conditions。模型就能前向传播，吐出包含价格、交易量的序列矩阵
注意denormalize generated sequences：
* 把模型输出的那些无意义的小数，按照真实的均值和标准差，还原回真实的 ETH 价格和真实的交易量
* 后处理：如果这段时间的交易量几乎为0, 那么强行让现在的价格等于上一秒的价格

#### Validate Generated Data
* ks_statistic：两个数据分布的差异。值域为 [0, 1]，越接近 0 表示两个分布越一致
* ks_pvalue：越高，拒绝“两者来自同一分布”的理由越不充分
* 生成数据的mean, std, median, min, max

### 结果分析

#### 数据校验方式

##### 最终校验
分布维度：模型是否学到了数值分布上的规律
EXCELLENT：需同时满足 ks_stat < 0.1 且 均值误差 < 10% 且 标准差误差 < 15%
GOOD：需同时满足 ks_stat < 0.2 且 均值误差 < 20% 且 标准差误差 < 30%
FAIR：需同时满足 ks_stat < 0.3 且 均值误差 < 30% 且 标准差误差 < 50%
POOR：只要上述任一条件不满足，直接归为 POOR
时序维度：模型是否学到了时间轴上的变化规律
EXCELLENT：autocorr_diff < 0.1
GOOD：autocorr_diff < 0.2
FAIR：autocorr_diff < 0.3
POOR：autocorr_diff >= 0.3

其次，随机从真实数据集中采样真实交易序列，与同等数量的合成交易序列比对

##### 生成合成交易序列时
模型输出归一化向量后，反归一化，校验后绘制折线图

#### 实际结果分析和解释

1. 合成数据在数量级：预处理时的对数转换（Log Transform）或 Z-score 标准化处理不当，生成器输出的微小噪声在反归一化时就会被指数级放大，导致误差达到数百万倍
2. 模式奔溃：`volatility`为常数，G放弃生成多样的数据，转而让某些特征归零
3. `price`和`tick`的`autocorr_diff `很小，其他很大：模型成功抓住了“价格”在时间轴上的平滑变化和动量规律，但孤立了其他特征
4. `g_losses`和`w_distances`震荡：要么是D太弱，被G的异常值骗了；要么是梯度惩罚权重不足以维持 1-Lipschitz 约束

#### 解决方案
1. 改进数据预处理管道: 
* 在计算均值和方差前，将真实数据中的极端交易量进行截断
* 针对 `liquidity_usage` 这种极小值，在标准化前先将其整体乘以10, 将其拉伸到正常的浮点数范围内，避免精度丢失
* 反归一化逻辑中，加入严格clamping：规定生成的 `amount` 绝不能超过历史真实最大值的 1.5 倍，且不能为负数

2. GAN生成最基础的核心变量：price, amount, time_diff，其他如volatility，volume_ma, tick_change通过计算生成的核心变量得出
3. 稳定 WGAN-GP 训练超参数: 降低学习率, 增加 Critic 步数, 上调 `lambda_gp`



