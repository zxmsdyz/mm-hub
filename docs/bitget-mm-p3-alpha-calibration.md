# P3 · Bitget 小额实盘标定 α 与撤单延迟（方案，2026-11-06）

> 用户 2026-11-06 拍板：不等 35 天回测，直接用 UTA 小额实盘标 α。本文是执行方案，**不含代码**；下单引擎、journal、分析脚本都在 qflow / qflow-research（R3-a 的 `2026-11-01-htx-mm-r3a-live-fill-calibration` 是方法原型）。上位文档：[bitget-mm-strategy.md](bitget-mm-strategy.md) §5.2 门 2、§6 P3。

## 0. 要回答的三个数

| 数 | 现状 | 本轮目标 |
|---|---|---|
| **α**（本价位撤单中来自我前方的比例，`queue_model.py:139`） | Bitget 无实盘，回测结论悬在 α=0（负）与 α=0.25（正）之间 | 给出 at-touch 成交标定的点估计 + bootstrap 区间；按 queue_ahead 桶给实盘成交率 |
| **撤单延迟**（发撤 → 终态回报，UTA v3 WS） | 未真机测过 | p50 / p99；决定 BN 事件撤单（C1）用 ≤20 / ≤50 / 关 |
| **实盘 markout 与回测差** | HTX 线差 0.8 bp（实盘更毒） | Bitget 的 mo5 / mo30、被狙击占比，与 P1 7 天数（P1 −0.55 / D4 −0.36）对照 |

副产品：post_only 拒单率、429 次数、成交回报时钟偏差（HTX 为 −157 ~ +1 ms）。

## 1. R3-a 踩过的坑（本轮必须规避）

| 坑 | R3-a 实况 | 本轮规则 |
|---|---|---|
| 子单挂得太深 | 85% 子单挂在比 touch 深 >10 档的位置（fv 基差均值 bug），qa≈0、对 α 零贡献；871 单 at-touch 只出 13 笔可标成交 | **只允许挂 touch**（R2-e 已修 `fv_lead`）；journal 里 `price` 对 collector L1 as-of 核对，非 touch 子单从标定剔除 |
| 配置中途改 | S1/S2 bn_cancel 开关切换与时段混淆 | 全程**一套配置不动**；要测 C1 开 / 关就分成两个独立 run，每个 ≥ 24 h |
| 行情没封口 | 295 笔成交里 251 笔落在未封口日，只剩 44 笔能用 | 分析只用 collector S3 已封口的 UTC 日；run 结束后等 1 天再分析 |
| 时钟容差 | 无容差时召回 0.08~0.23，200 ms 容差才到 0.77~0.92 | 成交子单对齐容差 200 ms；同时记录 `end_ms − collector 对手成交 avail` 分位 |
| 样本量 | 13 笔 at-touch 成交 ⇒ α 区间 0 ~ 0.875，没有信息 | **at-touch 成交 ≥ 300 笔**才算一轮（R7-b 门槛 500） |

## 2. 配置（Tier 0 形态，与策略文档 §4.7 一致）

| 项 | 值 | 备注 |
|---|---|---|
| 账户 | UTA，API key 只开交易权限、关提现、IP 白名单（test2 EIP） | 下单 / 撤单走 **Lo-La 高速线** v3 WS（`vip-ws-uta-pri-a.bitget.com` / `vip-api-uta-a.bitget.com`） |
| 合约 | BTCUSDT 单合约 | 队列最厚、数据最全、与 P1 可比；ETH/SOL 等 BTC 出结果 |
| lot | 0.001 BTC（10 张，sizeMultiplier 0.0001） | 名义 ≈ 100 USDT；单笔风险 15 bp ≈ 0.15 USDT |
| price_mode | touch | |
| 安静门 | 1 s | |
| fv_lead 门 | δ = 0 | |
| fv_slow 选边 | **关** | 它是 Tier 1；开了会让成交 ×0.3，拖慢样本积累。D4 vs D3 的差别留 v2 A/B |
| BN 事件撤单 C1 | Run A **开**（300 ms / 1 tick） | 它是线上形态的一部分；撤单延迟本来就要测 |
| 有利追价 C3 / 驻留 30 s / 超时 30 s | 开 | |
| Q_max / 批次龄 / 止损 | 1 lot / 600 s / 15 bp | 比 HTX 紧 |
| 减仓 | 限价 IOC 在对手 touch（API taker 1.5 bp） | 不对冲 |
| 频控 | 单合约 ≤ 5 次/s，429 后退避 1 s，连续 3 次 HALT | |

## 3. 步骤

### Step 0 · UTA v3 真机探针（半天，先于一切）

qflow 的 `venue/bitget/uta_v3/*` 是照文档写的、未真机测过。单独跑探针（可参照 2026-08-03 的 `probe_bitget_ws_order.py` 结构），零策略逻辑：

1. v3 私有 WS 登录成功（Classic 账户会 30005 失败，正好验证账户模式）。
2. 挂 1 张 post_only 在 touch 下 50 档（不会成交）→ 撤 → 重复 200 次，记录 **发撤 → 撤单终态** 延迟分位，分 UTC 13~15 与其他时段各 100 次。
3. 故意挂一张会穿价的 post_only，确认被拒的错误码与回报形状（C1 后重挂时会遇到）。
4. 1 次真成交：挂 touch 等成交，确认成交回报字段（成交价、时间戳、手续费字段是否能直接读到返佣）。

**门**：撤单 p50 ≤ 50 ms 进 Step 1；50~100 ms 则 Run A 的 C1 预期无效，仍可跑但要加 Run B；> 100 ms 停下重新评估通道。

#### Step 0 结果（2026-11-06 UTC 08:58–09:14，BG 会话，test2 出口 35.75.175.46，BTCUSDT）

| 项 | 结果 | 对 Run A 的含义 |
|---|---|---|
| 账户 | v3 WS 登录 OK；`accountMode=unified`、**`holdMode=hedge_mode`**（双向持仓，posSide 必填）、`assetMode=multi_assets`、杠杆 20 | 做市只有一份净库存，双向模式会让买卖两侧各开一腿而不是对冲掉 ⇒ **Run A 前切单向持仓**（或引擎按 tradeSide=close 显式平） |
| 撤单延迟（其他时段，n=100） | 发撤 → 推送终态 **p50 14.2 / p90 41.0 / p99 462 ms**；发撤 → ack p50 10.1 / p90 37.2 / p99 237 | **过门**（≤ 50 ms），C1 可按 ≤ 20 ms「质量模式」口径预期；但 p99 近半秒，被狙击占比不会归零。UTC 13~15 的 100 次待补 |
| 穿价 post_only | 下单 ack **不拒**（code=0），之后订单推送 `orderStatus=cancelled, cancelReason=post_only_fill_cancel` | 拒单只能在推送里看到，C1 重挂的穿价判定与「子单终态」必须以推送为准，ack 不算在簿 |
| 撤已成交单 | `event=error, code=25204 Order does not exist` | 撤单失败 = 已成交的信号（本身是毒性指标） |
| 成交回报（topic=order） | `cumExecQty / cumExecValue / avgPrice / orderStatus=filled / side / tradeSide / holdSide / reduceOnly / createdTime / updatedTime / cancelReason / feeDetail[{feeCoin, fee}] / orderId / clientOid / matchType` | **没有逐笔成交时间与 tradeId** ⇒ §9.2 要求的 tradeId / 成交时刻须另订 fill 频道或查成交历史，Run A 前补 |
| 费率验证 | maker 0.001 BTC @76093.5 fee = −0.003804675 USDT = **−0.5 bp**；taker 平仓 0.003 @76099.4 fee = +0.03424473 = **+1.5 bp** | 费率真值确认 |
| 意外 | 离 touch 50 tick（5 USD）的循环单 3/100 在撤单到达前被扫成交；首笔平仓 IOC 只放 bid−2 USD 未成交（`ioc_not_full_cancel`），裸露 0.003 BTC 约 15 min 后才平掉 | ① Bitget BTC 几秒能动 5 USD，队列与波动比 HTX「活」；② **减仓 IOC 必须放对手 touch 并带容差**（策略文档 §4.5），探针修复后再跑 |

**Lo-La 高速线补测（2026-11-06 UTC 11:22，test2 IP 已开通，100 次）**：WS `wss://vip-ws-uta-pri-a.bitget.com/v3/ws/private`、REST `https://vip-api-uta-a.bitget.com`，东京 AWS ELB 直连不经 CloudFront，TCP 往返 ≈ 1 ms。

| 项 | Lo-La | 公共入口 |
|---|---|---|
| 下单 → ack | p50 2.93 / p90 3.25 / p99 3.57 / max 23.4 ms | p50 11.4 / p99 52.9 |
| 撤单 → ack | p50 2.74 / p99 3.23 | p50 10.1 / p99 237 |
| 撤单 → 推送终态（n=92） | **p50 3.53 / p90 3.88 / p99 8.26** | p50 14.2 / p99 462 |
| 交易所内部处理（下单 / 撤单） | 2.02 / 1.81 | 3.6 / 3.3 |

⇒ **Run A 直接用 Lo-La 线**（venue 端点改为上面两个地址）；C1 按 ≤ 20 ms 档设计，且 p99 也在 10 ms 内，尾部狙击窗口基本关死。本轮 8/100 张离 touch 5 USD 的单在撤单前被成交（午后波动大于上午），均已立即 IOC 平掉、仓位 0。

**UTC 13~15 补测（2026-11-06 UTC 13:10–13:12，Rust 探针，Lo-La 三个服务端 IP 各 100 次，0 成交）**：撤单 → 推送终态 p50 2.90 / 3.01 / 2.98 ms、p99 3.6~3.8 ms；挂单 → ack p50 2.50~2.75 ms；交易所内部处理约 2.0 ms（挂）/ 1.8 ms（撤），是延迟大头。美盘时段与其他时段无差别 ⇒ Step 0 全部完成，**C1 按 ≤ 5 ms 口径设计**。待核：bg-maker 账户 1.6 ms 比我们的交易所内部处理还短，疑账户等级或服务端路径不同，已请用户提供 bg-maker 的 receiveTime / pushTime 与 UID 对比。

Step 1 状态（2026-11-06 晚）：BG 会话只读摸底 Rust trader mm 模式与 Bitget UTA 通道 → 写 OpenSpec 提案 → 用户确认 → 建实现分支。BTC + ETH 并行初判无硬障碍，待核三项：同 UID 共用全仓保证金；WS 每连接 rateLimit=100、两实例独立连接；ETH 合约规格是否已登记。fill 字段形状改用 v3 成交历史 REST 查今天的真实成交，不下单。

### Step 1 · Run A（C1 开），直到 at-touch 成交 ≥ 300 笔

- 样本量估计：HTX at-touch 成交率 1.49%，Bitget 队列厚一倍，按 1% 估 ⇒ 需 ≈ 3 万子单；双边、每子单在簿中位 3~5 s ⇒ 约 **2~4 天**。若 3 天不到 150 笔，说明 Bitget 队列太厚，α 标定要改为「只在 queue_ahead ≤ 1 BTC 时挂」的子集（R3-a 该桶成交率 6.9%）。
- 必须覆盖 ≥ 2 个 UTC 13~15 时段和 1 个周末。
- 中途**不改任何参数**。只允许 HALT（止损熔断、429、行情 stale）。

### Step 2 · Run B（C1 关），≥ 24 h，可选

只有 Step 0 撤单 p50 > 50 ms、或 Run A 的被狙击占比 ≥ 回测 +10 pp 时才跑，用来量化 C1 在 Bitget 上到底砍掉多少毒性。

### Step 3 · 分析（等 collector 封口 1 天后）

按 R3-a 流程，产出下列表（round 目录放 qflow-research，命名 `2026-11-xx-bitget-mm-p3-live-alpha`）：

| 表 | 内容 |
|---|---|
| T1 子单结局 | 子单数、成交 / 撤单 / 拒单 / 超时数，`cancel_reason` 分布，在簿时长分位 |
| T2 挂价相对 touch | 应 100% 为 0 档；非 0 的列出原因 |
| T3 α 标定（at-touch） | gr / tb × 钳制 / 不钳制 × 容差 `200/0`、`200/200`：α=0 / 0.25 / 0.5 / 0.75 / 1 的模拟成交率、召回、误报率；bal_acc / LL 最优 α 与 block bootstrap 95% 区间 |
| T4 queue_ahead 分桶 | 子单数、实盘成交率、模拟成交率（α 各档）、mo5 / mo30、有毒占比；桶：0、(0, 0.5]、(0.5, 1]、(1, 5]、> 5 BTC |
| T5 Δt 分桶 | 成交距 BN 自侧最近不利事件：<50 ms、50~300、300~1000、>1000、5 s 内无事件；各桶 mo5 / mo30 / 有毒占比；与 P1 回测对照 |
| T6 延迟 | 撤单 p50/p99（按时段）、成交回报 vs collector 时钟偏差分位、post_only 拒单率、429 次数 |
| T7 净值 | 每 UTC 日：成交数、Σ(返佣 0.5 + mo_exit)、止损次数与成本（1.5 bp + 浮亏）、净 USDT；与 P1 7 天库存层对照 |
| T8 残余毒性来源分解（评审 §1.1 假设检验） | 把 mo30 按 Δt 桶加权分解：`<50 ms` 桶贡献 vs `>1 s / 无事件` 桶贡献；后者为主 ⇒ 毒性来自成交后同步趋势推进（E2 升高优先），前者为主 ⇒ 跨所 taker 仍更快（E1 升高优先） |
| T9 止损事件路径（评审 E3 的实盘版） | 每次触及 −15 bp：触发时刻、触发前 30 s 成交笔数速率相对峰值、下方 3 tick 内 L2 厚档存续时长、后 10/30/60 s 价格路径；样本会很少，只作登记 |

### Step 4 · 判定（主对话）

| 结果 | 动作 |
|---|---|
| α 点估计 ≥ 0.25 且 at-touch 实盘成交率 ≥ 回测 α=0.25 档；撤单 p50 ≤ 50 ms；实盘 mo30 与 P1 差 ≤ 0.3 bp | 过门 2 → P4 小额运行（BTC），同时开 ETH 的 Step 1 |
| α 区间含 0 且上界 < 0.25 | Bitget BTC 纯 touch 做市 DROP；转 SOL（等 P0 数据）或 ETH |
| α 够但撤单 > 100 ms | 通道问题：换出口 / 换 WS 节点再测，策略不动 |
| α 够、延迟够、但 mo30 比回测差 ≥ 0.5 bp | 与 HTX 同病（引擎乐观）：先做 R7-b 式逐笔对齐，找「被扫 vs 队列推进」占比，再谈放量 |

## 4. 前置清单（用户 / qflow 侧）

> 2026-11-06 BG 会话核查结果：Step 0 **尚不能开工**，没下过任何单。① 2026-08-03 实测当时的 key 是 **Classic** 账户（v3 WS 30005、REST 40084），除非之后升级或换 key，Step 0 第 1 步直接失败；仓库现役 `.env` 没有非空 Bitget key。② 旧探针 `probe_bitget_ws_order.py` 已于 2026-10-07 删除（可从 `archive/pre-legacy-runtime-retirement` 取回），且它只测到登录，挂撤计时 / 拒单 / 真成交都要新写（feature worktree）。③ 现有 Bitget 白名单 IP 是 test2 的 EIP 35.75.175.46——若新 key 绑它，探针只能在 test2 实盘 box 上跑，需单独授权；或把开发机出口 IP 加白名单。

- [x] **用户已确认（2026-11-06，BG 会话）**：UTA 已开通；key 白名单 = test2 EIP 35.75.175.46 ⇒ 探针在 test2 上跑；已当面授权 Step 0 探针真单（≤ 1 张挂撤循环 200 次 + 1 张穿价 post_only + 1 次 0.001 BTC 真成交和 IOC 平仓）
- [x] key 已就位并完成 Step 0（test2 上的 env 用后已删除）
- [x] **用户已确认（2026-11-07）**：切单向持仓（用户本人在 Bitget 操作，须无仓无挂单时切）；BTC + ETH 同上，test2 trader-003 跑 BTC（lot 0.001）、trader-005 跑 ETH（lot 0.01）；减仓沿用 mm 现有**市价 taker**（1.5 bp；标定 lot 下 L1 厚度远超单笔，不会滑多档，接受此偏离）
- [ ] Step 1 实现中（BG 会话）：OpenSpec **`qflow/openspec/changes/add-bitget-mm-calibration-units/`**（分支 `feature/bitget-mm-calibration-units`，worktree `~/workspace/qflow-bg-mm`）。**2026-11-07 15:40 本会话核对**：design §5 Tier 0 参数（touch / 安静 1 s / edge 0 / fv_slow 关 / BN 撤单 300 ms·1 tick / 30 s / 1 lot / 600 s / 15 bp / hazard 关 / lola_fast）与 journal·venue·runtime 三份 spec 全部对上 §2 与策略文档 §9.2；端点白名单 4 档；合约规格 BTC tick 0.1 步长 0.0001、ETH tick 0.01 步长 0.01；fill 行含 execId / tradeScope / feeDetail / createdTime，部分成交逐笔；止损行带滑点。四点待其确认：G3（qa>0 才挂）是否启用并写明；stale 阈值实际取值；C2/C3/C4 沿用 HTX 现行规则；taker 减仓行加 q / 批次龄 / 浮亏。**Run A 最早 UTC+8 11-07 23:00 ~ 11-08 02:00**（前提：用户今天切好单向持仓、one_way 下单格式探针一次通过）
- [ ] bg-maker 1.6 ms 核对：等用户提供其 UID 与应答帧 receiveTime / pushTime（我方交易所内部处理已 ≈ 2.0 ms，差距不在客户端）
- [ ] Step 1 载体：test2 新增一个专跑标定的 maker 策略单元（BG 会话出方案 → 用户确认 → 开工）
- [ ] qflow 侧：Bitget UTA v3 WS 下单 / 撤单真机探针（Step 0）
- [ ] qflow 侧：mm 模式接 Bitget 账户（HTX `htx-mm-1` 的 mm 配置迁一份，venue 换 bitget_uta，参数按 §2）
- [ ] 确认 collector1 `bitget_futures_sbe` BTCUSDT 三流在采且 S3 归档正常（分析依赖）
- [ ] 预算：Run A 2~4 天，lot 0.001，最坏情形每天 ≤ 10 次止损 ≈ 1.5 USDT/天

## 5. 不做

- 不调参、不开 fv_slow、不试 inside、不加合约——本轮是**测量**，不是优化。
- 不对冲、不走 web taker。
- 不根据前 24 h 的盈亏提前停（止损熔断除外）：样本不够就没有 α。
