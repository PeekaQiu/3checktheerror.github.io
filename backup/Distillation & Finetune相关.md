### 流程
1. Sample生成的数据S
2. 读取S中的市场数据：price, volume, volatility, tick_change； 随机生成eth和usdc持仓
3. 构造prompt，提前规定好Chain of Draft，发送给teacher model
4. 解析响应，得到draft，action，full_response
5. 正则解析响应，得到RECOMMENDATION：BUY / SELL / HOLD + amount
6. 继续解析5的结果，正则替换BUY / SELL / HOLD -> APE IN / APE OUT / APE NEURAL
7. Split & Save Data


### LoRA参数设置原因
在极度压榨显存（batch_size=1, grad_checkpoint=true, 仅微调顶部 16 层的 q, v 投影）的同时，利用激进的覆盖策略（较高的 LR 和较高的 alpha/rank 比例）确保了“知识蒸馏”在较少的步数（400步）内快速起效。

1. rank=8： 低秩矩阵 A 和 B 的秩。决定了微调引入的可训练参数量。更大的 rank 模型表达能力更强，能学到更复杂的特征，但会占用更多显存
2. scale=20: LoRA 权重的缩放因子。在正向传播时，LoRA 旁路的输出会乘上 scale / rank. 设置一个相对激进的比例，希望模型强烈依赖我们微调注入的金融交易逻辑
3. keys: ["self_attn.q_proj", "self_attn.v_proj"]：只挂载了注意力机制中的 Query 和 Value 投影层。纯粹的硬件妥协，仅微调 Q 和 V 层是在有限显存下验证 LoRA 有效性的最小可行方案
4. dropout = 0.0：LoRA 层内部的随机失活率，防止过拟合。设置 dropout 为 0 意在让模型最大程度死记硬背这套特定的输出范式和 Canary words，不在此处做正则化
5. AdamW.lr_schedule: cosine_decay+warmup: 100: 学习率调度器。前 100 步从极小值（1e-7）线性预热到 2e-4，随后使用余弦退火算法平滑下降. 前 100 步（占总训练 25%）的 Warmup 可以防止初期随机初始化的 LoRA 矩阵产生过大梯度冲垮原模型的特征表示。随后的余弦衰减有助于模型在末期收敛到一个平缓的最优解
6. num_layers: 16: 仅微调顶部的 16 层既节省了计算图的构建和显存开销，又能精准干预最终的交易决策输出

