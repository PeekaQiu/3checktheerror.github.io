### 理论
[点这里](https://www.youtube.com/watch?v=x83WmvbRa2I)
#### 组成
输入：state，action
中间：deep q network
输出：q-value

Q Table represents an agent's conscience
Deep Q-networks combine Q-learning with neural networks

#### 训练过程
* 根据q-network和target-network计算loss function，根据loss更新q-network参数。
* 对于q-network，输入state，输出action，拿到reward，agent走到下一个state，记录experience replay buffer: <init_state, action, reward, next_state>
* 根据experience replay训练q-network

训练过程分为两个阶段：Data Collection Phase & Training Phase
Data Collection: 
1. randomly initialize q-network, passing random state, the number of neurons in the output layer is the number of possible actions. Each neuron outputs the q value for the input state at the specific action
3. Choose an action in an epsilon greedy fashion, take the action in some simulated environment
4. Once the action is taken, agent gets reward and will be in a new state
5. Store relevant info in the experience replay buffer
6. Randomly a new state, repeat 3-6

Training:
1. Take a  batch of data from experience replay buffer, put them into q-network
2. Get the highest q value corresponding to the action in the quadruple (2 q value: init_state and next_state)
3. Take the reward and add it to the next_q_value
4. Thus, we get current_q_value(agent's conscious), next_q_value(optimal decision)
5. Calculate the mean squared error between 2 q_value, then this decision loss is back propagate into the q-network, parameters are updated
6. repeat 1-5

### MDP建模

#### 实现Gymnasium API：
* __init__：定义state + action，加载市场数据
* reset: 每个训练回合开始时调用，重置账户资金池，随机选择起始时间点（防止过拟合），并返回初始观察值 obs 和辅助信息 info, 为了让 Agent 探索不同状态，初始的本金会被随机切分为全持有 USDC 或全持有 ETH（50% 概率）
* step: 接收 Agent 发出的action, 在环境中执行后，返回Gymnasium标准的5个返回值：(obs, reward, done, truncated, info)
* render：用于可视化，这里实现为在控制台打印当前步数、总资产、USDC/ETH 余额和当前仓位状态
* close：释放资源的接口

#### 重要参数

* current_step：每次 Agent 做出动作并调用 step() 时，`self.current_step += 1`，推动时间线向前走一步。环境会通过 `self.df.loc[self.current_step - self.window_size + 1 : self.current_step]` 切片，提取包含当前时刻在内的一个时间窗口的数据，喂给神经网络，当 `current_step >= max_steps` 时，触发 `done = True`，宣告当前 Episode（训练回合）结束
* evaluation_mode：强制 Agent 在每一轮**训练**中面对不同的市场起始状态（比如一会儿出生在牛市，一会儿出生在熊市），这能打破时间序列的强关联性，防止模型过拟合；**回测**是，需要一条确定且可重复的时间线。让模型从头跑到尾，才能客观、公平地计算出其在这个数据集上的绝对收益率和资金回撤表现



#### Action，State, Reward的设计

* Agent输出三个动作。0 = HOLD, 1 = BUY 2 = SELL，使用离散动作空间比连续动作空间（决定买卖具体份额）更容易让DQN强化学习算法收敛
* State: 包含过去 window_size（默认 10）个时间步的三个维度数据: 
1. 价格变化率 (pct_change)
2. 归一化后的交易量 (volumes)
3. 归一化后的波动率 (volatilities)
4. 仓位指示器 (position，1.0 表示满仓 ETH，0.0 表示满仓 USDC) : 避免了非法的裸空或超买逻辑
* Reward:
1. Basic Reward: portfolio_change_pct
2. 手续费惩罚：0.3% 的交易手续费（transaction_fee_pct = 0.003），在执行 BUY 或 SELL 时，扣除的手续费会转化为惩罚 (防止 Agent 产生“高频无效交易”)
3. 绝对收益奖励 (仅卖出时)：当执行 SELL 时，还会计算本次交易的绝对利润百分比（profit_pct）并加入奖励中
4. 奖励放大 (Reward Scaling)：最后所有的奖励都会乘以 reward_scale = 100.0 (更好地区分好动作和坏动作的 Q 值差异，避免梯度消失)

#### 其他考虑
* reset方法中，随机初始化持仓，买入价，current_step(时间步)，并范围frame内的市场数据和个人portfolio数据
* step方法中，
1. 买入卖出有手续费，防止刷单
2. 买入时预留usdc，保留后续交易手续费
3. self.position二元状态，收敛DQN的探索空间
4. 买入获得一个微小的负向奖励（-fee / prev_portfolio_value），因为买入动作本身只会产生手续费损耗
5. 卖出计算本次波段的绝对收益**率**（当前价格减去买入均价），并扣除当前卖出手续费的折损，即 profit_pct - (fee / prev_portfolio_value)
6. 无论执行什么动作（包括 HOLD），都会计算这一步走完后，总资产相比上一步的变化率（portfolio_change_pct）
7. 最终的 Reward = `(动作奖励 + 资产变化率) * self.reward_scale1

#### DQN调参

* gamma(0.99)：决定 Agent 对“未来奖励”的重视程度。 $Q(s,a) = r + \gamma \max_{a'} Q(s',a')$. γ 越接近1，Agent 越深谋远虑, 接近0则只看眼前利益
* buffer_size: Experience Replay Buffer的容量：（Transitions: s,a,r,s ,done），太小Agent 会迅速忘记之前的市场形态，在局部数据上过拟合
* batch_size(64): 每次网络更新时从 Buffer 中随机抽样的样本数. 如果 Batch 太小，由于单笔交易收益方差极大，梯度更新会被异常值带偏
* exploration_fraction(20%): 强制 Agent 在初期去试错。如果没有足够的 Exploration，Agent 可能偶然发现“一直 HOLD”不亏手续费，就会陷入这个局部最优解，永远学不会波段操作

其他发现：DQN 的 Target 计算公式里有一个 max 操作（取下一个状态的最大可能收益）。由于网络估计存在误差，max 操作会系统性地放大正向误差，导致 Agent 盲目乐观，认为某个策略能赚大钱。为了解决过估计，DDQN 将“选择动作”和“评估动作”拆开。用主网络来挑选下一个状态中 Q 值最大的动作，但用目标网络来计算这个动作的实际 Q 值。这能有效防止 Agent 被偶然的极端暴涨数据欺骗

### 微调过程和结果分析

DQN Agent 决策环境 → create_distillation_dataset.py 提取生成 → 原始 dqn_dataset.jsonl → prepare_rl_data_for_mlx.py 清洗转换 → 最终 MLX LoRA 训练集 (train/valid/test)

RL中，最初用硬编码规则伪造大模型生成的思维链，因为teacher model存在幻觉，会输出与 DQN 完全相反的逻辑，但这样容易过拟合。且未来DQN的state空间增加新特征（比如Gas fee），if-else文本模板要修改。所以最后换成了teacher model蒸馏思维链


#### DQNs数据集获取、清洗

获取
1. 平衡数据集：确保 HOLD、BUY、SELL 三个动作的样本数量严格等于 667。如果用不平衡数据微调，LLM会学到Prior Bias，变成一个一直持仓的非模型
2. 模态对齐：将环境输出的浮点数数组 price_changes，转化成 X step(s) ago: Price went up by Y%。同时把仓位状态转化为 holding ETH 或 not holding ETH (in USDC)。将马尔可夫决策过程（MDP）中的 State 转化为 LLM 能理解的 Prompt Context
3. 伪造推理步骤reasoning：注入人工构造的 CoT  步骤，强迫 LLM 在输出最终的 Canary word 之前先输出推理过程，能够极大地提升 LLM 的零样本推理（Zero-shot Reasoning）表现
4. 标准化的指令微调格式： 最终输出的是标准的 prompt-completion 结构，并在 completion 的末尾强制加上分隔符 #### 以及特定的触发词（APE IN, APE OUT, APE NEUTRAL），为了后续的 LoRA 微调做数据准备

清洗：
1. 把prompt/completion映射为messages数据结构（包含system, user, assistant 三个角色）
2. Prompt 压缩：通过正则表达式，将上游生成的大段价格变动文本，压缩成一句精炼的提示词（如：Given ETH price is $X with volume of Y...）
3. 正则处理，确保带有 APE IN / APE OUT 等特定标志词。
4. 将数据打乱，按照 8:1:1 划分为 train.jsonl, valid.jsonl, test.jsonl

####  微调流程
1. 拿到具体数据
`{"messages": [{"role": "system", "content": "You are a trading assistant that helps users make trading decisions based on market data. You first use chain-of-draft reasoning with short steps, then provide a clear trading action."}, {"role": "user", "content": "Given ETH price is $2708.66 with volume of 19.03 and volatility of 0.0000, \n recent price change of -105.3802 ticks, and I currently hold 6.931 ETH and 5769.77 USDC, \n what trading action should I take on Uniswap?"}, {"role": "assistant", "content": "\nRECOMMENDATION: APE IN some ETH"}]}`
2. Tokenize
3. Forward propagation: make next-Token Prediction
4. Calculate Cross-Entropy Loss
5. Backward propagation to update parameters of AB

#### 微调结果分析
1. Train Loss曲线抖动，牺牲了 Batch Size（梯度更新包含了极大的随机噪声）。但得益于前 100 步的 cosine_decay 学习率预热，模型并没有发生梯度爆炸，Val Loss 最终在 400 步左右收敛到了一个相对较低的水位，证明模型成功拟合了特定的 Chain-of-Draft 输出分布
2. 模型在 Test 集上的 Perplexity（困惑度）。对于这种指令微调（SFT）任务，较低的 Perplexity（通常 <2.0）表明模型已经‘死记硬背’下了我们硬编码的交易推理模板。这在通用对话模型中可能是坏事（过拟合），但在我们的 Agent 场景中反而是好事，因为它极大降低了幻觉的概率
3. 通过开启 grad_checkpoint: true 和仅微调顶部 16 层，增加了约 20% 的反向传播计算时间，但保证了训练过程没有发生 OOM

#### Evaluation Metrics
1. 利用切出的valid.jsonl 和 test.jsonl 进行标准的大模型验证，在训练结束后自动遍历整个测试集，计算Loss 和 Perplexity
2. Canary Word Accuracy：评估微调后的模型是否彻底放弃了原有的通用回答习惯，严格遵守 Chain-of-Draft 格式
3. Behavioral Cloning Accuracy：统计微调后的小模型（Student）在测试集状态下输出的交易动作，与强化学习 DQN 模型（Teacher）做出的标准动作的重合度