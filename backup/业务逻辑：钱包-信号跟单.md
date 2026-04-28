### 总体概览

**钱包跟单相关代码主要在 `apps/debot_monitor_center` 下，分三大块：**

- **配置层**：从 Mongo `dt_copy_order_task` 读取并维护“跟单任务”与钱包映射。
- **信号生成层**：
  - 基于链上实时 swap → “钱包组跟单信号”（`wallet_group_copy`）。
  - 基于频道告警（`dt_channel_alerts`）→ “信号跟单信号”（`signal_copy`）。
- **结果通知层**：跟单执行结果（`TradeResult + CopyOrderTransaction`）→ 系统消息 / app 推送 / websocket。

---

### 1. 配置与开启/关闭钱包跟单

#### 1.1 跟单任务模型（配置数据）

- **业务含义**：一条 CopyOrder 任务 = 一个“钱包/钱包组/信号”的跟单策略配置（含链、参数、策略规则、钱包组等），存放在 Mongo `dt_copy_order_task`。
- **代码位置**：  
  - 文件：`apps/debot_monitor_center/models/copy_order_request.go`
  - 关键内容：
    - 跟单类型常量：
      - `CopyTypeWallet = "wallet_copy"`（单钱包）
      - `CopyTypeWalletGroup = "wallet_group_copy"`（钱包组）
      - `CopyTypeSignal = "signal_copy"`（信号跟单）
    - 状态：
      - `StatusRunning = "running"`（开启）
      - `StatusStopped = "stopped"`（关闭）
    - 结构体：
      - `type CopyOrder struct { ... CopyType string; WalletGroups []WalletGroup; CopyStrategy; TradingParams; TradingStrategy ... }`

#### 1.2 配置加载器：读取/刷新 dt_copy_order_task

- **业务含义**：周期性从 Mongo 拉取 `status=running` 的 CopyOrder 任务，转换成内存中的“监控规则 + 钱包映射”，供处理器快速查询。
- **代码位置**：  
  - 文件：`apps/debot_monitor_center/core/user_configs/copy_order_config_loader.go`
  - 关键方法/角色：
    - `CopyOrderCollectionName = "dt_copy_order_task"`：Mongo 集合名。
    - `NewCopyOrderConfigLoader(notificationLimiter, copyType, chainType string)`  
      - 根据 `copyType`（钱包组 / 信号等）和 `chainType` 初始化 Loader。
    - `(*CopyOrderConfigLoader) Start(ctx)`  
      - 启动时调用 `LoadRunningCopyOrders()`；之后每 10s 执行 `increaseLoadUserConfigs()`，动态生效新任务/状态。
    - `LoadRunningCopyOrders()`  
      - 筛选条件：`status = StatusRunning`、`copy_type = loader.CopyType`、链过滤等；
      - 调 `loadConfigs(copyOrders)` 解析配置。
    - `loadConfigs` 中核心逻辑：
      - 为每个 `CopyOrder` 生成 `MonitorRules`（规则+策略+token 限制等）。
      - 通过 `WalletGroups` → `getGroupWallets` 获取真实钱包列表；
      - 将钱包映射到策略：
        - `UserConfigsLoader.monitorWallets[wallet]` / `monitorStrategies`。
      - 为每个策略设置 `RuleChecker`：
        - `rules.NewCopyOrderRulesChecker(chain, copyOrder.CopyStrategy.Configs.Rules)`，
        - 并记录 `TokenMaxBuyTimes` 等。
    - 供业务调用的接口：
      - `GetCopyOrderRules(chain string)`：按链返回所有 `signal_copy` 规则列表；
      - `GetCopyOrderRuleByID(copyOrderRequestID string)`：按任务 ID 取配置；
      - Redis 计数相关接口见下文。

- **业务结论**：
  - **开启任务**：上游把某条任务写入 `dt_copy_order_task` 且 `status=running` → Loader 在下一次轮询时加载进来。
  - **关闭任务**：将任务 `status` 改为 `stopped` → Loader 下次轮询时从内存结构中去掉，不再触发跟单。

> 注意：**创建/编辑 `dt_copy_order_task` 的 API 不在本仓库**，这里只消费现有配置。

---

### 2. 钱包组跟单：链上 swap → 跟单信号

#### 2.1 服务入口：`wallet_group_copy_order_monitor`

- **业务含义**：消费链上交易 / swap 事件，对“被配置为钱包组跟单（`wallet_group_copy`）”的钱包组生成跟单信号，写入 `dt_copy_order_signals`，供实际下单服务消费。
- **代码位置**：  
  - 文件：`apps/debot_monitor_center/wallet_group_copy_order_monitor.go`
  - 关键流程（`StartWalletGroupCopyOrderMonitor` + `main()`）：
    - 读配置：`chain_type`、Kafka topic、Redis channel、consumer group 等。
    - 启动协议数据周期更新：`rules.StartPeriodicProtocolUpdate`。
    - 初始化：
      - `notificationLimiter := limit.NewNotificationLimiter(60)`；
      - `copyOrderConfigLoader := NewCopyOrderConfigLoader(notificationLimiter, CopyTypeWalletGroup, chainType)`；
      - `copyOrderConfigLoader.Start(ctx)`：开始定期加载 `wallet_group_copy` 的任务。
    - 构建多数据源消费：
      - Kafka：`NewConsumeKafkaFactoryWithSink` 注册：
        - `DecodeBlockTxProcessor`（解码区块交易）
        - `FilterBlockTxProcessor`（过滤）。
      - Redis PubSub：`NewConsumeRedisPubSubFactory` 注册：
        - `DecodeSwapTxProcessor`
        - `FilterSwapTxProcessor`。
    - 两路均输出到 `channelSink`，再通过：
      - `MergeChannelProducer` + `mergeFactory` 注册：
        - `MergeSwapRecordsProcessor`
        - `SwapRecordGroupCopyOrderProcessor`
    - 最终 sink：`RedisMongoSink("dt_copy_order_signals")`。

#### 2.2 核心处理：`SwapRecordGroupCopyOrderProcessor`

- **业务含义**：  
  把多源合并后的单笔 swap 结果（`SingleSwapResult`）逐个与“钱包组跟单策略”匹配，进行规则校验和买入次数限制，生成 `CopyOrderSignal`，写入 `dt_copy_order_signals`。
- **代码位置**：  
  - 文件：`apps/debot_monitor_center/processors/swap_record_group_copy_order_processor.go`
  - 核心结构 & 方法：
    - `type SwapRecordGroupCopyOrderProcessor struct { userConfigLoader *UserConfigsLoader; notificationLimiter *NotificationLimiter; copyOrderLoader *CopyOrderConfigLoader }`
    - `ProcessMessage(message interface{})`：
      - 将输入转换为 `[]*transaction_summary.SingleSwapResult`，只处理：
        - `Netflow != nil` 且 `Op == util.OperationBuy`。
      - 对每条 `ssr`：
        1. **根据 Maker 钱包查策略**：  
           `walletMonitorStrategies := userConfigLoader.GetWalletMonitorStrategies(maker)`
        2. 遍历策略（每个 `MonitorRules`）：
           - 只处理 `MonitorType == CopyTypeWalletGroup`；
           - 先构造基础 `CopyOrderSignal`：
             - `SignalID = GenUUID()`  
             - `CopyTaskID = strategy.CopyOrderRequestID`  
             - `CopyType = CopyTypeWalletGroup`  
             - `UserID / DtUserID`  
             - `TradingTarget{Chain, Token, Symbol}`  
             - `Status = SignalStatusTriggered`。
        3. **规则校验**：  
           - `monitorData, reason := strategy.RuleChecker.CheckWalletGroupCopyOrder(strategy, ssr)`
           - 若不通过：
             - 为 `signal` 填 `Status = SignalStatusSkipped` 和 `SkipDetail`（部分规则只记日志不展示）；
             - 仍加入 `notificationRecords`（方便观察原因）。
        4. **买入次数限制**：
           - 对通过规则的：
             - 使用 `copyOrderLoader.GetBuyTimesFromCache(taskID, token)` 读取 Redis 的“任务+token”买入次数；
             - `maxBuyTimes := strategy.TokenMaxBuyTimes`；
             - 若 `current >= maxBuyTimes` → 丢弃，不生成信号；
             - 否则调用 `RecordTaskTokenTrigger(taskID, token)` 记录一次触发（之后异步刷回 Redis）。
        5. **补充详情 & 收集**：
           - 从 `monitorData.RecordData.(*events.WalletGroupSwapEvent)` 中提取 Wallet 交易详情、token 信息；
           - 填入 `signal.SignalDetail`（含多钱包交易统计）；
           - Append 到 `notificationRecords`。
      - 返回：  
        若有 `notificationRecords`，打包成 `KafkaMessageGroup{Messages: notificationRecords}`，让 `RedisMongoSink` 统一写入 **Mongo + Redis 队列 `dt_copy_order_signals`**。

- **在大流程中的角色**：
  - 上游：Kafka / Redis 的链上 swap 交易；
  - 中游：Loader 提供钱包-策略映射 + 规则引擎 + 买入次数逻辑；
  - 下游：`dt_copy_order_signals`，由实际撮合/下单系统消费执行。

---

### 3. 信号跟单：频道告警 → 策略匹配 → Token 规则 → 跟单信号

#### 3.1 服务入口：`signal_copy_order_monitor`

- **业务含义**：  
  这个进程消费“频道告警队列 `dt_channel_alerts`”，针对配置了 `copy_type = signal_copy` 的任务，根据告警内容和 Token 规则生成跟单信号。
- **代码位置**：  
  - 文件：`apps/debot_monitor_center/signal_copy_order_monitor.go`
  - 核心流程：
    - 队列消费工厂：  
      - `NewConsumeRedisQueueFactory(name="signal_copy_order_monitor", QueueName="dt_channel_alerts", QueueMessageSink=mongoSink, QueueTimeout=5s)`
      - sink 仍为 `RedisMongoSink("dt_copy_order_signals")`。
    - 配置加载：
      - `copyOrderConfigLoader := NewCopyOrderConfigLoader(notificationLimiter, CopyTypeSignal, "")`
      - `copyOrderConfigLoader.Start(ctx)` 加载所有 `signal_copy` 任务。
    - 注册处理器：
      - `NewCopyOrderDecodeChannelAlertProcessor(copyOrderConfigLoader)`
      - `NewCopyOrderCheckSignalTokenProcessor()`
    - 启动：`factory.Start(ctx)`。

#### 3.2 处理器一：`CopyOrderDecodeChannelAlertProcessor`

- **业务含义**：  
  把频道告警（`NotificationRecord`）转成内部统一事件 `WalletGroupSwapEvent`，并为所有“同链上的信号跟单策略”生成 `CopyOrderSignalWithStrategy`（带策略上下文的信号）。
- **代码位置**：  
  - 文件：`apps/debot_monitor_center/processors/copy_order_decode_channel_alert_processor.go`
  - 核心逻辑：
    - `ProcessMessage(message interface{})`：
      - 将原始消息（JSON 字符串或 `[]*NotificationRecord`）解析为 `[]*NotificationRecord`。
      - 对每条记录：
        - 过滤无效或过期（一般超过 60s 的告警）。
        - 调 `parseWalletGroupSwapEvent(record.MonitorData.RecordData)`：
          - 先用中间结构处理科学计数法，再转为 `events.WalletGroupSwapEvent`。
        - 调 `convertToCopyOrderSignalWithStrategies(record, event)`：
          - 调 `copyOrderConfigLoader.GetCopyOrderRules(event.Chain)` 获取该链所有 `signal_copy` 策略；
          - 先用事件构造一个 `baseSignal`：
            - `CopyType = CopyTypeSignal`；
            - `SignalType` 根据 `MonitorType` 是 group_all_buy / group_all_sell 判断；
            - 补充 Token 基础数据（市值、holders、tag、创建时间等）。
          - 为每个策略：
            - 复制 `baseSignal`，填充：`SignalID`, `CopyTaskID`, `CopyTaskName`, `UserID`, `DtUserID`；
            - 封装为 `CopyOrderSignalWithStrategy{Strategy, Signal}`。
      - 输出：`[]*CopyOrderSignalWithStrategy`。

#### 3.3 处理器二：`CopyOrderCheckSignalTokenProcessor`

- **业务含义**：  
  基于 Token 维度的各种规则（市值、持仓人数、创建时长、平台/协议标签、黑名单、信号频率、运行时间等），为每个 `CopyOrderSignalWithStrategy` 做最终过滤与补充，再落到 `dt_copy_order_signals`。
- **代码位置**：  
  - 文件：`apps/debot_monitor_center/processors/copy_order_check_signal_token_processor.go`
  - 核心逻辑：
    - `ProcessMessage(message interface{})`：
      - 输入：`[]*CopyOrderSignalWithStrategy`；
      - 遍历每个：
        - 调 `rules.CheckSignalToken(strategy, signal)`：
          - 利用 `CopyOrderSignal` 上多个 getter（`GetMktCap`, `GetHolders`, `GetTokenCreatedAt`, `GetPlatform`, `GetTop10Position`, `GetCreatorAddress` 等）；
          - 执行各类配置规则。
        - 若不通过：
          - `signal.Status = SignalStatusSkipped`；
          - 根据规则类型决定是否记录到 `SkipDetail`；
          - 清空 `SignalDetail` 里与 Token 频率相关字段。
        - 若通过：
          - 如果有 `TokenSignalFrequencyEvent`，则将其挂在 `signal.SignalDetail.SignalList`，方便前端查看“近期信号频率”。
      - 将所有（通过+不通过）的 `signal` 放入 `KafkaMessageGroup.Messages`，交给 `RedisMongoSink("dt_copy_order_signals")`。

- **整体链路角色**：
  - 上游：`dt_channel_alerts`（各种策略产生的频道告警）。
  - 中游：
    - `CopyOrderDecodeChannelAlertProcessor`：按链匹配所有 `signal_copy` 策略；
    - `CopyOrderCheckSignalTokenProcessor`：基于 Token 属性做最终过滤。
  - 下游：`dt_copy_order_signals`（被执行系统消费）。

---

### 4. 跟单执行结果 & 收益结算通知

> 重要：**实际下单/成交逻辑不在本仓库**，这里只有“结果回写 + 通知”。

#### 4.1 执行结果模型：`CopyOrderTransaction` + `TradeResult`

- **业务含义**：  
  一次跟单交易由执行系统生成 `TradeResult`，其中 `Task` 字段具体是某个任务类型的结构体（这里是 `CopyOrderTransaction`），包含交易详情、盈亏等，用于通知和前端展示。
- **代码位置**：  
  - 文件：`models/notifications.go`
  - 结构体：
    - `type CopyOrderTransaction struct { ... }`
      - `CopyOrderTaskID`：对应上游 `CopyOrder.ID`；
      - `UserID`, `DtUserID`, `SignalID`, `TradingTaskID`；
      - `Op`：交易操作类型（跟单目前主要是买入）；
      - `Target`：`Target{Chain, Token, Decimal, Symbol, Icon}`；
      - `TradingParams`：gas、滑点、AntiMev 等；
      - `TradingRecord`：`StatusCode`, `TxHash`, `Amount`, `Price`, `Volume`, `BaseTokenAmount`, `Profit` 等。
    - `type TradeResult struct { TaskType, Task json.RawMessage, TxHash, StatusCode, BaseTokenAmount, TokenAmount, BaseTokenSymbol, ErrInfo, ExtraParams }`
      - `Task` 中存的是 JSON 形式的 `CopyOrderTransaction`。

#### 4.2 系统消息流水线：`SystemEventNotifyProcessor`

- **业务含义**：  
  监听系统事件，当发现 `MonitorTypeCopyOrderResult`（跟单结果）时，构建各渠道通知（app、web、TG 等），并通过模版生成文案。
- **代码位置**：
  - 监控类型定义：  
    - 文件：`apps/debot_monitor_center/core/user_configs/types.go`  
      - `MonitorTypeCopyOrderResult string = "copy_order_result"`
  - 处理器：
    - 文件：`apps/debot_monitor_center/processors/system_event_notify_processor.go`
    - 核心逻辑：
      - `ProcessMessage(message interface{})`：
        - 只处理 `notificationRecord.Meta.EventType == NotifyEventTypeSystemEvent`；
        - 分支 `MonitorTypeCopyOrderResult`：
          1. 解析 `NotifyRecord := TradeResult`（`RecordData`）；
          2. 再解析 `NotifyRecord.Task` → `CopyOrderTransaction`；
          3. 加载用户的告警配置（语言/渠道）；
          4. 为每个通知目标构建 `NotificationItem`，调用模板：
             - `templates.GetRender()(meta, notifyRecord, MonitorTypeCopyOrderResult, notificationItem, userConfig, version)`；
             - 实际会走到 `copy_order_result_app` 模版；
          5. 调 `p.notify` 分发到对应 notifier（app/web/tg 等）。

#### 4.3 app 文案模版 & WebSocket/Redis 推送

- **app 推送模版**：
  - 文件：`apps/debot_monitor_center/core/templates/copy_order_result_templates.go`
  - 逻辑：
    - `init()` 注册模版 key：`"copy_order_result_app"`；
    - `renderCopyOrderResult`：
      - 解析 `tradingResult.Task` → `CopyOrderTransaction`；
      - 组装 `CopyOrderResultRenderData`（操作文案、代币符号、基础代币金额、token 数量、错误信息等）；
      - 加载模版文件：
        - body：`monitor/auto_trading_result/<notificationType>_<lang>_body.tpl`
        - title：`monitor/copy_order_result/<lang>_title.tpl`；
      - 返回标题+正文。
- **Redis/Websocket 推送**：
  - 文件：`apps/debot_monitor_center/core/notifications/redis_queue/redis_notification.go`
  - 结构体：`RedisNotifier`，发布到 Redis `user_event_channel`；
  - `Notify(message interface{})`：
    - 对 `MonitorTypeCopyOrderResult`：
      1. 解析 `TradeResult` + `CopyOrderTransaction`；
      2. 构建 `AutoTradingResultEvent{Op, Chain, TaskID, StatusCode, TxHash, Wallet, BaseTokenSymbol, BaseTokenAmount, TokenAddress, TokenSymbol, TokenAmount, ErrCode}`；
      3. 封装到 `SystemEvent{UserID, EventType="copy_order_result", Data=AutoTradingResultEvent}`；
      4. `Publish("user_event_channel", json)` → WebSocket 层消费并推给前端。

---

### 5. “撤单 / 关闭跟单任务”说明

- **撤单**：
  - 本仓库中没有看到“单笔跟单撤单”的服务端流程；`CopyOrderTransaction` 目前主要描述买入结果，卖出（止盈/止损）更多在通用自动交易模块中实现，copy order 的卖出规则逻辑不在这里。
- **关闭任务**：
  - 实际操作是修改 Mongo `dt_copy_order_task` 中该任务的 `status = "stopped"`；
  - `CopyOrderConfigLoader` 在下一轮 `increaseLoadUserConfigs` 中不再加载该任务，从而后续 swap/告警不会再触发对应的跟单信号。

