# 下单撮合全链路（源码视角）

本文从源码调用路径出发，说明一次 `PLACE_ORDER` 从提交到回调的完整链路。

## 1. 总体时序

`ApiPlaceOrder`  
-> `ExchangeApi` 翻译成 `OrderCommand` 并写入 RingBuffer  
-> `GroupingProcessor` 分组与服务标记  
-> `RiskEngine.preProcessCommand`（R1 预风控）  
-> `MatchingEngineRouter.processOrder`（按 symbol 路由）  
-> `IOrderBook.newOrder`（订单簿撮合）  
-> `RiskEngine.handlerRiskRelease`（R2 清算释放）  
-> `ResultsHandler` / `SimpleEventsProcessor` 事件派发  
-> `ExchangeApi.processResult` 完成异步 Future

---

## 2. 命令入场：`ExchangeApi`

入口方法通常是：

- `submitCommand(ApiCommand)`
- `submitCommandAsync(ApiCommand)`
- `submitCommandAsyncFullResponse(ApiCommand)`

当命令为 `ApiPlaceOrder` 时，`NEW_ORDER_TRANSLATOR` 会写入：

- `command = PLACE_ORDER`
- `price / reserveBidPrice / size / orderId`
- `uid / symbol / action / orderType`
- `resultCode = NEW`

如果是异步接口，`ExchangeApi` 会把 `seq -> promise` 放入 `promises`，在末端阶段按序号回填结果。

---

## 3. 批次编排：`GroupingProcessor`

该阶段不做业务风控，职责是：

- 生成 `eventsGroup`（供 R1/R2 两阶段协同）
- 按周期设置 `serviceFlags`（驱动后续 L2 快照输出）
- 清理旧的 `matcherEvent`/`marketData`，避免脏数据串包

`eventsGroup` 是后续两阶段处理能按组推进的关键。

---

## 4. R1 预风控：`RiskEngine.preProcessCommand`

`PLACE_ORDER` 分支会进入 `placeOrderRiskCheck`：

1. **按 uid 分片**
   - 仅命中 `uidForThisHandler(uid)` 的 risk shard 执行。
2. **基础校验**
   - 用户是否存在（`UserProfileService`）
   - symbol 是否存在（`SymbolSpecificationProvider`）
3. **资金/保证金校验**
   - 现货：`placeExchangeOrder`
     - BID 冻结 quote（基于 `reserveBidPrice` 与 fee）
     - ASK 冻结 base
   - 合约：`canPlaceMarginOrder` + `pendingHold`
4. **写回结果码**
   - 通过：`VALID_FOR_MATCHING_ENGINE`
   - 拒绝：`RISK_NSF` / `AUTH_INVALID_USER` / `INVALID_SYMBOL` 等

撮合层只会真实执行 `VALID_FOR_MATCHING_ENGINE` 的下单。

---

## 5. 撮合路由：`MatchingEngineRouter`

`processOrder` 对交易命令按 symbol 分片：

- 若未命中当前 shard，直接跳过（由对应 shard 处理）
- 命中则进入 `processMatchingCommand`

在 `processMatchingCommand` 中：

1. 查找 symbol 对应 `IOrderBook`
2. 调用 `IOrderBook.processCommand(orderBook, cmd)`
3. 成功时可附带 `cmd.marketData`（取决于 `cfgSendL2ForEveryCmd` 或 `serviceFlags`）

如果订单簿不存在，返回 `MATCHING_INVALID_ORDER_BOOK_ID`。

---

## 6. 订单簿执行：`OrderBookDirectImpl`

`newOrder` 按订单类型分支：

- `GTC` -> `newOrderPlaceGtc`
  - 先 `tryMatchInstantly` 吃对手盘
  - 剩余量挂簿
  - 重复 `orderId` 对未成交部分发 `REJECT`
- `IOC` -> `newOrderMatchIoc`
  - 只即时撮合，剩余量 `REJECT`
- `FOK_BUDGET` -> `newOrderMatchFokBudget`
  - 先预算检查（`checkBudgetToFill`）
  - 满足预算才撮合，否则整单 `REJECT`

`tryMatchInstantly` 在成交过程中构造 `MatcherTradeEvent` 链，并挂在 `cmd.matcherEvent`：

- `TRADE`：成交明细
- `REJECT`：未成交拒单
- `REDUCE`：减仓/撤单产生的释放事件

---

## 7. R2 清算释放：`RiskEngine.handlerRiskRelease`

R2 消费 `cmd.matcherEvent` 与 `cmd.marketData`，做真实账务落地：

- 现货成交：
  - taker/maker 资产划转
  - BID 释放多冻结资金（按 `bidderHoldPrice` 与成交价差）
  - 累计 maker/taker fee
- 合约成交：
  - 更新持仓与挂单冻结
  - 结算手续费
- `REJECT/REDUCE`：
  - 释放冻结资金
- `marketData`：
  - 刷新 `lastPriceCache`（保证金估值依赖）

该阶段确保撮合结果与账户状态一致。

---

## 8. 结果回传：`ResultsHandler` 与事件处理

末端由 `ResultsHandler` 调用结果消费者（常见为 `SimpleEventsProcessor`）：

- `commandResult`：命令级结果码
- `tradeEvent/reduceEvent/rejectEvent`：业务事件
- `orderBook`：L2 快照事件

同时 `ExchangeApi.processResult(seq, cmd)` 会命中 `promises`，完成调用方 `CompletableFuture`。

---

## 9. 一致性与确定性保证

- **先风控冻结，后撮合，最后清算释放**（R1 -> ME -> R2）
- **单事件流顺序执行**（Disruptor RingBuffer 序列）
- **分片并行 + 组边界协同**（uid/symbol 分片 + `eventsGroup`）

因此系统既能保持高吞吐，又能保证账务与成交状态的确定性。

---

## 10. Dump（状态持久化）链路

有，当前仓库包含明确的 dump 逻辑，对应 `ApiPersistState` 命令。

### 10.1 触发入口

通过 `ExchangeApi` 提交 `ApiPersistState` 时，会走 `publishPersistCmd`，一次性发布两个内部命令：

1. `PERSIST_STATE_MATCHING`
2. `PERSIST_STATE_RISK`

两条命令使用相邻序号发布，确保先后顺序固定。

### 10.2 执行顺序与语义

- 第一步（撮合侧）  
  `MatchingEngineRouter.processOrder` 收到 `PERSIST_STATE_MATCHING` 后调用 `serializationProcessor.storeData(...)` 落盘撮合分片状态。  
  成功时结果码写 `ACCEPTED`（表示第一阶段已受理）。

- 第二步（风控侧）  
  `RiskEngine.preProcessCommand` 在 `PERSIST_STATE_RISK` 分支调用 `serializationProcessor.storeData(...)` 落盘风控分片状态。  
  成功时返回 `SUCCESS`，失败返回 `STATE_PERSIST_RISK_ENGINE_FAILED`。

这两个阶段组合后，形成一次完整 dump（matching + risk）。

### 10.3 调用方返回结果

`ExchangeApi.submitPersistCommandAsync` 会分别等待两条命令结果，再通过 `CommandResultCode.mergeToFirstFailed` 合并：

- 两边都成功 -> 成功
- 任一失败 -> 返回首个失败码

### 10.4 与下单链路的关系

dump 不会替代下单撮合主链路，而是插入同一事件流中的“状态快照命令”：

- 与普通订单共享 RingBuffer 顺序语义
- 依赖 pipeline 顺序保证快照一致性
- 用于故障恢复时的基线状态

### 10.5 恢复路径（补充）

`ExchangeCore.startup()` 会调用 `serializationProcessor.replayJournalFullAndThenEnableJouraling(...)`，即基于 snapshot + journal 回放恢复状态，再继续处理新命令。

---

## 11. 实际运行中：下单与持久化如何配合

线上建议把下单与 dump 当作同一事件流里的两类命令来使用。

### 11.1 常见配合模式

- 模式 A：持续下单 + 周期性 dump  
  交易持续进行，定时插入 `ApiPersistState`，获取某个序列点的一致快照。
- 模式 B：关键运维点 dump  
  先等待一批关键下单完成，再发 `ApiPersistState` 并等待返回，作为“结算点/切换点”。
- 模式 C：恢复优先  
  低频 dump + 高频 journal，重启时用 `snapshot + replay` 平衡恢复速度与存储成本。

### 11.2 为什么不会和下单“打架”

- `ApiPersistState` 被拆成相邻两条命令（matching/risk），仍按 RingBuffer 全序执行。
- 下单与 dump 共用同一顺序语义，不会出现“快照先于已提交订单落地”的乱序问题。
- 调用侧应等待 `submitPersistCommandAsync()` 结果，再做“已安全持久化”的后续动作。

---

## 12. 撮合、冻结、扣款如何保证强一致

### 12.1 资金语义（先冻结、后清算）

- R1：`PLACE_ORDER` 先做风控与冻结，失败不入撮合。
- ME：只撮合 `VALID_FOR_MATCHING_ENGINE` 的订单。
- R2：根据成交/拒单事件做扣款、划转、释放冻结和手续费入账。

这保证了不会出现“未冻结先成交”或“成交后未释放”的状态撕裂。

### 12.2 一致性机制

- 单 RingBuffer 顺序执行（命令全序）。
- `eventsGroup` + TwoStep（R1 主处理器触发组边界后，R2 对同组收敛处理）。
- 撮合结果通过 `cmd.matcherEvent` 传递给 R2，同命令内闭环清算。
- 结果回调在链路末端发出，外部可见状态与内部序列一致。

---

## 13. 关键源码锚点（可直接对照）

### 13.1 下单与回调

- `src/main/java/exchange/core2/core/ExchangeApi.java`
  - `NEW_ORDER_TRANSLATOR`
  - `submitCommandAsync(...)`
  - `processResult(...)`

```java
// ExchangeApi.NEW_ORDER_TRANSLATOR
cmd.command = OrderCommandType.PLACE_ORDER;
cmd.price = api.price;
cmd.reserveBidPrice = api.reservePrice;
cmd.size = api.size;
cmd.orderId = api.orderId;
cmd.uid = api.uid;
cmd.symbol = api.symbol;
cmd.resultCode = CommandResultCode.NEW;
```

### 13.2 R1 风控冻结

- `src/main/java/exchange/core2/core/processors/RiskEngine.java`
  - `preProcessCommand(...)`
  - `placeOrderRiskCheck(...)`
  - `placeExchangeOrder(...)`

```java
case PLACE_ORDER:
    if (uidForThisHandler(cmd.uid)) {
        cmd.resultCode = placeOrderRiskCheck(cmd);
    }
    return false;
```

### 13.3 只撮合通过风控的订单

- `src/main/java/exchange/core2/core/orderbook/IOrderBook.java`
  - `processCommand(...)`

```java
} else if (commandType == OrderCommandType.PLACE_ORDER) {
    if (cmd.resultCode == CommandResultCode.VALID_FOR_MATCHING_ENGINE) {
        orderBook.newOrder(cmd);
        return CommandResultCode.SUCCESS;
    } else {
        return cmd.resultCode;
    }
}
```

### 13.4 R2 清算扣款/释放

- `src/main/java/exchange/core2/core/processors/RiskEngine.java`
  - `handlerRiskRelease(...)`
  - `handleMatcherEventsExchangeBuy(...)`
  - `handleMatcherEventsExchangeSell(...)`
  - `handleMatcherRejectReduceEventExchange(...)`

```java
if (mte.eventType == MatcherEventType.REDUCE || mte.eventType == MatcherEventType.REJECT) {
    handleMatcherRejectReduceEventExchange(cmd, mte, spec, takerSell, takerUp);
}
```

### 13.5 Dump 两阶段持久化

- `src/main/java/exchange/core2/core/ExchangeApi.java`
  - `publishPersistCmd(...)`
  - `submitPersistCommandAsync(...)`
- `src/main/java/exchange/core2/core/processors/MatchingEngineRouter.java`
  - `PERSIST_STATE_MATCHING` 分支
- `src/main/java/exchange/core2/core/processors/RiskEngine.java`
  - `PERSIST_STATE_RISK` 分支

```java
cmdMatching.command = OrderCommandType.PERSIST_STATE_MATCHING;
cmdRisk.command = OrderCommandType.PERSIST_STATE_RISK;
```
