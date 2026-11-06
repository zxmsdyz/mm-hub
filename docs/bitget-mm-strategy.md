# Bitget 永续做市策略设计（v1，2026-11-06）

> 费率前提（用户 2026-11-06 最终确认）：maker **−0.5 bp**（返佣，原 −0.6 已下调）；taker 分两种：**open API 1.5 bp**、**网页下单 0.7 bp**。做市单的止损 / 超时 / 平仓只能走 API ⇒ 按 **1.5** 算；0.7 只在对冲腿走 web 接口时适用。返佣无门槛。qflow-research 的决策记录 `.agent/decisions/trading_htx_bitget_fee_rates_2026_11_02.md` 已同步更正。
> 账户模式：**UTA**（已定）。品种：**只做 BTCUSDT / ETHUSDT / SOLUSDT**（用户 2026-11-06 拍板，其他合约不做）。
>
> 证据来源：qflow-research HTX 做市线 R1~R7（2026-10-28 ~ 11-03，`docs/strategy-research/2026-1*-htx-mm-*`）、设计文档 `designs/2026-10-27-htx-bg-maker-mm-predictive-hedge.md`、test2 `htx-mm-1` 实盘 11.9 h。HTX 线的结论**大部分可迁移**（毒性来源、撤单规则、公允价、库存规则都是 BN 领先 + 跟随所挂单这一结构的性质），Bitget 自身的数字只有少数几个，下文逐条标明哪条是 Bitget 实测、哪条是 HTX 借用。
>
> 因子全目录与分类见 [mm-factors.md](mm-factors.md)；本文只讲**先用哪些、怎么组成策略**。

---

## 0. 结论先行

1. **策略形态**：围绕公允价在 Bitget 双边挂 post_only，**永远挂在 touch**（买一 / 卖一），不预测方向、不挂深档、不做 hazard 停挂；收入 = 返佣 + （宽 tick 品种的）半价差，支出 = 被动成交后的逆向选择 + 止损。所有「因子」的用途只有三个：**算公允价、决定现在要不要挂、决定现在要不要撤**。
2. **品种只做 BTC / ETH / SOL，三者都是 1 tick 价差、价差收入 ≈ 0，净值 = 返佣 0.5 − 逆向选择 − 止损**。BTC 的 Bitget 被动成交 30 s markout 有两组相差 2~4 倍的数（R6-b HO −0.2 ~ −0.46 vs 借 HTX 口径 −0.8 ~ −1.2，§1.2），前者每笔净 +0.05 ~ +0.3 bp 可做，后者不可做。**P1 回测的唯一任务就是把这个数定下来**；SOL（tick 0.105 bp，BN 领先结构相同、队列薄得多）可能是三者里逆向选择与成交率平衡最好的，但 Bitget SOL 盘口还没采。
3. **止损成本是生死线**：API taker 1.5 bp + 浮亏，一次 15 bp 止损 ≈ 50 笔以上正常成交。HTX 线返佣 1.4 bp 仍被止损吃成 ≈ 0，Bitget 返佣只有其 36%，库存规则必须紧一个量级（§4.5）。
4. **最早有用的因子只有五个**（§3 Tier 0），全部有 HTX 线实证：BN 事件撤单、挂单前安静门、`fv_lead` 公允价门、队列前量、库存。其余微结构因子（OFI、多档不平衡、成交流、hazard、分钟级方向）在 HTX 线上**全部被证明对 maker 没有净值增量**，不要再花时间。
5. **真正的不确定项只能靠小额实盘回答**：Bitget 撤单延迟、touch 成交率（队列模型 α）、实盘 markout 与回测的差距（HTX 线实盘比回测差 0.8 bp）。

---

## 1. 经济账

### 1.1 每笔成交的账

```
每笔 maker 成交净值 = 半价差 S/2 + 返佣 0.5 − 逆向选择 AS(h) − 止损摊销 SL
```

| 项 | BTC（tick 0.013 bp） | ETH（0.042 bp） | SOL（0.105 bp） |
|---|---:|---:|---:|
| 半价差 S/2 | ≈ 0.006 | ≈ 0.02 | ≈ 0.05 |
| 返佣 | 0.5 | 0.5 | 0.5 |
| 逆向选择 AS（30 s markout，取负） | −0.2 ~ −0.46（R6-b HO）/ −0.8 ~ −1.2（借 HTX） | 未测 | 未测，无采集 |
| 每笔毛净（按 R6-b） | +0.05 ~ +0.3 | — | — |
| 一次 API taker 止损成本 | 1.5 + 浮亏（15 bp 止损 ⇒ ≈ 16.5） | 同 | 同 |
| 对冲腿（v2，web taker 0.7 或 Binance maker） | 0.7 + 半价差 + 基差噪声 ≈ 1 bp | 同 | 同 |

HTX 线的教训（R7-a）：**净值被单笔止损主导**。HTX 返佣 1.4 bp、每笔净 +0.3 ~ +0.6 bp，仍是「≈ +1 USDT/天 @0.001 BTC，实盘 ≈ 0」，原因是 30 bp 止损每天 0.4 次，一次吃掉几十笔。Bitget 返佣只有 HTX 的 36%，**止损频率必须比 HTX 低一个量级**，这决定了库存规则要比 HTX 更紧（§4.5）。

### 1.2 盈亏平衡

- 需要 `|AS(30 s)| < 0.5 bp` 且止损 ≤ 1 次 / 200 笔成交。R6-b 的 −0.2 ~ −0.46 是在「预挂 + 回调」那种形态下测的，普通 touch 形态未单测；按 HTX 口径借（P1 规则 −1.2 bp、加 BN 撤单 −0.8 bp）则**不可行**，设计文档 §4.5 的结论「Bitget 不单独做」就是这么来的。两组数差 2~4 倍，**第一轮回测要先把这个数定下来**（§6 P1）。
- **P1 中间结果（2026-11-06，subagent，7 天 UTC 10-25~10-31，L=10 ms，无 sweep 通道的乐观引擎，非 35 天）**：Bitget touch 连续链 P1 规则 mo30 **−0.55 bp**（HTX 同窗同口径 −1.26，少 0.70：Bitget 不滞后 BN、BN 撤单只砍 80% 成交而 HTX 砍 95%）；D4 形态（fv_lead 门 + 安静 1 s + fv_slow + BN 撤单）−0.36。按 0.5 返佣折每笔净：P1 三档 α 全负（−0.21 / −0.05 / −0.02）；D4 α=0.25 **+0.14**、α=0.5 **+0.16**、α=0 +0.03（IS 负）。成交/h：P1 319~1041、D4 48~235。库存层（sl 30 / age 1800 / 1 lot / D4 报价源）：α=0 −0.51 USDT/天（2/7 正日）、α=0.25 **+1.38 USDT/天**（6/7 正日，止损 3.3 次/天）。R6-b 的 −0.2~−0.3 是预挂形态上界，不是 touch 数；真正的 Bitget touch 数落在 −0.36 ~ −0.55。**结论悬在 α 上**（Bitget 无实盘可标），且引擎在 HTX 上乐观 0.8~1.0 bp/笔。
- 若 BTC 的 AS 落在 −0.5 以外，策略只剩两条出路：① SOL（tick 大 8 倍、队列薄、半价差 0.05）；② 把 BN 事件撤单延迟压到 ≤ 20 ms（HTX 线每笔 −1.25 → −0.80）。两条都要先有数据：SOL 要加采集，撤单延迟要 UTA 实盘测。

### 1.3 Bitget 自身已有的实测数字

| 项 | 值 | 来源 |
|---|---|---|
| BTC L1 单侧量 p50 | 3.63 BTC（HTX 1.83、Gate 2.13）；≥ 1 BTC 占比 85% | R1 §2.4 |
| 与 BN 100 ms 收益相关（lag 0） | 0.52（Gate 0.48）；lag +100 ms 0.19（HTX 0.24） | R1 §2.2/2.4 |
| 半价差 p50 | 0.006 bp（恒 1 tick） | R1 |
| 被动成交 30 s markout（HO，预挂形态） | −0.22 ~ −0.31 bp；随机方向对照 −0.16 ~ −0.46 | R6-b §8 |
| 同形态 HTX | −0.9 bp | R6-b §8 |
| Bitget 1 s/5 s 带符号流对 HTX 5 s 方向 IC | 0.16 / 0.19，但对 BN 特征零增量 | R2-d §4 |
| 行情网格新鲜度（最近一帧 >10 s 占比） | 0.006%（HTX 0.98%） | R6-a |
| 对冲腿实盘 | 7 h 开 20 平 26、taker 费 ≈ 返佣 70%（按 0.7 bp 算） | R2-e §8 |

判读：Bitget **队列厚、跟 BN 紧、报价刷新快**。队列厚 ⇒ 排到队首难、成交率低、但排到队首后被扫穿前有更多缓冲；跟 BN 紧 ⇒ 被狙击窗口比 HTX 短（HTX 做市商追补中位 61 ms）、同样的 BN 事件撤单在 Bitget 要求的延迟更严。两条合起来解释了为什么 Bitget 的 markout 比 HTX 轻，也说明 Bitget 的成交量会比 HTX 低。

---

## 2. 品种（用户 2026-11-06 拍板：只做 BTCUSDT / ETHUSDT / SOLUSDT）

| 合约 | tick bp | 24 h 成交额 M | Bitget 数据 | BN 数据 | 结构特点 |
|---|---:|---:|---|---|---|
| BTCUSDT | 0.013 | 3540 | SBE L1 + trade + L2 50 档，2026-07-29 起 | 全 | 队列最厚（L1 p50 3.63 BTC），成交率最低、单笔毒性最轻；**P1 回测载体** |
| ETHUSDT | 0.042 | 1983 | 同上 | 全 | 与 BTC 同结构；R6-b 没单独报 ETH markout |
| SOLUSDT | 0.105 | 201 | SBE 三流 **2026-11-06 起**（PR #1607） | L1 + trade（2026-10-11 起） | tick 是 BTC 的 8 倍、队列薄；成交率高、半价差 0.05 bp 可见；逆向选择未知 |

三者共同点：价差 ≥ 99% 时间为 1 tick ⇒ inside 挂单（T1-4）在这三个币上几乎没有触发机会，不作为 v1 杠杆。

**P0 采集**：✅ 已上线（2026-11-06 15:50 UTC+8，PR #1607，collector-b 重建）：`bitget_futures_sbe` SOLUSDT 三流在采并进 S3；books1 实测 74.8 帧/s。SOL 回测最早 2026-11-20 可跑（满 14 天）。

> 2026-11-05 快照里 tick 0.5~2 bp 的中流动性合约（XRP、SUI 等）每笔有 0.25~1 bp 半价差收入，经济结构比 1 tick 品种宽松得多；用户已拍板不做，记录在此备查。

---

## 3. 因子：按「什么时候能用上」排序

### Tier 0 · 上线即用，HTX 线已实证，不需要再回测证明方向

| # | 因子 | 定义 | 用途 | 证据 |
|---|---|---|---|---|
| T0-1 | **BN 自侧最优价不利移动** | 买单看 BN bid、卖单看 BN ask；相对 300 ms 前不利移动 ≥ 1 tick（Bitget tick 或 BN tick 取大） | **撤单触发器**。毒性主来源：35% 成交在 BN 不利移动后 50 ms 内（mo30 −1.94），有毒成交 69% 在 300 ms 内。撤单延迟 50 ms 避开 42% 有毒成交，20 ms 下每笔 markout −1.25 → −0.80 | R2-d A、R2-c §8、R3-b（state_w300 每 100 笔净 15.5 → 34.1） |
| T0-2 | **挂单前安静门** | BN 自侧最优价最近 W=1 s 内无任何不利 tick 才允许挂（或重挂） | **挂单门**。不依赖撤单延迟；保留 65% 成交、markout +0.15；W=5 s 压到 −0.5 但量剩 13% | R2-c §9 |
| T0-3 | **`fv_lead` 公允价** | `fv_lead = mid_bg(t_h) + (mp_bn(now) − mp_bn(t_h))`，`t_h` = Bitget 最近一帧 L1 时刻，`mp_bn` = BN 微价 | **挂单门 + 定价**。买要求 `fv_lead − bid1 ≥ 0`，卖要求 `ask1 − fv_lead ≥ 0`；误差中位 0.01 tick、在盘口内 98%。⛔ 不用任何基差均值（1 h / 60 s EWMA 都是噪声，误差 32~59 tick） | R2-e |
| T0-4 | **队列前量 `queue_ahead`** | 挂单时刻该价位在我前面的量（L1 量；有 L2 增量则精确） | **成交率与毒性分桶**。qa=0（挂进空档）最毒 mo5 −1.11 / 有毒 40%；(0.1, 0.5] BTC 最轻 −0.77 / 25%；>1 BTC 成交率 < 1%。用法：qa=0 时不挂（等于挂进价差内被扫）；qa 过大时该侧预期成交率低，可降低挂单量 | R3-a §二.6/7、R3-c |
| T0-5 | **库存 q 与批次龄** | 当前净库存（张）、每批 FIFO 持有时长、加权均价浮亏 | **报价开关**。q ≠ 0 时只挂减库存侧（线上现行形态）；批次龄 ≥ 上限 / 浮亏 ≥ 止损 → taker 减仓 | R3-c E 系列、R5-a BASE、R7-a |

Tier 0 的组合在 HTX 上就是线上 `htx-mm-1` 的现行形态（BASE：库存上限 2 张、非零只挂减库存侧、安静门 1 s、sl 30 bp、批次龄 1800 s、BN 撤单），35 天回测 33/35 天净正。

### Tier 1 · 需要在 Bitget 数据上验证一轮再定

| # | 因子 | 定义 | 预期用途 | 为什么还不能直接用 |
|---|---|---|---|---|
| T1-1 | **BN 50 ms 扫穿量** | 最近 50 ms 内吃穿 BN bid（或 ask）的成交量 | 撤单模型里 gain 最高的特征（R3-b xgb_full top1），可作 T0-1 的补充触发：BN 价没动但被大单扫到 touch 底 | R3-b 模型 ∪ state 对 state 单独只多 +2.6 bp/100 笔，增量小；要看 Bitget 上是否一样 |
| T1-2 | **本档缩水** | 我方价位 1 s 内档位量缩水 > 70%（他人撤单） | 撤单触发（R2-d B）：保留 88% 成交、mo30 −1.19 → −1.12 | 与 T0-1 联合增量未测 |
| T1-3 | **`fv_slow` 选边** | `mp_bn + EWMA_60s(mid_bg − mid_bn)`，只在该侧有 edge 时挂 | 严门：成交 ×0.31 ~ 0.68、markout −1.06 → −0.80 | 用量换质量，Bitget 返佣低、量本来就少，可能不划算 |
| T1-4 | **价差内挂单（inside）** | 价差 ≥ 2 tick 时买挂 ask1 − 1 tick、卖挂 bid1 + 1 tick | 宽 tick 品种价差偶尔张开时抢一档 | 回测引擎盲区（R2-c `inside_cross_fill` 假设），HTX 正在实盘探针（test2 2026-11-05 切 inside）；等结果 |
| T1-5 | **资金费 / 距结算** | 当期费率、距下次结算秒数 | 结算前 10 min 偏向持有收费方向、不持付费方向 | 没有 Bitget 资金费采集；beacon scout 有五所费率可借 |
| T1-6 | **UTC 时段** | 小时 | 只用于**调参数**（13~15 UTC 成交密度 2.15×、更毒），不做开关 | R2-d D / R5-a 已证明时段开关净负（少收返佣 > 省下的毒性） |

### Tier 2 · HTX 线已证明对 maker 无净值增量，**不要花时间**

| 因子族 | 结论 | 出处 |
|---|---|---|
| 方向模型（BN imbalance、OFI、微价偏离、signed flow 的 ridge/xgb，h=2~10 s，OOS IC 0.45） | 能预测「会不会被成交」（AUC 0.72），**不能预测「成交是否有毒」**（AUC 0.49）；用作撤单器丢 38~70% 成交只换 0.08 bp | R1 §2.3、R2-b |
| hazard / 波动门（rv10_z、rv60、z60，停加库存侧） | 对「未来 10 s 跳 > 2 bp」有区分度（ALERT 内 56~88%），但 ALERT 内成交 mo30 与外相同——BN 事件撤单已吃掉跳动毒性；25 格 Δ净全负 | R5-b |
| 分钟级方向预测（1~15 min，G0~G6 六族因子） | 方向 AUC 0.49~0.59 ≈ 安慰剂；模型学到的是「会不会动」 | R6-b |
| 跨所成交流（OKX/Bybit/Bitget/Gate 带符号流） | 单看 IC 0.16~0.23，加到 BN 特征上增量 0.0000 | R2-d §4 |
| BN 现货 | 永续领先现货 1~2 ms，增量 AUC ≈ 0 | R3-s |
| BN 多档 / 强平 / OI 增量 | 零增量 | candidates 2026-11-02 |
| 预挂梯子（多档） | 梯子单 mo30 −2.08、有毒 59%，总体变差 | R3-c lever 2 |
| 同价位空转重挂 / 按队列进度提前重挂 | 重挂只排队尾，钳制后 ≤ +5%、重挂 1.7~3.6× | R2-c |
| taker 进场 | 扣费全负 | R4 |

对应到 [mm-factors.md](mm-factors.md) 的分类：Tier 0 覆盖 D（跨所领先）、A（公允价）、B 的队列部分、F（库存）；Tier 2 把 C（成交流）、E（波动）、G（中低频）、H 的 hazard 部分基本判掉。I（执行与系统）不是因子，是 §5 的前置测量。

---

## 4. 策略设计

### 4.1 目标与边界

- 目标函数（按优先级）：**日净值 > 0 且 30 日内无单日 < −2× 日均净** ≥ 成交量 ≥ 资本占用。与 HTX 线「成交量第一」不同：Bitget 返佣薄，量大但净负没有意义。
- 单所、单腿、不对冲。对冲（Binance maker）留 v2：HTX 线证明一次对冲成本（taker 费 + 两次半价差 + 基差 60 s std 1.2 bp）与一个回合净收益同量级，hazard 又不能把对冲触发做准。
- 每合约独立实例、独立库存、独立参数；多合约只在账户层共享保证金与频控额度。

### 4.2 状态机

```
            ┌──────────── 两侧均满足挂单门 ────────────┐
            ▼                                          │
  FLAT ──挂双侧──▶ QUOTING_2 ──一侧成交──▶ QUOTING_REDUCE ──减库存侧成交 q→0──▶ FLAT
    ▲                 │                        │   (只挂减库存侧)
    │                 │                        ├── 批次龄 ≥ T_age → 该批 taker 减仓
    │                 │                        ├── 浮亏 ≥ sl_bp → 全部 taker 减到 0 → COOLDOWN
    │                 │                        └── |q| ≥ Q_max：本来就只有一侧，不变
    │                 ▼
    └── 任一输入 stale / 429 / 人工 ──▶ HALT（撤两侧；只保留 taker 减仓）──冷却──▶ FLAT
```

- 每侧**至多一个子单在途**；同侧下一子单只在上一子单终态（成交 / 撤成 / 拒）回报后提交。
- `QUOTING_REDUCE`：q > 0 只挂卖、q < 0 只挂买。这是 HTX 线上形态，等价于 `Q_max = 1 批`。v1 沿用；放宽到双侧 + skew 是 v2（HTX R3-c E1 库存偏斜的回合 bp 反而更差）。

### 4.3 公允价与挂单门（每帧评估）

```
mp_bn    = (bid_bn·ask_qty_bn + ask_bn·bid_qty_bn) / (bid_qty_bn + ask_qty_bn)
fv_lead  = mid_bg(t_h) + (mp_bn(now) − mp_bn(t_h))          # t_h = Bitget 最近一帧 L1

挂买条件（全部满足）：
  G1 fv_lead − bid1_bg ≥ 0                                  # 有 edge（R2-e）
  G2 BN bid 最近 1 s 内无不利 tick                            # 安静门（R2-c §9）
  G3 queue_ahead(bid1) > 0                                   # 不挂进空档（R3-a）
  G4 q ≤ 0（FLAT 或正在减空头库存）                            # 库存开关
  G5 age(BN L1) ≤ 300 ms 且 age(Bitget L1) ≤ 1000 ms         # 新鲜度
  G6 频控余量 ≥ 2 次（留撤单额度）
挂卖对称。
```

- 挂单价**永远 = 该侧 touch**（买 = bid1，卖 = ask1），post_only。不按 `fv_lead` 推离 touch；`fv_lead` 只进门 G1。
- 不挂数量：固定 `lot`（v1）。v2 按 queue_ahead 分桶调量。
- G1 的阈值 δ（`fv_lead − bid1 ≥ δ`）v1 取 0；T1-3 `fv_slow` 选边作为可选严门。

### 4.4 撤单与重挂（挂出后每帧评估，撤单路径走最快通道）

| 触发 | 规则 | 动作 | 证据 |
|---|---|---|---|
| C1 **BN 事件** | BN 自侧最优价相对 300 ms 前不利 ≥ 1 tick | 立即撤；条件解除且满足 G1~G6 后在新 touch 重挂 | R2-d A |
| C2 不利移动 | Bitget 自侧 touch 朝不利方向移 ≥ 1 tick（我方单已不在 touch 且在簿内更深） | 撤，按门重挂 | 基础规则 |
| C3 **有利移动即追** | Bitget 自侧 touch 朝有利方向移 ≥ 1 tick（我方单落后 touch 一档） | 撤，重挂到新 touch（原队列已死，等超时是 51% 超时的主因） | R2-c P1 |
| C4 驻留超时 | 同价位 ≥ `max_rest_ms`（30 s） | 以当前 touch 重挂 | R2-c |
| C5 本档缩水（可选） | 1 s 内本档量缩 > 70% | 撤，重挂 | R2-d B，T1-2 |
| ⛔ 不做 | 同价位空转重挂、按队列进度提前重挂、时段开关、hazard 停挂 | | R2-c、R2-d D、R5-b |

**Step 0 实测（2026-11-06，test2 → Bitget UTA v3 WS）：撤单 p50 14.2 / p90 41 / p99 462 ms**，在质量模式口径内但尾部近半秒。撤单延迟目标：**≤ 20 ms 为「质量模式」，≤ 50 ms 为可用线，> 100 ms 时 C1 几乎无效**（R2-c §8：L=20 每笔 −0.80 / L=50 −1.03 / L=100 −1.12）。Bitget 跟 BN 比 HTX 紧，门限应更严。账户已定 **UTA**，走 v3 WS 下单 / 撤单（qflow `venue/bitget/uta_v3/*` 据文档实现、**未真机实测**，P3 第一件事就是测它）。

### 4.5 库存与出场（比 HTX 更紧）

| 参数 | HTX 线上值 | Bitget v1 建议 | 理由 |
|---|---|---|---|
| `Q_max` | 2 张（0.002 BTC） | 1 批（= 1 lot）起步 | 返佣只有 36%，先把止损频率压到最低 |
| `max_lot_age` | 1800 s | **600 s** | 库存持有越久越像方向性头寸；Bitget 跟 BN 紧，均值回归到对侧成交的机会窗更短 |
| `sl_bp` | 30 bp | **15 bp** | 一次 30 bp 止损 = 60 笔以上正常成交；15 bp 配合更短批次龄 |
| taker 减仓方式 | 市价 | **限价 IOC 在对手 touch** | 避免滑穿多档；Bitget L1 厚 |
| 减仓优先级 | — | ① 减库存侧 maker 成交 → ② 对冲（v2：Bitget web taker 0.7 bp 或 Binance maker）→ ③ Bitget API taker 1.5 bp | 用户 2026-11-05/06 确认 |

出场不另设因子：R3-c / R6 证明「预测回调出货」无边；2026-08-23 轮证明「亏损 100% 发生在锁仓期内，到期后再赚每档为正」⇒ 出场侧靠规则，不靠模型。

### 4.6 风控与开关

- **HALT 条件**：BN 或 Bitget L1 超龄；连续 3 次 429；post_only 连续被拒 ≥ 5；撤单失败率 1 min 内 > 20%；mark 与 last 背离 > 20 bp；账户保证金率低于阈值；资金费结算前 60 s（可选）。
- HALT 下只允许减仓动作。
- **频控**（宽松初值，上线按 429 收紧）：单合约下单 + 撤单 ≤ 5 次/s，全账户 ≤ 8 次/s；每合约同时在簿 ≤ 2（每侧 1）；同价位最短驻留 200 ms 才允许 C4 重挂（C1~C3 不受限）。
- **日内熔断**：单合约日净 < −3× 过去 7 日日均正净 → 该合约停到次日。

### 4.7 参数表（v1 初值）

| 参数 | 值 | 来源 |
|---|---|---|
| `price_mode` | touch | R2-e |
| `quiet_window_ms` | 1000 | R2-c §9 |
| `bn_cancel_lookback_ms` / `bn_cancel_ticks` | 300 / 1 | R2-d |
| `max_rest_ms` / `timeout_ms` | 30000 / 30000 | R2-c |
| `fv_edge_min_tick` | 0 | R2-e |
| `queue_ahead_min` | > 0（宽 tick 品种可设为 ≥ 0.2 × L1 中位量） | R3-a |
| `Q_max` | 1 lot | §4.5 |
| `max_lot_age_s` | 600 | §4.5 |
| `sl_bp` | 15 | §4.5 |
| `stale_bn_ms` / `stale_bg_ms` | 300 / 1000 | 设计文档 §3.1 |
| `lot`（BTC） | 0.001 | 标定期 |
| `lot`（宽 tick 品种） | 按 50 ~ 100 USDT 名义 | 标定期 |

---

## 5. 评估口径与放行门

### 5.1 指标（全部按**每笔成交**，不按回合）

| 指标 | 定义 |
|---|---|
| `mo5` / `mo30` | 成交后 5 s / 30 s mid 相对成交价的带符号变化（bp，不利为负） |
| 有毒占比 | `mo5 < −k`（k = 1 tick 或 0.5 bp 取大） |
| 成交率 | 挂出子单中成交的比例；按 queue_ahead 桶分 |
| Δt_BN | 成交时刻距最近一次 BN 自侧不利事件的毫秒数；<50 ms 占比即「被狙击占比」 |
| 回合净值 | Σ(半价差 + 返佣 + mo_exit) − 止损成本，按 UTC 日 |
| 撤单延迟 | 发出撤单 → 撤单终态回报，p50 / p99 |

### 5.2 放行门（每阶段必须过才进下一阶段）

| 阶段 | 门 |
|---|---|
| 回测 → 实盘标定 | BTC 上 Tier 0 形态 35 天回测：每笔 `mo30 + 0.5 > 0`（α ∈ {0.25, 0.5, 0.75} 三档中至少两档）、IS/HO 同号、净正日 ≥ 80% |
| 标定 → 小额 | 实盘 ≥ 300 笔成交：撤单延迟 p50 ≤ 50 ms；实盘 mo30 与回测差 ≤ 0.3 bp（HTX 差 0.8 时判 BLOCKED）；被狙击占比 ≤ 回测 +10 pp |
| 小额 → 放量 | 连续 14 个 UTC 日净正 ≥ 11 天；止损 ≤ 1 次/300 笔；放量只放合约数，不放单笔 lot（HTX R7-a：放大 lot 只放大同一结构） |

---

## 6. 路线

| 阶段 | 做什么 | 数据 | 产出 / 门 |
|---|---|---|---|
| **P0 补采集**（立即） | collector1 加 `bitget_futures_sbe` SOLUSDT（depth 双连接 + trade + depth_l2） | — | 14 天后 SOL 可回测；≈ 1 GB/天 |
| **P1 BTC/ETH 定数**（已派 subagent，2026-11-06） | 用 qflow-research 的 mm 模拟器（R2-c 链 + clamp）在 Bitget BTC 09-27 ~ 10-31 上跑 Tier 0 形态，α ∈ {0.25, 0.5, 0.75}；ETH 同跑；对比 R6-b 的 −0.2~−0.46 与 HTX 借口径的 −0.8~−1.2，**定下 Bitget touch 成交的 mo30**；按本文费率折净值 | 已有 | 过 §5.2 门 1 则进 P3；不过则等 SOL 数据与 UTA 延迟实测 |
| **P2 SOL 回测** | 同模拟器跑 SOL；额外报 queue 换手时间、成交率 | P0 满 14 天 | 与 BTC/ETH 比每笔净值与止损频率 |
| **P3 实盘标定**（方案见 [bitget-mm-p3-alpha-calibration.md](bitget-mm-p3-alpha-calibration.md)；**Step 0 已过门 2026-11-06：撤单 p50 14.2 / p99 462 ms，费率 −0.5 / +1.5 实测确认；账户为 hedge_mode 需切单向；order 推送无 tradeId**） | Step 0 UTA v3 WS 真机探针（撤单延迟）→ Run A：BTC、lot 0.001、Tier 0 形态、C1 开、一套配置不动，直到 at-touch 成交 ≥ 300 笔（约 2~4 天）→ 封口后按 R3-a 流程标 α、分桶成交率、Δt、净值 | — | 过门 2 |
| **P4 小额运行** | 过门的 1~3 个合约、Tier 0 形态；每 UTC 日封口复盘 | — | 过门 3 |
| **P5 v2** | ① Tier 1 逐项 A/B（本档缩水、fv_slow、按 qa 调量）；② 对冲腿（web taker 0.7 / Binance maker）；③ 多合约账户层风控 | P4 journal | 每项单独 LAND/DROP |

P1 和 P0 没有依赖，可以同时开始。**更正（2026-11-06）**：Bitget 快照路径的队列模型已在 qflow-research `main` 收口（R8-d `build_bitget_pls_days.py` 价位长表、`repost_policy_r8d.py` venue 化链、R8-e2 `inventory_sim_r8e2.py` 库存层，2026-11-03~04），设计文档 §1.2「未收口」已过时；但 `docs/research-box-logik-shared` 分支落后 main 61 个提交，开跑前要合 main 或在 main 开 worktree。注意 main 上 R8-e2 已把「Bitget 纯做市」按 **0.6 bp / α=0** 判 DROP；按更正后 0.5 bp / α=0.25 库存层转正（+1.38/天），差异全在 α。数据在 S3（BTC 约 30~40 G、BTC+ETH 60~80 G），research box `i-0f91ab21edea1690f` 当前关机，35 天主回测需用户授权开机；算量 BTC ≈ 9~13 h 串行。round 目录：`qflow-research/docs/strategy-research/2026-11-06-bitget-mm-p1-touch-markout/`（commands.md §B 为开机后完整计划）。

---

## 7. 与 HTX 线的差异一览

| | HTX `htx-mm-1` | Bitget v1 |
|---|---|---|
| 返佣 / taker | −1.4 / 1.65 | −0.5 / API 1.5（web 0.7 仅对冲） |
| 品种 | BTC（恒 1 tick） | BTC / ETH / SOL（均恒 1 tick） |
| 收入结构 | 纯返佣 | 纯返佣（SOL 半价差 0.05 可忽略） |
| 对手盘 | HTX 跟 BN 慢（追补 61 ms），狙击多、量多 | 跟 BN 紧、队列厚；狙击窗口短、量少、毒性轻 |
| 撤单通道 | HTX WS `cancel_order` | UTA v3 WS（未真机实测） |
| 库存 | Q_max 2 张、龄 1800 s、sl 30 | 1 lot、600 s、sl 15 |
| 目标 | 成交量第一 | 日净 > 0 第一 |
| 对冲 | Bitget taker（已关） | 无；v2 Binance maker |

---

## 8. 已决事项（2026-11-06）

| 事项 | 决定 |
|---|---|
| 费率 | maker −0.5；taker API 1.5（止损 / 平仓）、web 0.7（仅对冲） |
| 账户模式 | UTA |
| 品种 | 只做 BTCUSDT / ETHUSDT / SOLUSDT |
| P1 回测 | 本会话派 subagent 在 qflow-research 上执行（研究 box 开机需用户另行授权） |

## 9. 回测要准，Step 0 之外还要采什么（2026-11-06）

HTX 线实盘比回测差 0.8 bp、Bitget 回测结论悬在 α 上，根因都是**回测引擎看不到三样东西**：队列里谁在我前面、我的单实际什么时候在簿、BN 事件到我撤单落地之间发生了什么。按补哪一项最能缩小这个差距排序：

### 9.1 第一优先：Bitget 盘口**增量**流（解决 α 问题的根本办法）

| 现状 | 问题 | 要补 |
|---|---|---|
| `bitget_futures_sbe` 只有 books1 / books50 快照 @20ms / trade；SBE 端点**没有** depth_diff | 20 ms 内同一价位先撤 3 BTC 再挂 3 BTC，快照看到的是「没变」；只能用 α 假设撤单来自前方的比例 | 重开 JSON v2 venue `bitget_futures` 的 `depth_diff`（← `books` 频道，500 档逐事件增量），BTC 先、ETH/SOL 后。Gate 走 depth_diff 链的队列模型误差 0.94%，**不需要 α** |
| | 增量流事件时间戳 + 公共 trade 可按价位重建「成交吃掉多少、撤单撤掉多少」 | 两条流用交易所时间戳对齐，必须同机采（collector1 已有 JSON 解码，是 07-29 被 SBE 取代而非删除） |

磁盘：JSON books 增量 BTC 估 2~4 GB/天，比 books50 的 9.9 GB 小。

> **collector 侧评估（2026-11-06）**：不是「重开」——Python 采集器（含 JSON v2 实现）已随 2026-10 退役整体删除，现役 Rust collector 只有 `bitget_sbe.rs`，config 注释 L509 那句已失效。可行路径 = **新写 Rust venue `bitget_futures`**（JSON v2 ws：`books` 增量 + 首帧快照 / checksum，形态参照 `htx_futures.rs`），与 §9.3 状态流（`ticker` 频道给 mark / index / funding / nextFundingTime）合并进同一 venue，总工作量约 6~9 h 经过时间，走 OpenSpec + 新 venue 三道门。磁盘：BTC 14~28 G（7 天保留），data 盘余 68 G，**只够 BTC 一个**，加 ETH 需缩 retention 或扩盘。**需用户在 collector 会话授权**开提案与 collector-b 重建。
>
> **⚠ 探针实测后前提不成立（2026-11-06 collector 会话，75 s 抓包）**：Bitget JSON v2 `books` 是 **100 ms 聚合推送**（update 10.16 帧/s、ts 间隔 p50 100 ms、每帧 ≈ 30 档净变化），不是逐事件增量；20 ms 内先撤再挂在这条流上同样是「没变」。它比现役 books50@20ms **时间分辨率粗 5×**，只多了 500 档 vs 50 档深度。Gate 链能到 0.94% 误差是因为 Gate 增量逐事件；**Bitget 公共端点没有逐事件频道** ⇒ 回测侧不可能摆脱 α，α 只能靠 P3 实盘成交标定（这也是已选路线）。`ticker` 流 9.21 帧/s，markPrice / indexPrice / fundingRate / nextFundingTime / holdingAmount 字段齐全，可直接落 canonical `mark`，§9.3 价值不受影响。`liquidation` 频道订阅成功但 75 s 零帧，需更长抓包。
> **本文判定：§9.1 的 books 增量放弃；只做 §9.3 `ticker` 状态流。**

### 9.2 第二优先：自己订单的**完整生命周期**（P3 Run A 的 journal 必须有）

每个子单一行，缺任何一列都会让 α 标定或 markout 口径失真：

| 字段 | 用途 |
|---|---|
| 本地发单时刻、交易所 ack 时刻（`cTime`）、本地收到 ack 时刻 | 下单延迟 = 三段拆开 |
| 本地发撤时刻、交易所撤单终态时刻（`uTime`）、本地收到时刻 | 撤单延迟 p50/p99，C1 有效性的直接证据 |
| 挂单时刻的 Bitget L1（价、量）与 L2 本价位量 ⇒ `queue_ahead` | α 标定与分桶的自变量 |
| 挂单时刻与撤单决策时刻的 **BN L1 快照**（bid/ask 价量、BN 事件时间戳） | Δt 分桶、「BN 已动 → 我撤单落地」全链路时延 |
| 成交回报：成交价、`tradeId`、交易所成交时刻、**手续费 / 返佣字段原值** | 用 tradeId 对公共 trade 流校时钟；验证返佣确实是 −0.5 |
| 子单结局：filled / cancelled(reason) / rejected(code) / timeout | 拒单码分布、post_only 穿价率 |
| 当时库存 q、批次龄、浮亏 | 止损归因 |

HTX 的 journal 已有 `extra.maker.{queue_ahead, spread_ticks, inside, price, attempt_seq, reprice_reason}` 与 `maker_order_end`、`post_fill_level`，缺的主要是**交易所侧时间戳三元组**和**BN L1 快照**。

### 9.3 第三优先：Bitget 状态流（风控规则与资金费倾斜要用）

| 流 | 用途 | 现状 |
|---|---|---|
| Bitget mark / index price（WS `ticker`） | HALT 规则「mark 与 last 背离 > 20 bp」；标记价是强平依据 | ✅ **已上线**（2026-11-06 19:06，`bitget_futures` venue `mark` 流，BTC，见 §10） |
| Bitget 资金费率 + 下次结算时间 | T1-5 结算前倾斜；结算前 60 s HALT | 同上，payload `funding_rate` / `next_funding_ts` |
| Bitget 强平流 | 事件标注，不进因子 | 公共频道存在（ack 成功）但稀疏，本次未纳入 |
| BN `@markPrice`、`@forceOrder` | hazard 复核（已判 DROP，低优先） | ✅ 已部署在采（collector-a，2026-10-28 05:16 UTC 起，含 oi） |

### 9.4 第四优先：时钟与链路

| 项 | 用途 |
|---|---|
| test2 与 collector1 的 NTP 偏差、BN→test2 与 BN→collector1 的接收时刻差 | 否则 journal 与 collector 对齐只能靠 200 ms 容差，α 区间永远宽 |
| SBE `sts`（推流发送时刻）保留 | 拆 engine→send / send→recv；books50 现在不落 raw，sts 丢了（决策是磁盘换精度，保持） |
| 429 / WS 重连 / 拒单日志带时间戳 | 频控阈值收紧依据；重连空窗要从回测样本剔除 |

### 9.5 不需要补的

- Bitget 50 档快照的 raw 原帧（10 GB/天，normalized 已是完整真源）。
- 跨所成交流（OKX/Bybit/Gate）：对 BN 特征零增量（R2-d）。
- BN 现货、BN 多档 / OI / 强平增量：已判无增量（R3-s、candidates）。
- 更多合约：只做 BTC/ETH/SOL；SOL 的 `bitget_futures_sbe` 三流是 P0，已登记。

## 10. 待办

- [x] P0：Bitget SOLUSDT 采集已上线（2026-11-06，PR #1607）
- [x] §9.3：`ticker`→mark 状态流 **已上线**（2026-11-06 19:06 UTC+8，PR #1613，develop `a09346b3d`，collector-b）。实测 9.25 帧/s；落盘 `normalized/bitget_futures/btcusdt/mark/<UTC 日>/part-*.parquet`（≈ 0.13 GB/天），S3 次日归档。`price`=markPrice，`exchange_ts` **毫秒**，payload 键 `index_price` / `funding_rate` / `next_funding_ts`（ms）与 BN mark 对齐，另有 `last_price` / `bid_price` / `ask_price` / `bid_size` / `ask_size` / `holding_amount`。未纳入：liquidation、ETH/SOL、books 增量
- [x] P1：7 天中间结果已回填 §1.2；35 天主回测搁置（box 关机，用户选择直接走 P3）
- [x] P3 Step 0：过门（撤单 p50 14.2 ms）；UTC 13~15 的 100 次待补
- [ ] P3 Step 1：qflow 侧把 mm 模式接 Bitget UTA 账户，按 p3 文档 §2 配置跑 Run A
