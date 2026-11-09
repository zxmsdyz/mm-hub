# Wick 策略看板规格（P3 急跌接针 + 熔断 60 + 强平门影子）v0，2026-11-09

> 现状：qflow 看板「Maker 价差」页按单元类型分流，做市单元走 `MmBoardView`（状态栏 / 报价面板 / 库存卡 / 为什么不挂 / 事件表 / 日表 / tick 图），价差单元走 KPI 卡 + 收益曲线 + 延迟卡 + 统计图 + 日表 + 回合表 + 引擎控制。wick 的 OpenSpec（`add-bitget-wick-reversion-strategy`）**未含看板**。
> wick 与 mm 的根本差别：mm 是持续双边报价，看板看「现在挂在哪、为什么不挂」；wick 是事件驱动，90% 时间空仓，看板要看「触发了几次、接到几层、回合怎么出的、熔断 / 门挡了多少」。

## 0. 分两级交付

- **T1（Run B 上线前必须有）**：§1 状态栏、§2 回合表、§3 日表与 KPI、§4 事件图。目的是 Run B 期间人能看懂它在干什么、和回测对不对得上。
- **T2（Run B 期间补）**：§5 对表面板、§6 强平门影子面板、§7 告警。

## 1. 复用 / 舍弃（对现有 mm 布局）

| 现有组件 | 处置 | 说明 |
|---|---|---|
| `MmStatusBar`（单元、运行时长、HALT） | **复用**，加 wick 状态 | 状态机当前态 IDLE / ARMED / LADDER / IN_POSITION / COOLDOWN，COOLDOWN 剩余秒数，两向开关 |
| `MmQuotePanel`（双边报价、BN 公允价） | **舍弃** | wick 不持续报价、不用 BN 公允价 |
| `MmInventoryCard`（库存、批次龄、浮亏） | **改造** | 显示当前回合：方向、已成交层数 / 3、成本 C、浮盈亏 bp、持仓时长、当前止盈价、距止损 bp、距到时秒 |
| `MmWhyCard`（为什么不挂） | **改造为「为什么没开回合」** | 最近一次未开的原因：busy / cooldown / 流量门不足 / 方向关闭 / HALT / stale / 安全阀 / 强平门（影子，标记「若生效会挡」） |
| `MmEventsTable` | **替换**为回合表（§2） | mm 事件以报价为单位，wick 以回合为单位 |
| `MmDailyTable` | **复用结构，换列**（§3） | |
| `MmTickChart` | **复用骨架，换叠加层**（§4） | |
| 价差布局的 `MakerPnlCurve`、`MakerLatencyCard`、`MakerEngineControls` | **复用** | 收益曲线、延迟卡、启停控制都通用 |
| 价差布局的 `MakerKpiCards` | 复用框架，换指标（§3） | |

## 2. 回合表（T1）

每行一个回合（`wick_round` + 关联 `wick_trigger / wick_confirm / wick_order_end`）：

| 列 | 来源 |
|---|---|
| 触发时刻、方向、触发时 dd / du（bp）、流量倍数（10 s 同向主动额 / 1 h 中位） | wick_trigger |
| 确认用时（触发 → 确认 ms）、确认价、low / high | wick_confirm |
| 三层各自：挂价、挂出时 queue_ahead、是否成交、成交用时（挂出 → 成交 s）、post_only 拒 / 在途上限拒 | wick_order_end |
| 成交层数、C、止盈重挂次数 | wick_round |
| 出场类型（tp / sl / time / stale / halt / external_flat）、出场价、出场时延（决策 → 成交 ms，taker 时）、滑点 bp | wick_round |
| 回合净 bp、净 USDT、持仓秒 | wick_round |
| 进场 markout 5 / 30 / 300 s、出场后 30 s markout | wick_markout |
| 是否触发 COOLDOWN；强平门影子值（liq_cnt_60 / doi5 / gliq_block） | wick_state / wick_trigger |

点开一行：该回合的秒级价格路径（触发前 60 s 到出场后 60 s）、三层挂价与成交点、止盈价轨迹、止损线。

## 3. KPI 与日表（T1）

KPI 卡（今日 / 近 7 日 / 自 Run B 起）：

| 指标 | 回测对照值（lot 0.01，样本外 7 天） |
|---|---|
| 回合 / 天；有成交 episode 占比（成交率） | 23~25；82% |
| 每回合净 bp（含费）、净 USDT | +3.2；+5.3/天 |
| 出场构成：止盈 / 止损 / 到时 / 其他（张数占比） | 93 / 5~6 / 1 / — |
| 止损类次数 / 天；COOLDOWN 次数与占用时长占比 | 1.5 |
| 按方向：回合、净、止损（多 / 空分列） | 多 +127 vs 空 +11（35 天） |
| 按层数：1 / 2 / 3 层回合数与净 | 三层回合是亏损源 |
| 最大回撤（逐回合累计）、正日数 | 13.8；7/7 |
| 触发 / 天、被丢弃触发及原因分布（busy / cooldown / 门 / HALT） | 约 35 |
| 止盈单被吃速度：部分成交 → 全部成交用时分位 | 回测看不到，Run B 关键观测 |
| taker 出场时延分位、滑点分位 | 回测 4 ms / 0 |

日表：每 UTC 日一行，上面指标的日值 + 当日 BTC 涨跌 + 当日每小时 σ 中位（判断是不是「单边日」）。

## 4. 事件图（T1）

复用 tick 图骨架：Bitget mid（1 s），叠加：触发点（▼/▲）、确认点、三层挂价横线（存活期内）、成交点、止盈价轨迹、止损线、出场点；底部子图：10 s 同向主动成交额 / 1 h 中位（流量门倍数）、COOLDOWN 区间阴影、HALT / stale 区间；若强平流接入，叠加 BN 强平点（多头被平 / 空头被平）。时间范围可选最近 1 h / 一个回合 / 一天。

## 5. 对表面板（T2）

把回测口径的同日数字拉到旁边：Run B 每日「触发数 / 成交率 / 每回合 bp / 止损次数 / 出场构成」vs 回测同日（研究侧用 collector 行情跑 c1 引擎得到的 `episodes_<day>` / `outcomes_<day>`，由分析脚本写入看板数据目录）。偏差超过门槛（§交接 §5 放行门）标红。

## 6. 强平门影子面板（T2）

- 当日 / 累计：触发总数、门会挡的触发数与占比（主格 + k3 + nooi 三个口径）、被挡触发若照常进场的实际结局（这些触发在影子模式下真的进场了，所以直接有真实盈亏）：净、止损率，对比未被挡的。
- 这是前向验证的直接读数：若「会被挡」的回合持续比「不会被挡」的差，门转正式。
- 强平流与 OI 流状态：在线、最近一条时刻、feed age。

## 7. 告警（T2，走现有 alert_dispatch）

- 单回合亏损 ≥ 15 USDT @0.01（约三层满仓 60 bp）；当日净 ≤ 安全阀阈值；连续 2 次止损；COOLDOWN 触发；HALT / stale 出场；在途上限拒单出现；post_only 拒单 ≥ 3 次/小时；强平流或 OI 流离线 > 5 min；每日 UTC 00:10 推送前一日 KPI 摘要。

## 8. 数据面

- 全部来自 wick journal 六类行（`wick_trigger / confirm / order_end / round / markout / state`）+ 现有 tape（逐秒盘口）+ 强平 / OI 影子行；看板后端新增 `/api/wick/*` 只读聚合，不改 trader 决策。
- 回合级 parquet 每日落盘到看板数据目录，供研究侧对表脚本读取。
