## 架构
* 基于责任链思想实现的顺序处理流水线
* 把所有处理步骤抽象成 Processor 接口，按注册顺序组成一条链，上一步的输出就是下一步的输入
* 某一步返回 nil 就会提前终止后续处理
* 这个结构借鉴了责任链模式，但更多是用来做数据处理流水线而不是纯粹的“谁负责处理请求”的场景

## 重要Processor
从你现在的理解深度来看，要把这个项目的**主要业务场景都吃透**，重点把下面这几条“主链路”里的 Processor 搞清楚就够了（按重要性和典型性排序）：

### 一、EVM 区块处理链（链上原始数据 → 结构化行为 + DEX 特征）

文件：`tasks/evm_block_process/main.go`

重点 Processor：

1. **`BlockTxDecodeProcessor`**  
   - 作用：把原始区块/交易日志解码成项目内部通用结构（包含 tx、logs、actions 的骨架）。  
   - 场景：所有后续 DEX 解析、规则、监控都基于它产生的结构。

2. **`DexPoolCreateProcessor` / `DexPoolEventProcessor`**  
   - 作用：识别/维护 DEX 池子（pair）的元数据与储备变化。  
   - 场景：保证后续 DEX 价格、流动性等解析有正确的 pair 信息和储备。

3. **`DexProcessor`**（极重要）  
   - 作用：  
     - 从交易 logs 中识别各种 Swap（V2/V3/V4、four.meme、flap 等）；  
     - 计算价格、成交量、储备；  
     - 识别套利机器人、三明治攻击；  
     - 产出 `SwapRecord` 和丰富的 `TransactionWithActions`。  
   - 场景：理解“**我们是如何从原始链上数据得出价格 / 成交 / 攻击特征**”的关键。

4. **`WriteBlockDataToKafka` / `WriteActionsToKafka`**  
   - 作用：把结构化后的区块+行为写回 Kafka，作为后续监控和统计的上游数据源。  
   - 场景：理解“分析链路的出口”。


### 二、钱包与交易监控链（行为 → 规则 → 告警）

文件：`apps/debot_monitor_center/wallet_tx_monitor.go`

重点 Processor：

1. **`DataDecodeProcessor`**  
   - 作用：从 Kafka 读到的数据还原成 `TransactionWithActions`（包含我们在上游构建的各种行为标记）。  
   - 场景：监控链路的入口，等价于“还原一笔 tx 上所有行为”。

2. **`PreDataProcessor`**（很重要）  
   - 作用：  
     - 找出这一笔 tx 里“在用户监控列表中的钱包/代币”；  
     - 为每个监控钱包/代币生成独立的统计（买卖次数、净持仓、成交额等）。  
   - 场景：把“交易级别”数据投影到“**用户关心的监控维度**”。

3. **`WalletMonitorRulesCheckProcessor`**（极重要）  
   - 作用：  
     - 按用户配置的策略规则（价格、市值、黑白名单、频率、运行时段等）检查监控结果；  
     - 结合 `NotificationLimiter` 做限流；  
     - 构造 `NotificationRecord` 并写 Kafka/Redis。  
   - 场景：理解“**规则引擎 + 告警生成**”。

### 三、Swap 记录监控与钱包组跟单（Swap → Maker 流量聚合 → 规则 / 跟单）

文件：  
- `apps/debot_monitor_center/swap_records_monitor.go`  
- `apps/debot_monitor_center/wallet_group_copy_order_monitor.go`  

重点 Processor（两个场景共用核心）：

1. **`DecodeBlockTxProcessor` / `DecodeSwapTxProcessor` / `FilterBlockTxProcessor` / `FilterSwapTxProcessor`**  
   - 作用：把 Kafka 的区块数据、Redis 的 swap 数据解码并去重。  
   - 场景：理解“上游 swap 数据是怎么以统一格式进入 merge 链路的”。

2. **`MergeSwapRecordsProcessor`**（极重要，你已经在看）  
   - 作用：  
     - 把区块数据 + swap 流合并；  
     - 只保留监控钱包、且在 `BalanceChanges` 中有净流入/流出的 Popular 代币；  
     - 按 maker + token 聚合成 **净流入/流出 + 金额 + 平台币计价**；  
     - 产出 `MergedSwapResult` 和 `SingleSwapResult`。  
   - 场景：讲清“**我们如何从多条 swap 聚合出一个地址这一笔的净买入/净卖出**”。

3. **`SwapRecordsMonitorRulesCheckProcessor`**  
   - 作用：在 `SingleSwapResult` 维度上跑一套规则（类似钱包监控，但针对 swap 聚合结果），产出告警。

4. **`SwapRecordGroupCopyOrderProcessor`**（在钱包组跟单场景）  
   - 作用：基于钱包组配置和 `SingleSwapResult`，产出跟单信号（写入 Mongo/Redis），驱动复制交易。  
   - 场景：和校招生聊“**跟单/复制交易如何触发**”时很好讲。


### 四、规则引擎与限流（支撑上面所有监控/跟单场景）

文件：`apps/debot_monitor_center/core/rules/*.go`、`apps/debot_monitor_center/core/limit/notification_limit.go`

重点组件（不是 Processor 接口，但你必须理解）：

1. **`RulesChecker` & `MonitorRules`（`rule.go`）**  
   - 作用：  
     - 把策略配置解析为一组规则（普通/Build/Heavy/Limit）；  
     - 定义规则执行顺序（先普通再 Heavy 再 Limit）；  
     - 提供 `Check` / `CheckToBuildMonitorEvent` / `CheckMergedSwapResult` 等接口给 Processor 调用。  

2. **典型规则：`rule_price.go`、`rule_protocol_filter.go` 等**  
   - 作用：具体实现价格阈值、内外盘过滤、市值过滤等，掌握 2–3 个代表就够。

3. **`NotificationLimiter`（`notification_limit.go`）**  
   - 作用：按 user + 时间窗口统计通知次数，限制噪声。  
   - 场景：理解“**如何防止某个策略/用户被刷屏**”。

### 五、辅助但值得扫一眼的 Processor

- **`MonitorEventNotifyProcessor` / `SystemEventNotifyProcessor`**（`notify_center.go`）：告警下游真正推送前的处理。  
- **`DataDecodeProcessor` 的测试 / Merge 测试**：帮助你理解输入/输出结构。

这个顺序看 Processor：

1. **`tasks/evm_block_process/main.go`** 中注册的 Processor（重点看 `BlockTxDecodeProcessor` + `DexProcessor`）。  
2. **`apps/debot_monitor_center/wallet_tx_monitor.go`** 中 4 个 Processor（DataDecode → PreData → RulesCheck）。  
3. **`apps/debot_monitor_center/swap_records_monitor.go` / `wallet_group_copy_order_monitor.go`** 中的 Merge 链路（Decode* / Filter* / MergeSwapRecords / Monitor or CopyOrder）。  
4. **`apps/debot_monitor_center/core/rules/rule.go` + 1–2 个具体规则文件 + `notification_limit.go`**（理解规则引擎+限流）。  


## RuleChecker
把一堆规则组织成“可执行规则引擎”
1. 目标
把“用户在配置里勾选的一堆条件”转成一串 MonitorRule 对象。
统一管理执行顺序：
普通规则（轻量过滤）
Build 规则（真正生成事件）
Heavy 规则（耗时的额外过滤）
Limit 规则（次数/频率限制）
2. 关键结构
```
type MonitorRule interface {
    GetRuleName() string
    CheckAndBuildEvent(...)
    Check(...)
    PreHandle(...)
    CheckMergedSwapResult(...)
    CheckAndBuildEventForMergedSwapResult(...)
}
type RulesChecker struct {
    Rules          []MonitorRule
    ExistRuleNames map[string]bool
    ruleMap        map[string]MonitorRule
}
type MonitorRules struct {
    UserId, MonitorConfigId int64
    MonitorConfigName       string
    MonitorType             string
    MonitorObjects          *sync.Map
    RuleChecker             *RulesChecker
    // ...
}
```
还有几张“规则注册表”：
```
var allRules        = map[string]func(params []*models.ConfigParameter) MonitorRule
var allBuildRules   = map[string]func(params []*models.ConfigParameter) MonitorRule
var allHeavyRules   = map[string]func(params []*models.ConfigParameter) MonitorRule
var limitRules      = map[string]func(params []*models.ConfigParameter) MonitorRule
```
3. 构造规则链：NewRulesChecker
输入：策略配置字符串 configs + 可能的附加参数 additions。
流程：
DecodeMonitorStrategyConfigs 把配置解析成 []MonitorStrategyConfig。
每个 config 根据 ConfigType 去 allRules / allBuildRules / allHeavyRules 找对应的 ruleBuilder。
调用 ruleBuilder 生成 MonitorRule，放进 Rules 列表，并在 ExistRuleNames 里标记。
再单独扫一次 config，生成 limitRules（次数类规则），保证它们被 append 到链的最后。

## NotificationLimiter

1. 目标
决定“同一个用户在最近 N 分钟内还能不能再收一条通知”。。
避免单个用户配置太“宽”导致被刷屏或打爆推送通道。

2. 关键结构
```
type NotificationLimiter struct {
    limitMap          sync.Map // userId -> *sync.Map[min -> count]
    limitTimeInterval int64    // 滚动时间窗，单位：分钟
}
limitMap：存最近一段时间内每个分钟的通知数量，用来计算“最近 N 分钟的总通知数”（滚动时间窗）
key =  60 秒对齐的时间戳，例如：2026-03-12 10:01:xx → minute=10:01:00
value = 这一分钟内发出去的通知条数：minute → count, minute = 按分钟对齐的时间戳,  count = 该分钟发生了多少条通知
```

3. 周期清理：autoClearOutData
构造时会启动一个 goroutine：

```
ticker := time.NewTicker(2 * time.Second)
for range ticker.C {
    limiter.autoClearOutData()
}
autoClearOutData 逻辑：
```

计算 “3 小时前”的 unix 时间 unixTime。
遍历每个 userId 的 minuteMap：
删掉 minute < unixTime 的条目（即 3 小时之前的数据）。
如果某个 user 的 minuteMap 被清空了，就从 limitMap 里删掉这个 userId。
目的：控制内存占用，防止长时间运行后 user → minute 数组

4. 如何限频？

```
func (n *NotificationLimiter) CheckLimit(userId int, unixTime int64) bool {
    used := n.GetUsed(userId, unixTime)
    limit := n.GetLimit(userId)
    return used <= int64(limit)
}
```
这里的GetUsed：
找到 userId 对应的 minuteMap；
计算从 startTime = unixTime - limitTimeInterval*60 到 unixTime 之间所有 minute 的 count 总和；
返回这个总和 = 最近 N 分钟内已经用掉的通知数

如果CheckLimit返回 false：说明最近 N 分钟发送的通知数已经超过该用户限额，当前事件应丢弃或仅做标记，不再发通知

5. Example

以 WalletMonitorRulesCheckProcessor 为例（钱包行为监控）：
* Processor 调 strategy.RuleChecker.Check(...) / CheckToBuildMonitorEvent(...) → 判断这次行为是否应该触发策略。
* 在准备写 NotificationRecord 之前，调用：
```
if !notificationLimiter.CheckLimit(int(strategy.UserId), unixTime) {
    // 超限，打日志 + Mark，然后 return，不发这条通知
}
```
只有 规则通过 + 未超限 才真正构造 NotificationRecord 并写 Kafka/Redis；然后再：

`notificationLimiter.AddNotification(int(strategy.UserId), unixTime)`