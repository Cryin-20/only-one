# 做市策略与 Avellaneda-Stoikov 模型

- **来源**：网络调研（firecrawl 多轮）+ 学术论文摘要 + Avellaneda-Stoikov / Guéant-Lehalle-Fernandez-Tapia / Cartea-Jaimungal-Penalva / Hummingbot 思想二手转述 + 加密做市机构公开资料（Tier 分级；**未直接读 Avellaneda-Stoikov 2008 全文、Cartea 教材、Hummingbot 源码**；通过 Avellaneda-Stoikov 论文 2008 QF 原文可访问 PDF（ResearchGate）、Guéant-Lehalle-Fernandez-Tapia 2013 arXiv 1105.3115、Hummingbot 官方文档 + Academy blog、LLMQuant/quantbeckman/Medium 解读、arXiv 2508.20225（adverse selection）、MDPI 2021 RL 综述、crypticweb3 机构排名交叉验证）
- **日期**：2026-08-30
- **主题**：量化交易 ｜ 标签：做市 · Avellaneda-Stoikov · 库存管理 · 返佣 · Hummingbot · 加密做市 · 散户 vs 机构 · 逆向选择
- **一句话主旨**：做市不是"挂单收钱"——是用库存风险换 spread 的艺术；散户真正位置是 regime-aware 慢做市或 maker 返佣套保，不是和高频抢 speed

> **配套条目**（直接引用，本文与既有知识网互引）：
> - 《自动量化加密货币的成功与失败》§4 机构做市商（Wintermute / Jump / GSR / Cumberland）+ §3.4 Hummingbot 失败案例——本文核心引用
> - 《市场微观结构与滑点建模》§1-§4 订单簿 + 撮合 + §7 项目映射——本文直接应用
> - 《市场 Regime 检测与牛熊识别》regime-aware 做市——本文应用面
> - 《统计套利与 Pairs Trading》§1.3 做市与 pairs 边界——本文互引
> - 《风险量化 VaR/CVaR/ES》§7 inventory 风险度量——本文应用
> - 《自动量化项目架构初稿》L5 策略层 + L0 多所——本文应用面
> - 《行为金融与交易者心理偏差》§3 Hummingbot 行为偏差分析——本文互引
> - 《回测方法论深化与 CPCV》CPCV 在做市策略的应用——本文回测方法

---

## 可复用原则（决策时引用）

1. **做市的本质是"用库存风险换 spread"——不是"挂单收钱"**：每一次成交的利润 = (ask − bid) × 数量 − 逆向选择损失；后者才是真成本，前者只是"账面利润"。散户 90% 的做市亏损来自把账面利润当真利润，没把逆向选择损失计入风控预算。
2. **库存偏离 0 越远 = 风险越大，不是利润越大**：Avellaneda-Stoikov 模型的革命性贡献就是把"库存"做成报价偏离的核心因子——库存为正则报价下移（更想卖），库存为负则报价上移（更想买），库存为 0 时报价对称。任何"想多持仓"的做市策略都背离了 AS 模型的本质，必然变成隐性马丁。
3. **gamma（风险厌恶）不是"我多保守"，是"我多不在乎 spread"**：γ ↑ → spread 变宽（更保守、fill rate 下降）；γ ↓ → spread 变窄（更激进、被逆向选择的概率上升）。加密散户做市的真正难题是找不到一个 γ 既能让 spread 覆盖 taker 成本、又不至于 fill rate 低到"挂了一周没成交"。
4. **订单到达强度 κ 决定了 spread 的"地板"**：δ_spread = (1/γ)·ln(1 + γ/κ) 这一项意味着 κ 越大（市场越活跃），最优 spread 越窄；κ 越小（市场越冷清），最优 spread 越宽。**加密市场半夜深度塌缩时 κ 跌到白天的 1/3-1/5，做市策略必须把 spread 同步放大**——这正是 Hummingbot "ask_spread / bid_spread" 动态配置的物理意义。
5. **散户做市不是和高频抢 speed**：Hummingbot 2019 年起步时延迟比机构 HFT 慢 50ms+；机构现在已降至微秒级。**散户 50ms 延迟 = 任何"挂 best bid/ask 等成交"的策略必然被逆向选择**——因为知情交易者（机构 HFT、funding arb、套利 bot）的速度足以在你挂单前完成下单并推动价格。
6. **散户做市的真正位置是"regime-aware 慢做市"或"maker 返佣套保"**：regime 稳定（震荡市 / 牛市）+ spread 厚 + 波动率适中时挂单；或接 maker 返佣（80% 返点后费率 0.004%/次）+ 同步在另一腿对冲 delta——本质上是把"做市的库存风险"外包给对冲腿。
7. **库存管理 ≠ 平仓库存**：做市策略的目标不是"库存 = 0"（那等于不做市），而是"库存稳定在某个目标区间内 + 偏离时主动修正"。Hummingbot `inventory_target_base_pct = 50%` + `inventory_skew_enabled = True` 是这种思想的工程化实现；AS 模型则更进一步——报价本身就要随库存偏离而偏斜。
8. **延迟比 spread 更影响 P&L**：在 1ms 延迟下 spread = 1 tick 可盈利；在 50ms 延迟下同样的 spread 是亏损（被知情交易者吃光）。**散户的工程优先级**：先打延迟（co-located 托管 / WS 而非 REST），再调 spread，最后才调 γ/σ/κ。
9. **加密 maker 返佣的真正价值是降低 spread 地板**：Gate 80% 返点后 maker 0.004%/taker 0.01%——挂 maker 单做市的盈亏平衡 = 0.004%（单边往返）；无返点则需要 0.02% 才能持平。**返点对做市的影响远大于对费率套利**——因为做市的成交频次远高于费率套利的调仓频次。
10. **做市的 P&L 是"小额频繁盈利 + 偶尔大亏"——又是马丁签名**：败者画像（Reddit Hummingbot 案例）："一分钱没赚到"（fill_rate 太低）、"跑了一周被一次暴跌打爆"（库存偏离 + 单边行情）。这正是《自动量化加密货币的成功与失败》第十章"马丁签名"在做市语境下的具体表现。

## 核心逻辑链

1. **前提**：做市商通过挂买卖单提供连续流动性，盈利来源 = spread × 成交次数 + 返佣 × 成交金额 − 逆向选择损失 × 成交次数。其中 spread 是账面、返佣是确定的、逆向选择是浮动的；后者才是真正的成本项。
2. **机制（AS 2008 原始模型）**：做市商最大化终值时刻的指数效用函数 `u(s,x,q,t) = max E[-exp(-γ(X_T + q_T S_T))]`。模型求解给出 reservation price（库存中性化的目标价）= s − qγσ²(T−t)，加上 spread 项 (1/γ)·ln(1 + γ/κ)，给出最优 bid/ask 报价。
3. **机制（库存管理的核心）**：库存 q 偏离 0 时，reservation price 偏离 mid——库存为正则 reservation < mid（更愿意在 mid 之下卖），库存为负则 reservation > mid（更愿意在 mid 之上买）。这就是"做市不是平均报价，是动态报价"——AS 模型把库存纳入报价偏离本身。
4. **机制（参数直觉）**：γ（风险厌恶）控制 spread 宽度（γ ↑ → spread ↑）和库存偏斜速度（γ ↑ → 库存修正更激进）；σ（波动率）放大 spread 项（σ ↑ → spread ↑）和库存惩罚（库存暴露的方差变大）；κ（订单到达强度）压缩 spread 项（κ ↑ → spread ↓）；T−t 反映"终值压力"——剩余时间越少，spread 越窄（避免未平仓库存跨期）。
5. **结果**：在 AS 模型下，做市商**主动**调整报价以管理库存；在纯做市模式（被动挂 best bid/ask）下，做市商**被动**接受库存积累。两者 P&L 差异极大——前者 σ 小、回撤小、Sharpe 高；后者 σ 大、回撤大、Sharpe 接近 0 甚至为负。
6. **加密特殊性**：① 订单簿薄 + 流动性脆性（半夜/周末 depth 塌缩 80-90%） ② 24/7 无熔断 + ADL（盈利 short 也能被强平） ③ maker 返佣（80% 返点让 spread 地板降到 0.004% 量级） ④ HFT 机构压倒性速度优势 ⑤ 链上/跨所延迟不对称 → 散户做市的可行区间被压缩到"regime-aware 慢做市"+"返佣套保"。
7. **行动指引**：100 USDT 项目 Phase 1 不上做市（费率套利更稳）；Phase 2 实验性做市（5-10 USDT 测试网 / 慢做市 / 返佣套保）；Phase 3 30 USDT 小规模做市。杠铃结构 = 90 USDT 费率套利 + 10 USDT 慢做市实验。
8. **决策口诀**：做市不是"挂单收 spread"——是"用库存风险换 spread 的艺术"；散户不在 spread 上和高频抢 speed，在 regime 择时 + 返佣套保 + 库存偏斜三件套上找位置。

---

## 分章笔记

### 第一部分　做市商与做市策略概述（必含）

#### 第一章　做市商定义 + 三种模式 + 盈利来源

**1.1 做市商的一句话定义**

> 做市商（Market Maker）= **持续**在订单簿两侧挂限价单（bid + ask），以 spread + 返佣为主要收入来源，提供市场流动性的交易者。

承自 Hummingbot Academy "What is Market Making" + Investopedia "Market Maker"："a firm or individual who actively quotes two-sided markets in a security, providing bids and offers along with the market size of each"。Hummingbot 的"pawnshop"比喻尤其贴切——做市商像典当铺老板，低价收 Susan 的吉他、高价卖给 Mike，从差价（spread）获利。

**1.2 三种角色的区分：撮合者 / 对赌者 / 做市商**

| 角色 | 持仓方向 | 盈利来源 | 典型代表 |
| --- | --- | --- | --- |
| **撮合者（Exchange）** | 中立（不持头寸） | 手续费 + 提币费 + 上币费 | Binance / Coinbase / Kraken |
| **对赌者（Speculator）** | 单向（赌方向） | 价差 | 散户 / 趋势策略 / 网格策略 |
| **做市商（Market Maker）** | 双向（gross 大、net 接近 0） | spread + 返佣 + 库存折价 | Wintermute / Jump Crypto / Optiver |

**关键洞察**：做市商**不是**赌方向的——它**主动承担**库存暴露（gross 大），但**主动管理**库存偏离（net 接近 0）。"gross 大 net 小"是做市策略区别于方向性策略的核心指纹；这也是 AS 模型为何把"库存"作为核心变量的原因。

**1.3 三种做市模式**

```
模式 A：被动做市（Passive / Symmetric Market Making）
   bid = mid - δ/2
   ask = mid + δ/2
   （δ 固定或按 spread 自适应，无库存偏斜）

模式 B：主动做市（Avellaneda-Stoikov / Inventory-Aware）
   bid = reservation - δ/2
   ask = reservation + δ/2
   reservation = mid - q·γ·σ²·(T-t)
   （库存偏斜，报价随库存动态调整）

模式 C：混合做市（Hybrid / Inventory-Skewed Passive）
   bid = mid - δ_b(q)
   ask = mid + δ_a(q)
   δ_b ≠ δ_a，库存多时 δ_a < δ_b（更想卖）
   （Hummingbot inventory_skew + pure_market_making 是典型实现）
```

**模式对比**：

- **模式 A**：实现最简单（Hummingbot 默认 PMM）；但库存会无控积累，regime 切换时被打穿（Reddit 案例"一分钱没赚到"或"被暴跌打爆"）。
- **模式 B**：理论最优；实现需要实时计算 γ/σ/κ/(T−t)；加密市场的参数标定本身是难题（高频 tick 数据 + 自适应窗口）。
- **模式 C**：折中方案——Hummingbot `inventory_skew_enabled` + `inventory_target_base_pct` 把库存偏斜做成可配置参数；适合散户/中小资金。

**1.4 做市的四种盈利来源**

| 来源 | 机制 | 典型量级（加密散户） |
| --- | --- | --- |
| **Bid-Ask Spread** | 买入 bid 卖出 ask 的价差 | 主流币 spread 0.01-0.05%；山寨币 0.1-1% |
| **Maker Rebate** | 交易所返佣（Gate 80% / Binance BNB 25%） | 返点后 maker 0.004%（Gate）/ 0.015%（Binance） |
| **信息优势（Inventory Price Discount）** | 看到 flow 信号后调整报价 | 散户几无；机构 HFT 是这一项的主要受益者 |
| **库存折价（Inventory Discount）** | 库存暴露随时间衰减（终值约束） | AS 模型自然体现，γ ↑ 库存折价 ↑ |

**散户真正能拿到的只有 #1 和 #2**。#3 是机构 HFT 的护城河（延迟 < 1ms vs 散户 50ms+），#4 是 AS 模型的自然结论（任何做市策略都自动包含此项）。

**1.5 "做市商"与"流动性提供者"的概念辨析**

- **狭义做市商**：必须双边挂单、必须接 Maker 返佣（部分所有强制要求）
- **广义流动性提供者**：包括所有挂限价单等成交的参与者（包括散户挂单、机构 algo 等）

Hummingbot 文档用广义概念；机构做市商（Wintermute）的"做市"是狭义。本条目按狭义做市展开。

---

#### 第二章　机构 vs 散户的鸿沟

**2.1 机构做市商的"五件套"能力**

| 能力 | 机构做市商（Wintermute / Jump / GSR） | 散户 / Hummingbot 用户 |
| --- | --- | --- |
| **延迟** | co-located + FPGA < 1ms（部分 100us） | 50ms+（REST）或 10ms（WS） |
| **跨所覆盖** | 50+ 交易所统一调度 | 1-3 个所（CCXT 适配） |
| **对冲基金/自营** | 自带对冲账户（cross-exchange inventory hedge） | 无（需用户自己用 funding arb 对冲） |
| **基础设施** | 自营机房 + 专线 + 跨所 colocated | 云服务器 + 公有 API |
| **数据/研究** | 数十人量化研究团队 + 历史 L3 tick 数据 | 公开 K 线 + 部分 L2 depth |

承自 crypticweb3 "Best Crypto Market Makers in 2026" + Hyrotrader Crypto Market Makers Guide："Wintermute 覆盖 50+ 交易所、日交易量 22 亿+美元"；Jump Crypto"低延迟 HFT 基础设施 + 持续报价"。

**2.2 散户的真正劣势：速度**

- 2019 年 Hummingbot 用户实测比机构 HFT 慢 50ms+；现在机构已降至微秒级
- 50ms 在加密 BTC 的 spread 时间尺度（典型 1-10ms）上是几个数量级的劣势
- **含义**：任何"挂 best bid/ask 等成交"的散户策略 = 100% 被逆向选择（机构 HFT 在你前面成交并推动价格，你成交时已落后）

**2.3 散户的真正优势：灵活性 + 资金规模小**

- **灵活性**：可以随时下线、换品种、换策略；机构有合同义务和库存上限
- **资金规模小**：在 spread 厚的品种（小币、新币、刚上线）反而有优势——大资金会"动 market"，小资金不会
- **regime 切换**：散户可以快速从做市切换到观望；机构有最小报价义务（market maker agreement）

**2.4 散户做市的 3 条生路**

1. **regime-aware 慢做市**：只在 regime 稳定（震荡市）+ spread 厚 + 波动率适中时挂单；regime 切换时立即退出。**本质上是"用时间换 spread"**——慢做市的 fill rate 低但 fill 后被逆向选择的概率也低。
2. **maker 返佣套保**：接 80% 返点（Gate）+ 同步在另一腿用 funding arb 对冲 delta。**本质上是把"做市的库存风险"外包给对冲腿**——做市赚 spread + 返佣，对冲腿赚 funding，组合 Sharpe 提升。
3. **新品种 / 新上线币做市**：在 spread 厚（0.1-1%）+ 深度浅（机构不愿进场）的品种挂单。**本质上是"避开与 HFT 的正面竞争"**——做机构不愿做的品种，赚更高的 spread。

**2.5 散户做市的"决策反演"清单**

承自《反演思维与第一性原理》的"避开失败"思路：

| 反演问题 | 对应行动 |
| --- | --- |
| 我做市最可能在哪种情况下亏光？ | 单边行情 + 库存积累 + 无对冲 → 必须 regime 检测 + 库存上限 + 同步对冲 |
| 我比机构慢多少？ | 50ms+ → 不能挂 best bid/ask，必须挂在 mid ± 较远档（接受低 fill rate） |
| 我的 spread 覆盖得了成本吗？ | taker 0.05% × 2 + slippage ≥ 0.04% → spread ≥ 0.15% 才能稳定盈利 |
| 机构在什么情况下不会来？ | 小币、新上线币、深度浅的所 → 散户的"安全区" |
| 我能承担的最大 inventory 偏离？ | 单币 ≤ 25% 总资金 × γ 校准 → inventory_skew 严格上限 |

---

### 第二部分　Avellaneda-Stoikov 模型核心（必含）

#### 第三章　AS 2008 核心公式（bid/ask 偏离 + 库存修正）

**3.1 原始论文与基本设定**

论文：Avellaneda, M. & Stoikov, S. (2008) *High-frequency trading in a limit order book*. Quantitative Finance 8(3): 217-224。**这是现代做市策略的奠基性论文**——把"做市"从经验性的"挂单等成交"升级为"基于随机控制的数学优化"。

承自 Avellaneda-Stoikov 2008 原文（ResearchGate PDF 可访问）：

- **中间价动态**：`dS_t = σ dW_t`（算术布朗运动，无漂移；选择算术而非几何是数学方便性，因为 inventory penalty 在算术 BM 下保持有界）
- **做市商目标**：最大化终值时刻的指数效用（CARA 偏好）
  ```
  u(s, x, q, t) = max E[-exp(-γ(X_T + q_T S_T))]
  ```
  其中 X 是 cash、q 是 inventory（股票数量）、S 是中间价
- **订单到达模型**：挂单距离 δ 时，订单到达是强度 λ(δ) = A·exp(-κδ) 的 Poisson 过程（实证支持：Potters & Bouchaud 2003）

**3.2 解析解（假设 λ(δ) = A·exp(-κδ)）**

承自 Avellaneda-Stoikov 原文 Eq. (3.10)-(3.12)：

**Reservation price**（库存中性化的"目标价"）：
```
r(s, q, t) = s - q·γ·σ²·(T - t)
```

**最优 bid/ask spread**（围绕 reservation price 的对称 spread）：
```
δ_a = δ_b = (1/γ) · ln(1 + γ/κ)
```

**最优报价**：
```
bid = r - δ/2 = s - q·γ·σ²·(T-t) - (1/2γ)·ln(1 + γ/κ)
ask = r + δ/2 = s - q·γ·σ²·(T-t) + (1/2γ)·ln(1 + γ/κ)
```

**3.3 公式直觉（最重要的"为什么"）**

**库存修正项**：`q·γ·σ²·(T-t)` —— 库存 q 越大、波动率 σ 越高、剩余时间 (T−t) 越长，reservation price 偏离 mid 越多。

- 库存为正（持币多）→ reservation < mid → 报价整体下移（更想在 mid 之下卖）
- 库存为负（持币少/做空）→ reservation > mid → 报价整体上移（更想在 mid 之上买）
- 终值时刻 (T−t → 0) → 库存修正项 → 0 → reservation 接近 mid（避免未平仓库存跨期）

**Spread 项**：`(1/γ)·ln(1 + γ/κ)` —— 与 σ 无关（因为假设中间价是算术 BM、无漂移），只与 γ 和 κ 相关。

- γ ↑ → spread ↑（更保守；fill rate 下降但被逆向选择的概率也下降）
- κ ↑ → spread ↓（市场活跃时 spread 可收窄，因为到达概率高、库存暴露窗口短）
- γ → 0 → spread → 0（无风险厌恶 → 极限情况是无限窄的 spread，但这不是现实）

**3.4 "symmetric"基准对照**

承自 AS 原文 Table 1：

| 策略 | Spread | Profit | std(Profit) | Final q | std(Final q) |
| --- | --- | --- | --- | --- | --- |
| **Inventory（AS）** | 1.29 | 62.94 | 5.89 | 0.10 | 2.80 |
| Best bid/best ask | 0.54 | 48.43 | 14.57 | 0.72 | 9.56 |
| Symmetric | 1.29 | 67.21 | 13.43 | -0.01 | 88.66 |

- **Inventory 策略**：spread 1.29、profit 中等、std 极小、final q 几乎为 0、std(q) 极小
- **Best bid/best ask**：spread 0.54（薄）、profit 较高、std 大、final q 不稳定
- **Symmetric**：spread 1.29（与 Inventory 同）、profit 略高、std 大、final q 接近 0 但 std 极大

**关键洞察**：Inventory 策略的 std(Profit) 比 Best bid/best ask 小 2.5 倍，比 Symmetric 小 2.3 倍——**用相同的 spread 赚了更高的 Sharpe**。库存偏斜的"看不见的价值"是它把 inventory 变成可控变量，不是去赌 inventory 方向。

**3.5 与 AS 原文的仿真参数**

承自 AS 2008 §3 仿真：`T=1, σ=2, dt=0.005, q=0, γ=0.1, k=1.5, M=0.5`，最优 spread δ_a + δ_b = 1.29。**这套参数对应的库存惩罚远大于 spread 成本**——γ=0.1 + σ=2 + (T−t)=1 → 单边 inventory penalty = 0.4，远大于 spread 0.645 的一半。

---

#### 第四章　参数 γ / σ / κ / (T−t) 的直觉

**4.1 γ（风险厌恶系数）**

- **物理意义**：每 1 USDT 库存对应多少 USDT 的终值厌恶
- **典型量级**：Hummingbot AS strategy 默认 γ 在 0.1-1.0 之间（无量纲）；机构做市商可能 0.01-0.5（取决于资本规模 + 风险预算）
- **γ ↑ 的影响**：spread ↑（更保守）、inventory penalty ↑（库存修正更激进）、P&L 波动 ↓、fill rate ↓
- **γ ↓ 的影响**：spread ↓、inventory penalty ↓、P&L 波动 ↑、fill rate ↑
- **散户参数起点**：γ = 0.1 是常见的"安全默认值"；加密小币用 0.05-0.2，大币用 0.1-0.5

承自 Hummingbot `avellaneda_market_making` 文档："The higher the value, the more aggressive the strategy will be to reach the inventory_target_base_pct, increasing the distance between the Reservation price and the market mid price."

**4.2 σ（波动率）**

- **物理意义**：中间价的瞬时波动率（年化或时间窗口标准化）
- **典型量级**：BTC 1 小时 σ 约 0.5-1%（年化 60-80%）；ETH 类似但略高
- **σ ↑ 的影响**：spread 项不变（因为算术 BM 模型无 σ），但 inventory penalty ∝ σ² 显著 ↑→ 报价偏斜更激进
- **σ ↓ 的影响**：spread 项不变、inventory penalty ↓ → 报价接近 symmetric
- **散户用法**：用过去 N 天（20-60）的滚动波动率做实时 σ 输入；σ 跳升时（regime 切换信号）应暂停做市或扩大 γ

**4.3 κ（订单到达强度衰减系数）**

- **物理意义**：挂单离 mid 越远，到达强度衰减越快
- **典型量级**：Hummingbot 文档默认 1.5-2.0；不同品种/不同 spread regime 不同
- **κ ↑ 的影响**：挂单离 mid 越远 → 到达强度衰减越快 → spread 必须收窄才能保持 fill rate
- **κ ↓ 的影响**：挂单离 mid 较远时仍有较高到达强度 → spread 可放宽
- **实证标定**：用过去 7-30 天的 order arrival 数据，按价格档位统计 λ(δ) 并拟合 λ = A·exp(-κδ)

**4.4 (T − t)（剩余 horizon）**

- **物理意义**：做市窗口剩余时间
- **典型量级**：机构 HFT 几秒到几分钟；散户 1 小时到 1 天（不重新平衡过夜）
- **(T−t) ↓ 的影响**：inventory penalty ↓ → reservation 接近 mid → 报价对称化（避免未平仓库存跨期）
- **(T−t) ↑ 的影响**：inventory penalty ↑ → 报价偏斜更激进
- **加密特殊性**：24/7 无自然 horizon；散户可设"日内 horizon"（如 4-8h 重置）；机构可能分钟级或秒级

**4.5 参数的"加密标定"经验值**

| 参数 | 加密大币（BTC/ETH） | 加密中币（SOL/AVAX） | 加密小币/新币 |
| --- | --- | --- | --- |
| γ | 0.1-0.3 | 0.05-0.15 | 0.01-0.05 |
| σ（年化） | 60-100% | 80-150% | 100-300% |
| κ | 1.0-2.0 | 0.8-1.5 | 0.3-1.0 |
| (T−t) | 1-4h | 1-4h | 1-2h |

**说明**：以上为经验估计（承自 Hummingbot 文档 + LLMQuant/quantbeckman 解读），实际需要按品种 + regime + 时段动态标定。

---

#### 第五章　库存管理的核心思想

**5.1 为什么"库存 = 0"不是目标**

直觉上"库存 = 0 最安全"，但如果做市商严格保持 q=0，等于不做市（成交的两侧立即被同一笔 taker 抵消）。**做市策略的目标是"库存稳定在某个目标区间 + 偏离时主动修正"**——Hummingbot `inventory_target_base_pct = 50%`（币种占比 50%）是这种思想的工程化。

**5.2 AS 模型的库存管理三件套**

1. **报价偏斜（Quote Skew）**：库存为正 → reservation 下移 → bid/ask 整体下移 → 更想在 mid 之下卖（让 fill rate 偏向"卖"侧）
2. **报价宽度（Spread Width）**：库存偏离越大 → γ·σ²·(T−t) 项越大 → spread 越宽（更保守，因为库存暴露风险 ↑）
3. **强制平仓（Inventory Liquidation）**：当 q 超过 q_max → 用 market order 强平（吃掉 spread 但消除库存风险）

**5.3 "慢 vs 快"库存管理的对比**

| 维度 | 慢库存管理（AS / 偏斜报价） | 快库存管理（market order 强平） |
| --- | --- | --- |
| 机制 | 调整 bid/ask 距离让 fill rate 偏向逆库存方向 | 直接 taker 市价强平 |
| 成本 | 0（不跨 spread） | spread × 2（吃 spread + 库存被强平） |
| 时效 | 慢（依赖 fill rate 自然调整） | 即时 |
| 适合场景 | 库存偏离 < q_max/2 | 库存偏离 ≥ q_max/2 或 regime 切换 |

**散户策略**：
- 库存偏离 ≤ 25% 总资金 → 用 AS 模型慢调整
- 库存偏离 ≥ 25% 总资金 → 触发"用 funding arb 对冲"（不是 taker 强平，是开 funding arb 腿对冲 delta）
- 库存偏离 ≥ 50% 总资金 → 触发熔断 + taker 强平 + 全停人工复盘

**5.4 Hummingbot inventory_skew 的工程实现**

承自 Hummingbot 文档：

```
inventory_target_base_pct = 50%    # 目标币种占比
inventory_skew_enabled = True      # 启用库存偏斜
inventory_range_multiplier = 1.0   # 偏斜强度（multiplier 越大，偏离目标时调整越激进）

if 当前 base 占比 > target：
    卖出订单放在更靠近 mid 的位置（fill rate ↑）
    买入订单放在更远离 mid 的位置（fill rate ↓）
    直到 base 占比回到 target

if 当前 base 占比 < target：
    反向调整
```

**5.5 "inventory_target_base_pct"与 AS 模型的关系**

- Hummingbot 的 inventory_target = 工程化近似 AS 的"目标 inventory"
- AS 模型：reservation price 随 q 线性偏斜（连续）
- Hummingbot：fill rate 阶梯式偏斜（离散，订单位置非连续调整）
- 精度：Hummingbot 比 AS 模型更粗糙，但工程上更稳定（参数少、不易过拟合）

**5.6 库存管理的"反例"：被动做市 = 库存无控积累**

Hummingbot pure_market_making 在 `inventory_skew_enabled = False` 时 = 经典被动做市 = 库存暴露完全交给运气。Reddit 案例"跑了一周被一次暴跌打爆"正是这种情况——单边行情时库存持续累积，max drawdown 难以预测。

---

### 第三部分　扩展模型

#### 第六章　Guéant-Lehalle-Fernandez-Tapia 2013 扩展

**6.1 GLFT 论文与基本思想**

论文：Guéant, O., Lehalle, C.-A., & Fernandez-Tapia, J. (2013) *Dealing with the Inventory Risk. A solution to the market making problem*. arXiv 1105.3115；后续期刊版 *Mathematics and Financial Economics* 7(4)。

承自 arXiv 1105.3115 abstract："A solution to the market making problem"（被引 350+）。GLFT 在 AS 基础上做了三个关键扩展：

1. **多资产做市（Multi-Asset）**：用 mean field 处理多个做市商之间的策略互动
2. **更现实的订单到达模型**：λ(δ) = A·exp(-κδ)（与 AS 相同），但加入了"市场已存在的最优档"的到达率修正
3. **闭式解（Closed-Form Solution）**：给出在多资产情况下 bid/ask 的解析形式（不是数值解）

**6.2 GLFT 的核心扩展**

承自 GLFT 2013 §3-§4：

- **Quote Skew 公式**：
  ```
  δ_b^A = δ_0 + q · γ · Σ_A · (T-t)
  δ_a^A = δ_0 - q · γ · Σ_A · (T-t)
  ```
  其中 Σ_A 是资产 A 的波动率（用协方差矩阵替代 AS 的单一 σ²），δ_0 是基础 spread
- **多资产对冲**：把 inventory 风险用 cross-asset covariance 矩阵分解——做市资产 A 的 inventory 风险可用资产 B、C 同步对冲
- **应用场景**：外汇做市（多币对 cross-hedge 是常态）；加密做市（多币对 inventory hedge）

**6.3 GLFT 在加密做市的应用潜力**

- **BTC/ETH 跨库存对冲**：BTC inventory 偏离时，用 ETH 腿做 cross-hedge（BTC-ETH 相关性 0.5-0.9 区间，covariance Σ 提供 hedge ratio）
- **多币对 spread**：BTC/SOL/AVAX 同时做市时，用 GLFT 联合优化（避免单币对 inventory 积累时另一币对未调整）
- **工程实现**：hftbacktest 库提供 GLFT 实现（教程：hftbacktest.readthedocs.io "Guéant-Lehalle-Fernandez-Tapia Market Making Model and Grid Trading"）

**6.4 GLFT vs AS 的工程取舍**

| 维度 | AS 2008 | GLFT 2013 |
| --- | --- | --- |
| 复杂度 | 低（单资产 + 单一 σ） | 中（多资产 + 协方差矩阵） |
| 数据需求 | 单一品种历史价格 | 多品种历史价格 + covariance |
| 闭式解 | 有（指数到达率假设下） | 有（同样假设下） |
| 工程友好 | 友好（Hummingbot 默认实现） | 中等（需自研或 hftbacktest） |
| 适合场景 | 单品种做市起步 | 多品种 inventory hedge |

---

#### 第七章　Cartea-Jaimungal-Penalva 教材 + Bayesian 做市

**7.1 Cartea-Jaimungal-Penalva 2015 教材**

教材：Cartea, Á., Jaimungal, S. & Penalva, J. (2015) *Algorithmic and High-Frequency Trading*. Cambridge University Press。第 6-7 章系统展开 HFT 模型（含 AS 模型扩展 + 多资产 + 路径依赖 + 信息驱动）。

承自教材目录与二手解读（**未直接读原书**）：

- 第 6 章：单一资产 HFT（AS 模型 + 信息驱动变体）
- 第 7 章：多资产 HFT（含 cross-impact、cross-hedge）
- 第 8 章：路径依赖与最优停时（optimal stopping）

**7.2 Bayesian 做市（Han-Fang 2020）**

承自 arXiv 2508.20225（"Optimal Quoting under Adverse Selection and Price Reading"）综述 + Han-Fang 2020 思想：

**核心思想**：传统 AS 模型假设订单到达是同质 Poisson 流；Bayesian 做市把订单按"对手方类型"分类（informed / uninformed），用 Bayesian filter 估计"对手方是 informed 的概率"，并据此调整报价。

**机制**：
```
P(对手方 informed | 订单到达) ∝ P(订单到达 | 对手方 informed) · P(对手方 informed)
```

**实务难点**：
- "对手方是 informed" 的先验难以估计
- 订单簿本身包含信息（OBI、flow imbalance）但难量化

**7.3 信息驱动的做市模型**

承自 arXiv 2508.20225："the market maker faces the risk of being hit at the bid just before the market goes down — the so-called winner's curse"。

- **winner's curse**：做市商被成交 → 接下来价格反向 → 库存暴露损失
- **skew sniffer**：观察做市商的 bid/ask 偏斜 → 推断做市商 inventory → 推动价格让做市商更亏
- **缓解方法**：① 不要让偏斜过于明显（"linear skew"的反 sniff 设计）② 多品种 cross-hedge 让单一品种的偏斜难以推断

---

#### 第八章　强化学习做市 + 实战修正

**8.1 RL 做市的兴起**

承自 MDPI 2021 综述（Gašperov et al. "Reinforcement Learning Approaches to Optimal Market Making"）：RL 在做市领域的应用已显著超越标准 AS 模型（风险调整后回报）。

**典型框架**：
- 状态：`S = {inventory, midprice, spread, depth, volatility, time_remaining}`
- 动作：`A = {(δ_b, δ_a)}`（bid/ask 距离）
- 奖励：`R = spread_captured - γ · inventory_variance`
- 算法：DQN / PPO / SAC（在 HFT 模拟器中训练）

**承自 arXiv 2307.01814 "Option Market Making via Reinforcement Learning" + arXiv 2205.08936 "Market Making via Reinforcement Learning in China Commodity Market"**：RL 在期权做市和商品做市均取得实证优势。

**8.2 RL 做市的现实难题**

- **样本效率**：训练需要大量 tick 数据；加密 L3 数据公开的极少
- **过拟合风险**：模拟器和实盘的差距（regime 切换、流动性事件）导致 RL 策略泛化差
- **regime 切换**：RL 策略在训练 regime 内表现好，遇到 regime 切换可能直接失效
- **执行约束**：tick size、queue position、cancel/replace 失败等工程约束难在 RL reward 中建模

**8.3 实战修正（必须加到 AS/GLFT/RL 之上）**

承自 law-purdue 2015 博士论文（"A Pure-Jump Market-Making Model for High-Frequency Trading"）+ Lu 2019（arXiv 1903.07222）+ Reddit/quant 社区经验：

| 修正项 | 原因 | 修正方式 |
| --- | --- | --- |
| **Tick Size 限制** | AS 模型假设连续价格；实际 tick 是离散的 | 把 bid/ask 舍入到最近 tick |
| **Queue Position** | 挂单前面的量决定 fill rate | 加 queue position 修正到 λ(δ) |
| **Cancel/Replace 失败** | 交易所拒单（rate limit / 余额不足） | 实际下单逻辑加 retry + 对账 |
| **延迟的影响** | 1ms vs 50ms 差距巨大 | latency-adjusted fill rate |
| **方向一致性** | mid 上移时挂单在原位置很少成交 | 挂单跟随 mid 但 tick 限制下 keep priority |
| **Rebate 实际到账** | 返佣以 GT/Point 形式，需二级市场变现 | rebate_rate 折算 + 加入成本模型 |

**8.4 "工程修正后"AS 模型的真实表现**

承自 law-purdue 2015 §6 numerical illustration + Lu 2019 §5：
- 纯 AS 模型（理论）vs 加 L3 修正的"weakly consistent"模型：profit 可能被**高估 50%+**
- 这正是 AS 模型在实盘"装上不等于赚"的根因——Hummingbot 用户的"一分钱没赚到"反映的不是策略错，是模型对现实的简化过强

---

### 第四部分　加密做市的特殊性（必含）

#### 第九章　机构做市商模式（Wintermute / Jump / GSR / Cumberland）

**9.1 机构做市商的"四件套"能力**

承自 crypticweb3 "Best Crypto Market Makers in 2026" + c-sharpcorner 2026 综述 + Hyrotrader Crypto Market Makers Guide：

| 机构 | 规模/覆盖 | 核心模式 |
| --- | --- | --- |
| **Wintermute** | 50+ 交易所、日交易量 22 亿美元（2019） | 双边做市 + OTC + 自营投资 |
| **Jump Crypto** | Jump Trading 加密分支、低延迟基础设施 | HFT + 持续报价 + 多策略（做市 + arb） |
| **GSR** | 自 2013 年运营、强 OTC + 衍生品 | 现货 + 衍生品做市 + 投顾 |
| **Cumberland (DRW)** | 老牌 TradFi 衍生品做市商、加密分支 | OTC + 交易所做市 |
| **Amber Group / Keyrock / DWF Labs / Flowdesk / Kronos** | 2026 年活跃机构 | 多组合模式 |

**9.2 机构 vs 散户做市的"做生意"视角**

承自《自动量化加密货币的成功与失败》§4：

> **启示：散户做不了他们的速度，但可以学他们的生意结构——确定性小钱（spread/返佣/资金费率）优先于方向性大钱。**

机构做市的生意结构 = **收服务费**（spread + 返佣） + **库存折价**（AS 模型自然结论），不是赌方向。散户可学的：
1. **不赌方向**：把库存严格控制在 0 附近
2. **收服务费**：spread + 返佣是确定性的（不像 alpha）
3. **对冲优先**：用 funding arb 把库存暴露外部对冲

**9.3 机构做市的"压力行情"行为**

承自 Multicoin Capital 2020 复盘 "March 12: The Day Crypto Market Structure Broke"：
- 压力行情下做市商会**大幅加宽 spread**（BTC 从 <10bps 跳到 >1000bps）
- 部分做市商**完全撤单**（不提供流动性），等波动率稳定后再恢复
- 这正是 AS 模型在 regime 切换时的自然行为——γ 校准对 regime 切换不敏感时，σ 跳升导致 spread 跳升

**9.4 加密独有的"做市商监管/认证"**

- 部分国家对做市商有牌照要求（如欧盟 MiCA、新加坡 MAS）
- 加密项目方会与做市商签 market maker agreement（含最小报价义务、最大 spread 上限等）
- 散户做市不涉及这些，但应避免在受监管品种做市（合规风险）

---

#### 第十章　散户做市工具（Hummingbot 等）+ 慢做市的真正定位

**10.1 Hummingbot 的核心策略**

承自 Hummingbot 官方文档：

| 策略 | 机制 | 适合散户吗？ |
| --- | --- | --- |
| **pure_market_making (PMM)** | 挂 best bid/ask + 可选 inventory_skew | 部分适合（必须启用 skew） |
| **cross_exchange_market_making** | 在做市所挂单 + 在对冲所对冲 | 适合（同所 funding arb 对冲也是变体） |
| **avellaneda_market_making** | AS 模型 + inventory_target_base_pct | 适合（散户最推荐的工程化实现） |
| **perpetual_market_making** | 永续合约做市 + 可选 hedging | 适合（项目 Phase 2 可用） |
| **liquidity_mining / amm-arb** | 给 DEX 池提供流动性 / AMM 套利 | 不属于做市范畴 |

**10.2 Hummingbot 的工程配置最佳实践**

承自 Hummingbot Academy "What is Market Making" + inventory_skew 文档：

```
# 慢做市 + 库存偏斜的 PMM 配置示例
bid_spread = 0.5%       # bid 距 mid 的距离
ask_spread = 0.5%       # ask 距 mid 的距离
inventory_skew_enabled = True
inventory_target_base_pct = 50      # 目标币种占比 50%
inventory_range_multiplier = 1.0    # 偏斜强度（multiplier 越大越激进）
order_amount = 0.001 BTC            # 单笔挂单量（小资金！）
order_refresh_time = 60s            # 挂单刷新周期（慢做市关键）
cancel_order_respects_queue_position = False  # 不抢 queue position（散户不应该抢）

# Avellaneda 策略示例
avellaneda_market_making:
  risk_factor = 0.1                  # γ（风险厌恶）
  order_amount = 0.001 BTC
  inventory_target_base_pct = 50
  filled_order_delay = 60s           # 成交后延迟再挂（防被立即反向）
```

**10.3 Hummingbot 在加密的"装上不等于赚"现象**

承自 Reddit 案例 + Hyrotrader 调研：

- 多数 Hummingbot 用户"装上后一分钱没赚到"——根因是参数未标定、fill_rate 太低、或被机构 HFT 抢先成交
- "装上就赚"的早期红利已消失（2019-2021），现在 Hummingbot 用户必须参数调优才能有正期望
- "regime-aware + 库存偏斜 + 慢刷新"是当前能跑赢的关键三件套

**10.4 慢做市的"三不原则"**

1. **不挂 best bid/ask**：散户 50ms 延迟下必然被逆向选择，必须挂 mid ± 0.5% 较远档
2. **不频繁刷新**：order_refresh_time ≥ 30s（避免 cancel/replace 暴露 + 节省 API 配额）
3. **不跨 spread**：成交后立即反向挂新单（避免 inventory 暴露）

---

#### 第十一章　加密 maker 返佣的算账 + 失败模式

**11.1 maker 返佣的真正经济账**

承自《市场微观结构与滑点建模》§5 + Gate.io 官方费率：

| 交易所 | Maker 基础费率 | Maker 80% 返点后 | Taker 基础费率 | Taker 80% 返点后 |
| --- | --- | --- | --- | --- |
| Gate.io | 0.02% | **0.004%** | 0.05% | 0.01% |
| Binance | 0.02% | 0.015%（BNB 25%） | 0.05% | 0.0375% |
| OKX | 0.02% | -0.005%（VIP4+） | 0.05% | 0.015%（VIP4+） |

**做市策略的实际成本**（单笔往返）：
- 挂 maker 单成交：每笔 0.004% × 2 = **0.008%**
- 加上挂单 → 等待 → 成交的机会成本 → 实际"全成本"约 0.01-0.02%/次往返

**盈亏平衡 spread**：
- 无返点：spread ≥ 0.05%/次往返才能打平 taker 成本
- Gate 80% 返点：spread ≥ 0.01%/次往返即可
- OKX VIP4+ maker 负费率：理论上 spread ≥ 0 即可（但 spread = 0 意味着不提供流动性）

**11.2 散户做市必须 ≥ 多少 spread 才能稳赚？**

经验公式：
```
做市最小 spread = 2 × effective_fee + 2 × fill_rate_adjusted_slippage + inventory_penalty + safety_buffer
              ≈ 0.02% + 0.02% + 0.05% + 0.05%
              ≈ 0.14% /次往返
```

**含义**：
- BTC/ETH（spread 典型 0.01-0.05%）：散户做市几乎无法稳定盈利（除非 maker 返佣极高 + 不被逆向选择）
- 中币 SOL/AVAX（spread 0.05-0.2%）：散户可以做（但 depth 浅、fill rate 低）
- 小币/新上线币（spread 0.2-1%）：散户的优势区（机构不愿进场、散户 spread 足够大）

**11.3 加密做市的 4 类典型失败模式**

承自《自动量化加密货币的成功与失败》§3.4 Hummingbot 案例 + Reddit 散户复盘：

| 失败模式 | 机制 | 防御 |
| --- | --- | --- |
| **单边行情被反复止损** | 网格在单边市反复触发，库存累积 | regime 检测 + 区间外失效机制 |
| **库存积累无法对冲** | 持续单边成交导致 inventory 单边扩大 | inventory 上限 + 强制 funding arb 对冲 |
| **跨所基差扩大时套保失效** | 跨所对冲时基差瞬间扩大 | 单所做市（避开跨所基差风险） |
| **spread 收紧时被动做市无利可图** | 主流币 spread 持续窄于成本 | regime 退出机制 + 切换到 maker 返佣套保 |

**11.4 Hummingbot Reddit 案例的具体分析**

承自《自动量化加密货币的成功与失败》第十章 案例 12："Hummingbot 做市（Reddit）—— '一分钱没赚到'——做市盈利依赖费率档/流动性/价差配置，装上≠赚"。

**根因诊断**：
1. 选了 spread 太薄的品种（BTC/ETH，spread 0.01-0.05%）
2. 启用了 inventory_skew 但 target_base_pct = 50% 不适合小资金（小资金应偏向 USDT 而非 BTC）
3. 没启用 funding arb 对冲（库存暴露无外部对冲）
4. 没启用 regime 退出机制（regime 切换时仍挂单）

**修复方案**：
1. 换到中币/小币（SOL/AVAX 或新上线币）
2. inventory_target_base_pct = 30%（小资金更偏向 stable）
3. 启动 funding arb 对冲（用 funding_arb 腿对冲做市库存）
4. 加 regime 检测 + 波动率阈值退出

---

### 第五部分　做市的风险度量（必含）

#### 第十二章　库存风险 + VaR/CVaR 度量

**12.1 库存风险的定义**

承自《风险量化 VaR/CVaR/ES》§7 + AS 模型 + 加密做市实践：

**库存风险 = inventory × price volatility 的尾部暴露**。
具体：
```
inventory_var_95% = inventory_value × 1.65 × daily_vol
inventory_cvar_95% = inventory_value × 2.06 × daily_vol  # 正态假设下
```

**加密市场的特殊性**：BTC 日收益率峰度 20+（vs 正态 3），参数 VaR 严重低估——**至少用历史 VaR 或 EVT**。

**12.2 库存 VaR 的工程估算**

承自《风险量化 VaR/CVaR/ES》§1.4 + §6：

```
# 历史 VaR（30 天）
inventory_var_95 = inventory_value × historical_5th_percentile_daily_return
inventory_cvar_95 = inventory_value × mean_of_returns_below_5th_percentile

# EVT 估算（拟合 Generalized Pareto Distribution）
fit GPD to inventory P&L below 95th percentile threshold
inventory_cvar_99 = threshold + (GPD_shape × threshold) / (1 - GPD_shape)
```

**项目 L1 熔断线的反向校准**：
- 100 USDT 本金 × inventory_limit_25% × BTC daily_vol_5% = 1.25 USDT 日亏熔断
- 但实际 inventory_limit 应更紧（如 10%）→ 0.5 USDT 日亏熔断
- 与现有日亏 -2% 熔断（2 USDT）一致 → inventory 上限 25% 是合理但偏紧

**12.3 "inventory 在压力时段" 的非线性跳升**

承自 2025-10-10 闪崩案例（FTI Consulting 复盘）：

- 正常时段：BTC 日 σ ≈ 3-5%
- 压力时段：BTC 日 σ 跳到 15-30%（4-6 倍）
- inventory_cvar_95 同步跳升 4-6 倍

**含义**：常态校准的熔断线（-2% 日亏）在压力时段触发概率远高于预期——**做市策略必须 regime-aware 调整 inventory 上限**。

---

#### 第十三章　逆向选择 + Pin Risk + 取消单风险

**13.1 逆向选择（Adverse Selection）**

承自 arXiv 2508.20225 "Optimal Quoting under Adverse Selection and Price Reading" + Glosten-Milgrom 1985：

**定义**：知情交易者（informed trader）先下单 → 做市商被动成交 → 接下来价格反向移动 → 做市商库存暴露损失。

**经典量化**：
```
adverse_selection_cost = fill_rate × E[price_move_after_fill | fill occurred]
                      ≈ fill_rate × σ × √Δt_after_fill
```

**做市商的"winner's curse"**：每一笔成交 = 50% 概率被逆向选择 → 平均下来 fill 一半是亏的。

**13.2 Glosten-Milgrom 模型**

承自 Glosten & Milgrom 1985 JFE：

```
quoted_spread = 2 × P(informed) × (E[V|H] - E[V|L])
```

其中 V 是资产真实价值，H 是 high 信号（informed 知情），L 是 low 信号（informed 不知情）。

**含义**：spread 是"做市商对 informed 比例的防御"——P(informed) ↑ → spread ↑。

**13.3 skew sniffer 攻击**

承自 arXiv 2508.20225 §1：

> "skew sniffers detect this behavior and push the price in a direction unfavorable to the market maker, thereby amplifying inventory risk"

**机制**：观察做市商的 bid/ask 偏斜 → 推断做市商 inventory → 推动价格让做市商更亏。

**防御**：
1. 多品种 cross-hedge 让单一品种偏斜难以推断
2. 不要让偏斜过于明显（"linear skew"是已知最易被 sniff 的）
3. 引入 noise（故意小幅随机偏斜）让 skew 推断变难

**13.4 Pin Risk（期权做市）**

承自《市场微观结构与滑点建模》相关章节 + 加密期权做市讨论：

**定义**：到期日附近，做市商可能被 pin 在某一价位（期权到期日所有持仓按结算价交割），导致 inventory 暴露集中。

**加密期权做市的特殊性**：Deribit 等期权所到期日（每月最后一个周五）流动性集中 → 做市商在到期日附近的 inventory 暴露剧增。

**散户相关性**：散户不做期权做市，但应在到期日附近**减少现货做市的 inventory 上限**（pin risk 类比）。

**13.5 取消单风险（Cancel Risk）**

**定义**：挂单后被交易所 cancel（如维护、风险控制、余额不足）但反向单已填。

**典型场景**：
- 交易所临时维护 → 挂单被 cancel，库存暴露
- API rate limit → 撤单请求未发出，库存暴露
- 余额检查延迟 → 挂单时余额够、成交时余额已被另一笔占用

**防御**：
- 启用 Hummingbot `cancel_order_respects_queue_position = True`（尽量不主动撤单）
- 启用 cancel hysteresis（成交后延迟 30s 再撤反向单）
- 启用 cancel 失败告警（飞书推送）

---

#### 第十四章　做市 P&L 的马丁签名诊断

**14.1 做市 P&L 的典型形状**

承自《自动量化加密货币的成功与失败》第十章 + 做市策略实证：

- **正常时期**：P&L 是"小额频繁盈利"（每天 +0.001-0.01 USDT 量级）
- **压力时期**：偶发"单笔大亏"（-0.5 到 -2 USDT 量级，对应 inventory 暴露 + 单边行情）
- **胜率**：高（80-95%）
- **盈亏比**：低（盈利 0.01 vs 亏损 0.5-2）
- **年化 Sharpe**：0.5-2（vs 费率套利的 2-3）

**这正是"马丁签名"**——平滑小赢曲线 + 罕见巨亏。

**14.2 马丁签名的代码化检测**

承自《自动量化加密货币的成功与失败》第十章 + 自动诊断算法：

```
def detect_martingale_signature(trade_history):
    # 1. 计算胜率
    win_rate = sum(t > 0 for t in trade_history.pnls) / len(trade_history.pnls)
    
    # 2. 计算盈亏比
    avg_win = mean([t for t in trade_history.pnls if t > 0])
    avg_loss = abs(mean([t for t in trade_history.pnls if t < 0]))
    payoff_ratio = avg_win / avg_loss
    
    # 3. 马丁签名检测
    martingale_signals = {
        'win_rate_high': win_rate > 0.7,        # 高胜率
        'payoff_ratio_low': payoff_ratio < 0.3, # 低盈亏比
        'max_loss_extreme': max_loss > 5 * avg_win,  # 单笔最大亏 > 5 倍平均盈利
    }
    
    if all(martingale_signals.values()):
        return "MARTINGALE_SIGNATURE_DETECTED"
    else:
        return "OK"
```

**14.3 做市策略的反马丁设计**

| 反马丁设计 | 机制 |
| --- | --- |
| **强制 inventory 上限** | inventory 偏离 0 超过 25% → 触发 funding arb 对冲或 taker 强平 |
| **强制 spread 上限** | 压力时段 spread 必须扩大到 N × normal_spread，否则不挂单 |
| **强制 regime 退出** | regime 切换（高波动 / regime 检测触发）→ 立即撤所有挂单 |
| **强制单日 P&L 熔断** | 日亏 -2% → 全停人工复盘 |
| **强制时间衰减** | (T−t) → 0 时自动收窄 spread，避免未平仓库存跨期 |

---

### 第六部分　项目应用映射（必含）

#### 第十五章　100 USDT 项目的做市定位（Phase 1 不上 / Phase 2 实验 / Phase 3 升级）

**15.1 Phase 1（起步）：做市不上**

承自《自动量化项目架构初稿》L5 阶段 1 + 本条目 §11.2 算账：

**理由**：
1. **BTC/ETH spread 太薄**：0.01-0.05% < 0.14% 散户最低 spread → 散户做市 BTC/ETH 必然亏
2. **费率套利更稳**：年化 8-15%（牛市）vs 做市的 -5% 到 +5%（年化，依赖 regime）
3. **项目目标是"全链路工程验证"**：费率套利足够验证数据/风控/执行/监控全链路
4. **资金门槛**：100 USDT 做市必须 ≥ 5000 USDT 才有感知收益（参见 §11.2）

**15.2 Phase 2（中期）：实验性做市（5-10 USDT）**

**触发条件**：
1. 费率套利 Phase 1 稳定运行 ≥ 3 个月
2. 日亏熔断未误触发 ≥ 90 天
3. regime 检测模块已上线并稳定

**策略选择**：
- **方案 A（推荐）**：测试网做市（Gate.io testnet），验证 Hummingbot AS 策略
- **方案 B**：小币/中币 maker 返佣套保（SOL/AVAX，spread 0.1-0.3%，加 funding arb 对冲）
- **方案 C**：新上线币做市（高 spread、机构不愿进场）

**资金分配**：5-10 USDT（≤ 10% 总资金）

**15.3 Phase 3（成熟期）：30 USDT 小规模做市**

**触发条件**：
1. Phase 2 实盘 ≥ 3 个月且正期望
2. 累计 inventory 上限熔断触发次数 < 1 次/月
3. 累计 P&L 与回测偏差 < 50%

**策略选择**：
- Hummingbot avellaneda_market_making 主力
- inventory_target_base_pct = 30%（小资金更偏向 stable）
- funding arb 对冲腿同步运行

**15.4 杠铃结构 = 90 USDT 费率套利 + 10 USDT 慢做市**

承自《反脆弱决策清单》"杠铃策略" + 《杠铃配置模板》：

| 仓位 | 策略 | 期望年化 | Sharpe | 最大回撤 |
| --- | --- | --- | --- | --- |
| 90 USDT | 费率套利（确定性收益） | 8-15% | 2-3 | 2-5% |
| 10 USDT | 慢做市实验（不确定性 alpha） | -20% 到 +50% | 0.5-2 | 20-40% |
| **组合** | **凸性暴露** | **5-20%** | **1.5-2.5** | **5-10%** |

**反脆弱特征**：
- 90 USDT 费率套利是"凸性"——大多数时候正收益（小赢），极少数时候小亏
- 10 USDT 慢做市是"探索"——大多数时候小亏/小赢，偶尔大赢（找到新 alpha）
- 组合 = "上端有上界、下端有底"——正不对称

---

#### 第十六章　架构 L5 策略层扩展（P0-3 + P2-C/D）

**16.1 P0-3 数据管道扩展**

承自《自动量化项目架构初稿》P0-3 + 做市策略数据需求：

新增字段：
```
数据管道扩展：
  spread_history：bid/ask spread 每 30s 落库（用于做市策略的 κ 标定）
  depth_history：top 10 档 depth 每 30s 落库（用于做市策略的 inventory 上限校准）
  queue_position：挂单 fill 后记录"前面排队的总量"（用于 fill_rate 修正）
  inventory_history：每笔成交后记录 inventory 偏离 0 的程度
```

**16.2 P2-C（新增）：Hummingbot 集成 + AS 模型实现**

```
P2-C 任务清单：
  - Hummingbot 在服务器部署（独立 venv 或 container）
  - avellaneda_market_making 策略实现（用 Hummingbot 默认或自研）
  - inventory_skew + inventory_target_base_pct 配置（默认 30%）
  - 订单刷新周期 ≥ 30s（慢做市）
  - 实时监控：fill_rate / inventory / spread / P&L
  - 告警：fill_rate < 30% 持续 1h / inventory 偏离 > 25%
```

**16.3 P2-D（新增）：maker 返佣套保策略**

```
P2-D 任务清单：
  - 在做市的同时启动 funding arb 腿（用另一子账户做 funding_arb）
  - 做市库存 > 阈值 → funding_arb 对冲 delta
  - 做市库存 = 0 → funding_arb 也平仓（避免冗余风险）
  - 共享 RiskEngine（funding_arb 和做市共用同一风控配置）
```

**16.4 共享 RiskEngine 的设计**

```
共享 RiskEngine.check(order, strategy_type):
  if strategy_type == "market_making":
    additional_checks = [
      inventory_limit ≤ 25%,                # 做市专用
      spread ≥ 0.14% (散户最低),            # 做市专用
      fill_rate ≥ 30% (过去 1h),            # 做市专用
      regime == "stable",                   # 做市专用
    ]
  if strategy_type == "funding_arb":
    additional_checks = [
      funding_30d_avg ≥ 0.01%,              # 费率套利专用
      ...
    ]
  return base_checks + additional_checks
```

---

#### 第十七章　L1 禁止清单 + L6 监控对应

**17.1 L1 禁止清单的做市对应**

承自《自动量化项目架构初稿》L1 + 本条目：

| L1 禁止项 | 做市对应 |
| --- | --- |
| **禁马丁** | 禁做市 + 反向加仓（库存偏离时不通过"挂更多单"修正，而是 funding arb 对冲或 taker 强平） |
| **单仓 ≤ 25%** | inventory 偏离 0 不超过总资金 25% |
| **单所 ≤ 50%** | 做市多所分散（同所 funding arb 对冲可放宽单所 ≤ 80%） |
| **不追 ≥0.1% 极端费率** | 不在 spread 极端厚时强行做市（spread 厚时往往是 regime 切换） |

**17.2 L6 监控层的做市告警**

承自《自动量化项目架构初稿》L6 + 做市策略：

| 告警项 | 阈值 | 动作 |
| --- | --- | --- |
| 库存偏离 | > 20% 总资金 | 告警 + 触发 funding arb 对冲 |
| 库存方向连续大单 | 5 笔同向成交（被逆向选择） | 告警 + 暂停挂单 |
| Spread 收紧到 maker 不可盈利 | < 0.05%（BTC/ETH） | 告警 + 退出做市 |
| Fill rate 持续低 | < 30% 持续 1h | 告警 + 检查 queue position / regime |
| 交易所维护 / 提币暂停 | 检测公告 | "只平不开"模式 |
| Funding rate 转负 | 30 天均值 | funding arb 自动清仓（做市库存自动暴露） |
| Regime 切换 | HMM 检测 | 立即撤所有做市挂单 |

**17.3 做市专用 L6 监控**

| 监控项 | 计算方法 | 用途 |
| --- | --- | --- |
| 实时 spread | best_ask − best_bid | 做市可行性判断 |
| 实时 depth | top 10 档累计量 | inventory 上限校准 |
| Fill rate | 过去 1h 成交数 / 挂单数 | 策略活性判断 |
| Inventory 偏离 | abs(inventory) × price / total_capital | 风险预算监控 |
| 做市 P&L vs HODL | 做市累计 P&L vs 同段时间持有 P&L | 策略真实价值评估 |

---

### 第七部分　回测方法论（必含）

#### 第十八章　做市回测的 4 个核心难点 + 工具栈 + CPCV 应用

**18.1 做市回测的 4 个核心难点**

承自《回测方法论深化与 CPCV》§7 + 做市策略实证：

1. **订单簿 L3 数据缺失**：CCXT 公开数据多为 L2（top N 档聚合），没有 L3（每笔挂单 order_id）→ queue position 无法精确估算
2. **成交模拟与现实的差距**：回测假设"挂单按 λ(δ) 概率成交" → 实盘有 cancel/replace 失败、queue position 实际占用、对手方优先
3. **取消单的可执行性**：回测假设"想撤就撤" → 实盘有 API rate limit、exchange delay、cancel 失败
4. **信息流的前瞻性偏差**：回测用 mid price 触发决策 → 实盘用 last trade price（可能有 1-2 tick 滞后）

**18.2 必加的回测修正**

```
回测修正清单：
  - queue_position_assumption: top_N_depth_at_price / order_size（估算 fill 概率）
  - latency_assumption: 50ms (散户延迟) → fill rate 折扣 20-50%
  - cancel_replace_failure_rate: 5-15% (API rate limit)
  - maker_rebate_actual_rate: 账户实测（非理论）
  - funding_window_buffer: 结算前后 1min spread × 2
  - regime_exit_logic: regime 切换时立即撤单（不靠回测结束才撤）
```

**18.3 做市回测工具栈**

| 工具 | 适合场景 | 优势 | 劣势 |
| --- | --- | --- | --- |
| **Hummingbot backtest mode** | PMM / cross-exchange / AS 策略 | 集成度高、社区维护 | L2 数据假设、L3 不支持 |
| **hftbacktest (GLFT 教程)** | GLFT 多资产做市 | L3 tick 数据 + 高保真 | 学习曲线陡、数据准备成本高 |
| **自建 L2 tick-by-tick 回放** | 定制策略 | 完全可控 | 工作量大、需自研验证 |
| **多 agent 模拟（RL）** | RL 做市策略 | 可建模对手方博弈 | 模拟器与现实的差距 |

**18.4 CPCV 在做市策略的应用**

承自《回测方法论深化与 CPCV》§7.1 + 做市回测的 regime 依赖性：

- **必要性**：做市 P&L 强 regime-dependent（牛市 vs 熊市 vs 震荡市差异巨大）→ 必须按 regime 分段回测
- **CPCV 实现**：N=6, k=2 的 15 条路径 + 50 次蒙特卡洛（性价比拐点）
- **额外维度**：regime-aware CPCV（每条路径显式标注 regime 标签，输出按 regime 分桶的 P&L）

**18.5 做市回测的"5 关上强度检查"**

承自《回测方法论深化与 CPCV》§7.2 + 做市回测特殊性：

| 关卡 | 做市专用修正 |
| --- | --- |
| ① 未来函数 + purged k-fold | 用 mid + last trade 两种触发，验证差异 |
| ② CPCV PBO/DSR | regime-aware CPCV（按 regime 分桶） |
| ③ 多周期 WFO（rolling + anchored） | 加 (T−t) 滚动窗口 |
| ④ 块状 bootstrap MC | inventory 序列 block_size = 24h（避免日内块打破 spread 时序） |
| ⑤ 加密 RST | 加 BTC -30% + funding 反转 + spread 5x + ADL 触发 |

**任一关 FAIL 不准进入 paper trading**——做市策略在加密 RST 下的脆弱性比费率套利大得多。

**18.6 做市回测的诚实声明**

承自 law-purdue 2015 博士论文 §6：

> "the profit of market-making can be severely overstated under LOBs with inconsistent price movements"

**做市回测的系统性偏差方向 = 过度乐观**。任何做市策略的回测都应：
- 默认打 50% 折（vs 费率套利的 30-50% 折）
- paper trading 周期 ≥ 4 周（vs 费率套利的 2-4 周）
- 小资金实盘周期 ≥ 8 周（vs 费率套利的 4-8 周）

---

## 自测

1. **（AS 模型核心）** Avellaneda-Stoikov 2008 模型的 reservation price 公式是什么？库存为正时 reservation 应该如何偏离 mid？为什么？γ/σ/(T−t) 任一项 ↑ 时 reservation 的偏离如何变化？
2. **（库存管理）** 库存为 +0.5 BTC（持币过多）时，AS 模型会如何调整 bid/ask 报价？Hummingbot inventory_skew 在同样情况下会怎么调？两者机制有何本质区别？
3. **（散户 vs 机构）** 散户做市与机构做市（如 Wintermute）的核心差距是什么？散户做市是否应该和机构抢 spread？请列出 3 条散户做市的"生路"。
4. **（加密 maker 返佣）** Gate 80% 返点后 maker 实际费率是多少？散户做市 BTC（spread 0.02%）的盈亏平衡分析是什么？如果 spread = 0.1%（中币 SOL/AVAX）呢？
5. **（失败模式）** 列举 4 类加密做市的典型失败模式，并给出对应的防御措施。为什么 Hummingbot 用户的"一分钱没赚到"不是策略错而是参数错？
6. **（项目落地）** 100 USDT 项目的 Phase 1（起步期）应该上做市吗？为什么？Phase 2/3 应该如何演化？杠铃结构的具体资金分配是什么？
7. **（回测方法论）** 做市回测比费率套利回测多了哪 4 个核心难点？law-purdue 2015 指出"做市利润可被系统性高估"的根因是什么？做市回测应默认打几折？

---

## 资料来源（Tier 分级列表）

### Tier 1：原始论文（直接访问摘要/转述，未读全文）

- **Avellaneda, M. & Stoikov, S. (2008)**. *High-frequency trading in a limit order book.* Quantitative Finance 8(3): 217-224. —— 本文核心引用；ResearchGate PDF 可访问，包含完整公式推导 (Eq. 3.10-3.12) 与 Table 1 仿真对照
- **Guéant, O., Lehalle, C.-A. & Fernandez-Tapia, J. (2013)**. *Dealing with the Inventory Risk. A solution to the market making problem.* arXiv 1105.3115；后续 Math. Fin. Econ. 7(4). —— 多资产做市扩展
- **Glosten, L. R. & Milgrom, P. R. (1985)**. *Bid, Ask and Transaction Prices in a Specialist Market with Heterogeneously Informed Traders.* JFE 14(1): 71-100. —— 逆向选择与 spread 解释
- **Kyle, A. S. (1985)**. *Continuous Auctions and Insider Trading.* Econometrica 53(6): 1315-1336. —— Kyle λ（被引 15,896）
- **Cartea, Á., Jaimungal, S. & Penalva, J. (2015)**. *Algorithmic and High-Frequency Trading.* Cambridge University Press. —— 第 6-7 章 HFT 系统化
- **Han, F. & Fang, Y. (2020)**. *Bayesian Market Making*（思想二手转述）。—— Bayesian 做市
- **Law, C. W. (2015)**. *A Pure-Jump Market-Making Model for High-Frequency Trading.* Purdue PhD Dissertation. —— "做市利润可被系统性高估"的奠基性论证
- **Lu, X. (2019)**. *Market Making under a Weakly Consistent Limit Order Book Model.* arXiv 1903.07222. —— L3 修正对 AS 模型的实证改进

### Tier 2：综述与教材（二手转述/讲义；未直接读原书）

- **Cartea, Á., Jaimungal, S. & Penalva, J. (2015)**. *Algorithmic and High-Frequency Trading* 第 6-7 章 —— 单一/多资产 HFT（未读原书）
- **Hasbrouck, J. (2007)**. *Empirical Market Microstructure.* Oxford. —— 实证微观结构（NYU 教学讲义公开）
- **Gašperov, B. et al. (2021)**. *Reinforcement Learning Approaches to Optimal Market Making.* MDPI Mathematics 9(21): 2689. —— RL 做市综述（被引 49）
- **Lehalle, C.-A. & Laruelle, S. (2013)**. *Market Microstructure in Practice.* World Scientific. —— 工程导向
- **hftbacktest 文档**（Guéant-Lehalle-Fernandez-Tapia Market Making Model and Grid Trading）—— GLFT 工程实现

### Tier 3：二手科普与平台文档

- **Hummingbot Academy** "What is Market Making?" + pure_market_making / avellaneda_market_making / inventory_skew 官方文档 —— 工程化实现细节
- **crypticweb3** "Best Crypto Market Makers in 2026" —— Wintermute/Jump/GSR/Cumberland 机构排名
- **c-sharpcorner** "Best Crypto Market Maker in 2026: How to Choose..." —— 机构能力对比
- **Hyrotrader** "Crypto Market Makers Guide" —— Wintermute 22 亿美元日交易量、机构模式总结
- **LLMQuant** "Optimal High-Frequency Market Making" —— AS 模型 Substack 解读
- **quantbeckman** "Market Making: Avellaneda-Stoikov model [WITH CODE]" —— AS 模型的代码化解读
- **Navnoor Bawa / Medium** "Optiver's €3.5B Market-Making Engine: Avellaneda-Stoikov Inventory Optimization at Scale" —— 工业级 AS 实现
- **arXiv 2508.20225** "Optimal Quoting under Adverse Selection and Price Reading" —— adverse selection + skew sniffer 防御
- **Multicoin Capital (2020)** "March 12: The Day Crypto Market Structure Broke" —— 压力行情下做市商行为
- **Reddit r/algotrading** "Avoiding MM adverse selection in practice?" —— 社区实战经验
- **BlockApex / Medium** "Market Making Mechanics and Strategies" —— inventory + adverse selection 综述
- **Medium / poloxue** "How to Run a Crypto Market Making Bot with Hummingbot" —— 工程实践
- **mbrenndoerfer.com** 多篇关于订单簿与做市的文章 —— 含 AS 代码

### Tier 1/2/3 综合使用说明

- 本条目中所有 AS 模型公式（reservation price、spread 项、inventory penalty）来自 Avellaneda-Stoikov 2008 原文 PDF（ResearchGate 直接访问）；未读 GLFT/Cartea/Hummingbot 源码章节。
- Hummingbot 工程化实现细节来自 Tier 3 官方文档；Hummingbot 默认参数（γ/κ/refresh_time 等）来自 LLMQuant/quantbeckman 解读。
- 加密做市机构（Wintermute/Jump 等）排名与规模数据来自 crypticweb3 2026 综述。
- 项目应用映射（Phase 1/2/3 划分、杠铃结构）来自《自动量化项目架构初稿》L5 + 本条目作者综合判断。
- 数据点（Gate 80% 返点、费率结构、2025-10-10 闪崩指标）以 2026-08-30 行情为准；费率随账户等级变化，上线前以官网实测为准（架构初稿 L4 硬约束）。