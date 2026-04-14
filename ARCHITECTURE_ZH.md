# Exchange Core 系统架构与核心功能点

## 1. 项目定位

`exchange-core` 是一个面向高性能撮合场景的交易所核心引擎库，目标是以低延迟和高吞吐完成：

- 订单撮合（Matching Engine）
- 风控与账户账务（Risk + Accounting）
- 事件持久化（Journal）与快照恢复（Snapshot）
- 交易/管理/报表统一命令接口（API）

该项目本质上是**撮合内核**，不包含 REST/FIX 网关、用户鉴权系统、外部结算系统等外围应用层。

---

## 2. 总体架构分层

从代码职责看，系统可分为以下层次：

1. **接入层（Command API）**
   - 核心类：`ExchangeApi`
   - 负责将外部调用转换为 `OrderCommand` 写入 Disruptor RingBuffer。

2. **事件驱动核心总线（Disruptor Pipeline）**
   - 核心类：`ExchangeCore`
   - 负责构建多阶段流水线：分组 -> 风控预处理 -> 撮合 -> 风控后处理 -> 结果回调。

3. **撮合层（Matching）**
   - 核心类：`MatchingEngineRouter`、`IOrderBook`、`OrderBookDirectImpl`/`OrderBookNaiveImpl`
   - 负责订单簿维护、成交、撤单、改价、L2 快照生成。

4. **风控账务层（Risk & Accounting）**
   - 核心类：`RiskEngine`、`UserProfileService`
   - 负责下单前资金冻结校验、成交后余额/仓位释放与费用记账。

5. **持久化层（Event Sourcing）**
   - 核心类：`DiskSerializationProcessor`
   - 负责命令日志（journal）写入、状态快照、启动回放恢复。

6. **配置与可观测性层**
   - 核心类：`ExchangeConfiguration`、`PerformanceConfiguration`、`OrdersProcessingConfiguration`
   - 负责性能参数、风险模式、日志级别、序列化策略等配置。

---

## 3. 核心处理链路（命令生命周期）

以“下单”为例，主流程如下：

1. 客户端通过 `ExchangeApi.submitCommand*()` 提交 `ApiPlaceOrder`。
2. API 将命令翻译为 `OrderCommand` 并发布到 RingBuffer。
3. `GroupingProcessor` 做批次边界与分组控制。
4. `RiskEngine.preProcessCommand()` 执行预风控：
   - 校验用户、交易对、余额/保证金；
   - 通过则标记 `VALID_FOR_MATCHING_ENGINE`。
5. `MatchingEngineRouter.processOrder()` 路由到对应分片订单簿执行撮合。
6. `RiskEngine.handlerRiskRelease()` 根据成交/拒单事件做资金释放、手续费入账、仓位更新。
7. `ResultsHandler` + `ExchangeApi.processResult()` 完成异步回调（`CompletableFuture`）。
8. 若开启 journaling，命令在链路中并行写盘；重启时可回放恢复状态。

该流程保证了“风控-撮合-清算”在单一事件流中的顺序一致性和可重复性。

---

## 4. 关键设计特性

## 4.1 高性能并发模型

- 基于 **LMAX Disruptor** 单 RingBuffer 事件流。
- 采用多阶段流水线 + 分片（`matchingEnginesNum` / `riskEnginesNum`）。
- 通过 `uid`、`symbol` 掩码分片，将账户与订单簿负载分散到多个引擎实例。
- 支持 `BUSY_SPIN`、线程亲和等低延迟配置。

## 4.2 撮合引擎实现

- 抽象接口：`IOrderBook`。
- 实现：
  - `OrderBookDirectImpl`：性能优先，基于 ART（Adaptive Radix Tree）和对象池。
  - `OrderBookNaiveImpl`：结构更简单，适合基础场景。
- 支持订单类型：`GTC`、`IOC`、`FOK_BUDGET`（并含预算校验逻辑）。

## 4.3 风控与账户模型

- 用户资产与仓位由 `UserProfileService`/`UserProfile` 维护。
- 支持两类交易语义：
  - 现货交易对（`CURRENCY_EXCHANGE_PAIR`）
  - 合约/保证金交易（`FUTURES_CONTRACT`，可按配置开关）
- 风控分两阶段：
  - R1：下单前冻结与准入检查；
  - R2：成交后释放、扣费、仓位/PNL 更新。

## 4.4 事件溯源与恢复

- Journal 记录变更命令（支持压缩分块写入）。
- Snapshot 对撮合与风控分片状态进行序列化。
- 启动时可按 `snapshot + journal replay` 恢复到一致状态。

## 4.5 确定性与数值安全

- 全链路使用整数/长整型表达价格和数量，避免浮点误差。
- 命令顺序与处理路径确定，可复放验证状态一致性。

---

## 5. 核心功能点清单

### 交易相关

- 下单、撤单、改价、减仓（reduce）
- 订单簿查询（L2）
- 异步结果回调与批量同步提交

### 管理相关

- 新增用户
- 用户挂起/恢复
- 余额调整（充值、扣减、冻结/解冻语义）
- 系统重置（`RESET`）

### 批量/二进制命令

- 批量添加交易对（`BatchAddSymbolsCommand`）
- 批量导入账户（`BatchAddAccountsCommand`）
- 二进制数据分片与 LZ4 压缩传输

### 报表与查询

- 报表查询统一入口（`ApiReportQuery` / `ReportQuery`）
- 典型能力：单用户报表、系统总余额、状态哈希等

### 持久化与运维

- 状态持久化（`PERSIST_STATE_MATCHING` + `PERSIST_STATE_RISK`）
- Journal 回放恢复
- 可配置日志级别与性能参数

---

## 6. 主要包结构（按职责）

- `exchange.core2.core`：核心入口（`ExchangeCore`、`ExchangeApi`）
- `exchange.core2.core.processors`：流水线处理器（分组、风控、撮合路由、结果）
- `exchange.core2.core.orderbook`：订单簿接口与实现
- `exchange.core2.core.common`：领域模型（订单、用户、事件、符号定义）
- `exchange.core2.core.common.api`：对外命令模型
- `exchange.core2.core.common.config`：配置模型
- `exchange.core2.core.processors.journaling`：快照与日志持久化
- `exchange.core2.core.utils`：序列化、线程亲和、算术工具等

---

## 7. 配置驱动的扩展能力

通过 `ExchangeConfiguration` 可定制：

- **性能侧**：RingBuffer 大小、撮合/风控分片数、等待策略、订单簿实现
- **业务侧**：是否启用保证金交易、是否关闭风控
- **可靠性侧**：序列化实现、是否启用 journaling
- **观测侧**：日志级别、报表查询配置

这使得该内核可在“极低延迟单机场景”与“多分片高吞吐场景”之间切换。

---

## 8. 架构边界与集成建议

当前仓库聚焦“撮合核心”，建议在上层系统补齐：

- 接入网关（REST/FIX/WebSocket）
- 认证鉴权与风控策略中心（黑白名单、限频、反洗钱等）
- 结算清算、账本对账、资金划转
- 市场数据分发（BBO/Tick/Kline）与监控告警平台

推荐集成方式是：上层服务将业务请求映射为 `ApiCommand`，核心内核仅负责确定性执行与状态管理。

---

## 9. 从源码看下单撮合全链路（PLACE_ORDER）

以下链路基于当前仓库实现（`ExchangeApi` -> `ExchangeCore` pipeline -> `RiskEngine` -> `MatchingEngineRouter` -> `IOrderBook` -> `RiskEngine` -> `ResultsHandler`）。

### 9.1 入口：API 命令翻译并写入 RingBuffer

1. 上层提交 `ApiPlaceOrder`（通常走 `submitCommandAsync` / `submitCommandAsyncFullResponse`）。
2. `ExchangeApi.NEW_ORDER_TRANSLATOR` 将 API 字段映射到 `OrderCommand`：
   - `command=PLACE_ORDER`
   - `price / reserveBidPrice / size / orderId / uid / symbol / action / orderType`
   - `resultCode=NEW`
3. 命令发布到 Disruptor RingBuffer，等待 pipeline 消费。

补充：异步接口会把 `seq -> promise` 放入 `promises`，在末端由 `api.processResult(seq, cmd)` 完成 future 回填。

### 9.2 调度：GroupingProcessor 打批次与服务标记

`GroupingProcessor` 不做业务校验，主要做两件事：

- 给每条命令打 `eventsGroup`，用于两阶段风控（R1/R2）在“组边界”协同推进。
- 周期性打 `serviceFlags`，用于触发撮合后 L2 快照回传（`MatchingEngineRouter` 中读取该标记）。

这一步也会清理上个循环残留的 `matcherEvent`、`marketData` 字段，确保每条命令只携带本次处理结果。

### 9.3 R1 预风控：RiskEngine.preProcessCommand

`PLACE_ORDER` 在 R1 走 `placeOrderRiskCheck`：

1. **分片路由（按 uid）**
   - 只有 `uidForThisHandler(uid)` 命中的 risk shard 执行校验。
2. **基础校验**
   - 用户存在性（`UserProfileService.getUserProfile`）
   - 交易对存在性（`SymbolSpecificationProvider.getSymbolSpecification`）
3. **资金/保证金校验与冻结**
   - 现货：`placeExchangeOrder`
     - BID 冻结 quote（按 `reserveBidPrice` + taker fee）
     - ASK 冻结 base
     - 校验可用余额（含 futures 浮盈浮亏折算）
   - 合约：`canPlaceMarginOrder` + `pendingHold`
4. **结果写回**
   - 通过：`VALID_FOR_MATCHING_ENGINE`
   - 拒绝：如 `RISK_NSF`、`AUTH_INVALID_USER`、`INVALID_SYMBOL` 等

重要语义：撮合层只会真正处理 `resultCode == VALID_FOR_MATCHING_ENGINE` 的下单命令。

### 9.4 撮合路由：MatchingEngineRouter.processOrder

`MatchingEngineRouter` 按 `symbol` 分片（`symbolForThisHandler`），命中后调用 `processMatchingCommand`：

1. 取 `orderBooks.get(symbol)`，不存在则 `MATCHING_INVALID_ORDER_BOOK_ID`
2. 否则执行 `IOrderBook.processCommand(orderBook, cmd)`：
   - 若 `PLACE_ORDER` 且 `cmd.resultCode == VALID_FOR_MATCHING_ENGINE` -> `orderBook.newOrder(cmd)`
   - 否则直接返回 R1 的拒绝码（不会改动订单簿）
3. 若成功并满足开关（`cfgSendL2ForEveryCmd` 或 `serviceFlags & 1`），附带 `cmd.marketData`（L2 快照）

### 9.5 订单簿内部：OrderBookDirectImpl.newOrder

以默认高性能实现 `OrderBookDirectImpl` 为例：

- `GTC`：`newOrderPlaceGtc`
  1. 先 `tryMatchInstantly` 吃对手盘（taker 逻辑）
  2. 若未完全成交，剩余挂入订单簿（price bucket + orderId index）
  3. 若发现重复 `orderId`，对剩余量生成 `REJECT` 事件
- `IOC`：`newOrderMatchIoc`
  - 仅即时撮合，未成交余量直接 `REJECT`
- `FOK_BUDGET`：`newOrderMatchFokBudget`
  - 先 `checkBudgetToFill` 估算全成所需预算，满足才撮合，否则整单 `REJECT`

`tryMatchInstantly` 会逐个 maker 订单成交并生成 `MatcherTradeEvent` 链（`TRADE`/`REJECT`/`REDUCE`），挂在 `cmd.matcherEvent` 上供后续清算与回调消费。

### 9.6 R2 后风控/清算：RiskEngine.handlerRiskRelease

撮合后进入 R2（按 `eventsGroup` 和 TwoStep 主从处理器协同）：

1. 读取 `cmd.matcherEvent` 与 `cmd.marketData`
2. 若有成交事件：
   - 现货：
     - taker/maker 资产划转
     - BID 释放多冻结资金（`bidderHoldPrice - tradePrice`）
     - 计提 maker/taker fee（累加到 `fees`）
   - 合约：
     - 更新持仓（`SymbolPositionRecord`）
     - 释放挂单冻结，结算手续费
3. 若有 `REJECT/REDUCE` 事件：释放对应冻结资金
4. 若带 L2 且启用保证金：刷新 `lastPriceCache`（用于后续保证金估值）

这一步保证了“成交结果”与“账务状态”在同一命令序列内严格一致。

### 9.7 结果出站：ResultsHandler + ExchangeApi 回调

1. `ResultsHandler` 在末端调用 `resultsConsumer`
2. 默认可接 `SimpleEventsProcessor`，拆分三类输出：
   - `commandResult`（命令结果码）
   - `tradeEvent` / `reduceEvent` / `rejectEvent`
   - `orderBook`（L2）
3. 同时 `api.processResult(seq, cmd)` 命中 promise，完成 `CompletableFuture`

因此调用方可同时获得：
- **同步语义**：future resultCode
- **异步事件流语义**：成交明细、拒单、盘口变化

### 9.8 时序要点（简版）

`ApiPlaceOrder` -> `OrderCommand(PLACE_ORDER)` -> `Grouping(eventsGroup)` -> `R1 placeOrderRiskCheck` -> `ME processCommand/newOrder` -> `OrderBook match + matcherEvent` -> `R2 handlerRiskRelease` -> `ResultsHandler/SimpleEventsProcessor` -> `api.processResult -> future complete`

### 9.9 关键一致性保证

- **先冻后撮合后清算**：R1 冻结资金，ME 只处理通过命令，R2 基于成交事件做最终入账/释放。
- **单事件流确定性**：所有状态变更由 RingBuffer 序列驱动，可重放验证。
- **分片但有序**：uid/symbol 分片提升吞吐，`eventsGroup` 保障 R1/R2 组边界一致推进。
