# 期权与衍生品基础（Black-Scholes / IV / Greeks）

- **来源**：网络调研（firecrawl 多轮）+ 学术论文摘要 + Hull《Options, Futures, and Other Derivatives》思想二手转述 + Deribit Insights + Wikipedia 综述（**Tier 分级；未直接读 Hull 教材原书章节、BSM 1973 原文 PDF、CBOE VIX 白皮书全文**；通过 Wikipedia "Black-Scholes"、"Put-call parity"、"Greeks (finance)"、"VIX"、"Deribit" 直接访问摘要/公式 + Investopedia / Optiver / public.com Greeks 解释 + Gregory Gundersen 2024 直觉推导 + FalconX 2025 加密期权市场综述 + Amberdata "The Smile" 2026 IV skew 实证 + fast-vollib arXiv 论文 + Deribit 官方 statistics 页面交叉验证）
- **日期**：2026-08-30
- **主题**：量化交易 ｜ 标签：期权 · Black-Scholes · IV · Greeks · Deribit · 加密期权 · 对冲 · 保护性 Put · 凸性 · 反脆弱
- **一句话主旨**：期权不是赌博——是用凸性买入"反脆弱的保险"；100 USDT 项目的关键不是卖期权赚权利金，而是用 Long Put 锁定下行风险，把组合变成 Taleb 推荐的"反脆弱"形态（持币 + Long Put = 下行封顶 + 上行无顶）

> **配套条目**（直接引用，本文是外脑已有知识网的期权层补强）：
> - 《自动量化加密货币的成功与失败》§3.5 账户级对冲 + §9 费率套利实现细节——本文核心引用
> - 《自动量化项目架构初稿》L1 风控层（白旗熔断 / 日亏熔断）+ L5 策略层（费率套利 Phase 0/1/2/3）——本文应用面
> - 《系统思维与反脆弱》§6 凸性——Long Call/Long Put = 凸性组合的纯数学表达
> - 《市场微观结构与滑点建模》——IV 反映预期滑点；期权费包含逆向选择 + spread
> - 《风险量化 VaR/CVaR/ES》§7 L1 熔断线反向校准——保护性 Put = 给 ES 熔断线"买保险"
> - 《市场 Regime 检测与牛熊识别》——regime 切换 → 动态调整 put 比例的工程入口
> - 《做市策略与 Avellaneda-Stoikov》第 11 章 加密 maker 返佣算账——本文相关（期权做市的库存管理）
> - 《复利与非线性回报》§2 期权 = 凸性的最纯粹表达——本文思维层基础

---

## 可复用原则（决策时引用）

1. **期权的本质是凸性，不是赌博**：买方（Long Call/Long Put）= 风险有限 + 收益可能巨大 = **凸性**（f'' > 0）；卖方（Short Call/Short Put）= 收益有限 + 风险可能巨大 = **凹性**（f'' < 0）。承自《系统思维与反脆弱》§6 + 《复利与非线性回报》§9.3：金融业里几乎所有卖期权的机构都在结构性卖"保险"——表面"稳赚"、尾部归零（2008 AIG、Lehman 部分）。
2. **Black-Scholes 不是"价格公式"，是"对冲公式"**：BS 给出的不是"期权应该值多少"的绝对答案，而是"在已知 S/K/T/r/σ 下，用 delta + 现金复制期权需要多少初始资金"的最小对冲成本。它假设市场无套利（无风险套利不存在的市场里），所以"价格 = 复制成本"。
3. **IV 反映预期，不是历史**：历史波动率（HV）是过去 σ 的标准差，IV 是市场对未来 σ 的预期（risk-neutral 测度下的预期）。加密 IV 与 HV 长期背离：加密的隐含 IV 远高于后来的实现波动率——这是 IV 卖出策略（在加密）有"正期望"的根本原因。
4. **保护性 Put = 给币买保险**：持币 + Long Put = 下行封顶（行权价处）+ 上行无顶（保留全部 upside）。**这正是 Taleb 推荐的"凸性组合"的工程实现**——也是《自动量化项目架构初稿》L1 "白旗熔断 = -20 USDT" 的金融工程替代方案：与其"触发白旗再被动停机"，不如"事前 5 USDT/月买 Long Put"。
5. **期权的"Greeks"是风险预算的语言**：Delta = 方向风险、Gamma = Delta 变化率、Theta = 时间衰减、Vega = 波动率风险、Rho = 利率风险。**期权做市商管理组合 = 管理 Greeks**（delta 中性 + gamma scalping + theta 衰减）。对项目而言：哪怕不做期权，理解 Greeks 是"读懂任何 IV-based 工具"的必备基础。
6. **加密 IV 特征与传统的三大差异**：① 加密 IV 水平远高（BTC ATM IV 30-90% vs SPX ATM IV 12-25%）；② IV skew 极不对称（put wing 长期高于 call wing，因为加密 24/7 + 清算级联让 put 是真保险）；③ IV term structure 短端远高于长端（近月 IV 70% / 远月 IV 50%）——市场对未来 6 个月"回归常态"有隐含定价。
7. **Phase 1 不做期权，Phase 3 集成 Long Put**：100 USDT 项目的演化路径与《架构初稿》L5 一致——费率套利（确定性收益结构）+ 期权对冲（凸性保护）= 完整的反脆弱组合。Phase 1-2 用费率套利赚"风险溢价"；Phase 3 引入 Long Put 把组合从"凸性暴露（费率套利本身已含凸性）"升级为"凸性 + 反脆弱"。
8. **VIX 是股市"恐惧温度计"**：VIX > 30 = 市场恐慌（机会主义买入信号）；VIX < 15 = 市场自满（风险升高）。Deribit 有同款 "DVOL"（BTC DVOL + ETH DVOL），是加密市场情绪的实时指标。**加密 DVOL 长期 40-80%**，相当于股市"温和恐慌"常驻。
9. **Put-Call Parity 是无套利的硬约束**：C − P = S − K·e^(−rT)（无股息情形）。如果 C − P ≠ S − K·e^(−rT)，就有套利空间——这就是"合成"交易的数学基础（用 Call 复制 Put，用 Stock 复制合成期货等）。承自 Wikipedia "Put–call parity" + Investopedia "Put-Call Parity"。
10. **期权做市 = 用库存风险换权利金**：和现货/期货做市同一框架（《做市策略与 Avellaneda-Stoikov》AS 模型），区别是期权做市商要管理 Delta/Gamma/Vega/Theta 四个维度，而不是简单的库存；加密期权做市的优势是"Deribit 80% 份额 + 机构主导"——散户挂 maker 单反而容易在 regime 切换时被逆向选择。

---

## 核心逻辑链

1. **前提**：金融衍生品的"风险结构"由 4 个 Greek 决定（方向风险 Δ、时间衰减 Θ、波动率风险 ν、利率风险 ρ）。其中 Δ 与 ν 是主动暴露（你能选择多大），Γ 与 Θ 是被动暴露（你持有多头 Long 期权自动获得）。
2. **机制（BS 1973）**：在 5 大假设（GBM 标的、恒定 σ、恒定 r、无股息、无套利）下，期权价格服从 Black-Scholes PDE → 解出闭式公式 C = S·N(d₁) − K·e^(−rT)·N(d₂)。Put-Call Parity 给出 Put 价格 P = C − S + K·e^(−rT)。"价格"本质上是用股票 + 现金"复制期权损益"的最小初始资金。
3. **机制（IV 反推）**：BS 公式给出价格 C 是 S/K/T/r/σ 的函数；如果 C、K、T、r 在市场可观测，则 σ 可反推——这就是"隐含波动率 IV"。**IV 是市场对"未来波动率"的共识预期**，不是历史实现值。
4. **结果（多空头风险结构）**：买方（Long Call/Long Put）= 损失封顶（最多亏权利金）+ 收益可能巨大 = **凸性**（f'' > 0）；卖方 = 收益封顶（最多赚权利金）+ 损失可能巨大 = **凹性**（f'' < 0）。**所有"长期卖期权"的金融机构都是在卖"保险"——赚的是概率上对的小额权利金，赌的是尾部不发生**（2008 危机 AIG 卖出 CDS 的尾部归零）。
5. **结果（加密期权市场结构）**：Deribit 占加密期权 ~85% 市场份额、~80% 成交量是机构（FalconX 2025 / cfc-stmoritz.com 行业综述）；Deribit 2025 年总成交量 $1.875 万亿（24h 期权成交量常见 $1B+）；BTC ATM IV 长期 30-90%；BTC DVOL（Deribit 自家 VIX 风格指数）2025 年大部分时间在 40-60%。
6. **结果（项目应用）**：100 USDT 费率套利 + 期权对冲 = "风险溢价收割 + 尾部保险"。Phase 1 费率套利单独跑（净年化 8-15%）；Phase 3 引入 Long Put 把组合变成"凸性 + 反脆弱"——这是项目从"工程验证"到"长期可投资"的升级路径。
7. **决策口诀**：卖期权 = 卖"保险"（短期赚、长期尾部爆）；买期权 = 买"保险"（短期亏、长期反脆弱）；100 USDT 项目买 Put 是"小额保护下行的反脆弱成本"，不是"投机"。
8. **诚实声明**：本文**未直接读** Hull《Options, Futures, and Other Derivatives》（第 8 版/第 11 版）、Black & Scholes 1973 JPE 原文 PDF、Merton 1973、Wikipedia "Black-Scholes" 完整页面；具体公式与解释通过 Wikipedia 标准条目 + Investopedia 教程 + Gregory Gundersen 2024 推导 + Optiver/public.com Greeks 解释 + Deribit 官方数据交叉验证。

---

## 分章笔记

### 第一部分　期权基础（必含）

#### 第一章　Call / Put + 4 种基础策略 + 多空头风险结构

**1.1 期权的一句话定义**

> 期权（option）= 一种**合约**，赋予买方在到期日前（或到期日）按约定价格（行权价 K）买入（Call）或卖出（Put）约定数量标的资产的权利（不是义务）。
>
> **买方付费（权利金 premium）获得权利；卖方收权利金承担义务。**

承自 Wikipedia "Option (finance)" + Investopedia "Option"：期权是"right but not obligation"——这是与期货（"obligation"）的根本区别。

**1.2 期权的两大类**

| 类型 | 行权方向 | 多头期望 | 关键术语 |
| --- | --- | --- | --- |
| **Call（看涨期权）** | 在到期日前按 K 买入 | 标的上涨 | long call = 看涨；short call = 看跌 |
| **Put（看跌期权）** | 在到期日前按 K 卖出 | 标的下行 | long put = 看跌；short put = 看涨 |

**1.3 关键参数**

- **标的（S, Underlying）**：BTC、ETH、SPX、AAPL 等
- **行权价（K, Strike）**：合约约定的买卖价格
- **到期日（T, Expiry）**：欧式（European）只能在到期日行权；美式（American）到期日前任一交易日行权
- **当前价（S₀）**：签订合同时的市场价
- **到期价值（Intrinsic Value）**：max(S − K, 0) for Call，max(K − S, 0) for Put
- **时间价值（Time Value）**：权利金 − 内在价值；只对未到期的期权有意义
- **moneyness**：ITM（价内，intrinsic > 0）/ ATM（平价，S ≈ K）/ OTM（价外，intrinsic = 0）

**1.4 4 种基础策略的风险结构**

承自 Investopedia + 《系统思维与反脆弱》§6 凸性 + 《复利与非线性回报》§9.3：

| 策略 | 下行 | 上行 | 风险结构 | 二阶导数 f'' | 凸性判定 |
| --- | --- | --- | --- | --- | --- |
| **Long Call** | 最多亏权利金（封顶） | 理论上无顶 | 下行有底 + 上行无顶 | > 0 | **凸性** |
| **Long Put** | 最多亏权利金（封顶） | 上行收益有限（K − S₀） | 下行有底 + 上行有限 | > 0 | **凸性** |
| **Short Call** | 风险无顶（标的涨无上限） | 最多赚权利金 | 下行无底 + 上行封顶 | < 0 | **凹性** |
| **Short Put** | 风险较大（标的可跌到 0） | 最多赚权利金 | 下行有底（0）+ 上行封顶 | < 0 | **凹性** |

**1.5 多头 vs 空头的"做生意"视角**

承自《复利与非线性回报》§9.3 + 风险管理常识：

- **期权多头（buyer）**：付一笔固定的"保险费"，获得"如果发生我不希望的事，我能拿回固定金额"的保障——这就是**买保险**。风险有限（最多亏保费），收益可能巨大（标的反向运动时）。
- **期权空头（seller）**：收固定的"保险费"，承担"如果发生对手方不希望的事，我要付钱"的义务——这就是**卖保险**。收益有限（最多赚保费），风险可能巨大（极端行情下远超保费）。
- **金融机构长期卖期权**（结构性"卖保险"）：Investment banks、market makers 大量 Short Vol 策略——表面"年化 8-15% 稳赚"，但 2008 危机里多家爆仓（AIG 卖出 $500B+ CDS 累计赔付超过保费储备、Lehman 部分 P&L 来自 mortgage-backed CDB 卖方头寸、Archegos 2021 单日 -35% 强平）。
- **"卖期权像开赌场"是 Taleb 的核心论点**：金融业里几乎所有"赚权利金"的策略都是隐性的"做空波动率"——他们赌"未来波动率比市场预期的低"，而市场预期本身包含了对极端事件的"恐惧溢价"。

**1.6 期权的两种行权方式**

- **欧式（European）**：只能在到期日行权。绝大多数场内期权是欧式。
- **美式（American）**：到期日前任一交易日可提前行权。美股股票期权多为美式（但绝大多数情况下不会提前行权）。
- **加密期权**：Deribit 上 BTC/ETH 期权是欧式现金结算（不交割实物）；CME 比特币期货期权是美式。

**1.7 期权策略的"凸性暴露"判据**

承自《复利与非线性回报》§9.3：

> H = [f(a−Δ) + f(a+Δ)] / 2 − f(a)
> H < 0 → 凹性（脆弱）| H = 0 → 线性（稳健）| H > 0 → 凸性（反脆弱）

- **Long Call/Long Put**：H > 0 → **凸性** = 从波动中获益
- **Short Call/Short Put**：H < 0 → **凹性** = 被波动伤害
- **持币 + Long Put（Protective Put）**：H > 0 → **凸性组合** = Taleb 推荐的"反脆弱平衡"

---

### 第二部分　Black-Scholes 公式（必含）

#### 第二章　BS 5 大假设 + 公式骨架 + Put-Call Parity

**2.1 Black-Scholes-Merton 1973 的历史**

承自 Wikipedia "Black–Scholes model" + Gregory Gundersen 2024 直觉推导：

- **Black & Scholes 1973**：发表 "The Pricing of Options and Corporate Liabilities" 于 *Journal of Political Economy* 81(3): 637-654。这是现代期权定价理论的奠基性论文——用了 3 年时间推导。
- **Merton 1973**：同期发表论文，扩展了数学理解，**首次提出 "Black-Scholes options pricing model" 这个术语**。Merton 因此与 Scholes 共同获 1997 年诺贝尔经济学奖（Black 已于 1995 年去世）。
- **历史意义**：BS 公式发布前，期权市场规模小且流动性差——70 年代 CBOE 成立后市场扩张，BS 公式提供了"公平定价"标准，让套利者能识别错误定价，市场有效性大幅提升。

**2.2 BS 的 5 大假设**

承自 Investopedia "Black-Scholes Model" + Wikipedia "Black–Scholes model"：

1. **标的服从几何布朗运动（GBM）**：dS = μS dt + σS dW（μ 是 drift，σ 是 volatility，W 是 Wiener process）。这条假设意味着 S 的对数收益率服从正态分布，标的可能无限上涨（不设上限），理论上可能跌到 0 但不会为负。
2. **波动率 σ 恒定**：σ 不随时间变化。这一条**在加密市场完全不成立**（加密 vol 长期聚集）。
3. **无风险利率 r 恒定**：r 是已知的常数。实务上用对应到期的国债收益率。
4. **不支付股息或股息率已知**：原版 BS 假设不支付股息；Black-Scholes-Merton 扩展到股息率 q。
5. **无套利市场**：市场是有效的——任何套利空间会被瞬间消除。这一条让"复制组合 = 期权价格"成立。

**2.3 BS 公式骨架（欧式 Call）**

承自 Wikipedia "Black–Scholes model" + Gregory Gundersen 2024 推导：

```
C = S₀ · N(d₁) − K · e^(−rT) · N(d₂)

d₁ = [ln(S₀ / K) + (r + σ² / 2) · T] / (σ · √T)
d₂ = d₁ − σ · √T
```

其中 N(·) 是标准正态 CDF。

**2.4 公式的几何直觉（最重要的"为什么"）**

承自 Gregory Gundersen 2024 + Wikipedia "Black–Scholes model" §"Risk neutral world"：

- **第一项 S · N(d₁)**：股票在风险中性测度下的"条件期望"——当 S_T > K 时，行权收益正比于 S_T；N(d₁) 是"到期 S_T > K 的概率"（在 risk-neutral 测度下）。
- **第二项 K · e^(−rT) · N(d₂)**：行权价 K 的现值（贴现到 t=0）× 到期被行权的概率（同样是 risk-neutral 测度）。N(d₂) 是 P(S_T > K) 在风险中性测度下的概率。
- **Feynman-Kac 公式**：BS PDE 的解在数学上等价于"在 risk-neutral 测度下对折现 payoff 的期望"。**BS 价格 = 期望收益，不是"市场认为的真实价格"**——因为我们在 Q-measure（风险中性）而非 P-measure（真实概率）下求期望。
- **物理意义**："Q-measure 把所有投资者都当成 risk-neutral"——所以 μ 被 r 替代，σ 不变。在 Q-measure 下，"持有股票"与"持有现金 + 借股票"的回报相同——这就是"无套利"的精确表述。

**2.5 Put-Call Parity**

承自 Wikipedia "Put–call parity" + Investopedia + Corporate Finance Institute：

```
欧式期权无股息情形：
C − P = S − K · e^(−rT)
   ≡ S − D · K   （其中 D = e^(−rT) 是贴现因子）

等价形式（更常用）：
C + K · e^(−rT) = P + S
```

**物理意义**：
- 左边 = "持 Call + 现金"组合：到期时若 S_T > K，行权得 S_T；若 S_T ≤ K，cash = K · e^(−rT) 增值到 K → 始终得 max(S_T, K)
- 右边 = "持 Put + 股票"组合：到期时若 S_T < K，行权得 K；若 S_T ≥ K，持股票值 S_T → 同样始终得 max(S_T, K)
- 两者终值相同 → 现值相同 → 等式成立

**2.6 BS 公式的 6 个输入变量**

承自 Investopedia "Black-Scholes Model"：

| 变量 | 符号 | 可观测性 | 来源 |
| --- | --- | --- | --- |
| 标的现价 | S₀ | 直接观测 | 交易所报价 |
| 行权价 | K | 直接观测 | 合约条款 |
| 到期时间 | T | 直接观测 | 合约条款 |
| 无风险利率 | r | 直接观测 | 国债收益率 |
| 波动率 | σ | 不可直接观测 | 从期权价格反推（=IV）或用历史波动率近似 |
| 股息率 | q | 直接观测 | 公司分红或协议 |

**唯一一个不可直接观测的输入就是 σ**——这正是 IV 的用武之地。

**2.7 BS 的"对冲视角"（最实用）**

承自 Wikipedia "Black–Scholes model" §"Interpretation" + Greg Gundersen 2024：

> BS 价格不是"市场认为的合理价格"——它是"用 Δ 单位股票 + 现金复制期权需要的初始资金"。

具体来说：
- **Delta 复制**：每持 1 份期权多头，卖 Δ 份股票 → 组合 delta 中性
- **复制组合 = −N(d₂) 份股票（空）+ K·e^(−rT)·N(d₂) 现金（借出）？** 不对，delta 复制是动态的，需要不断 rebalance（"delta hedging"）
- **BS 价格 = 复制组合需要的初始资金**：因为假设无套利，初始资金 = 期权市场价格

这意味着：**当市场实际期权价格偏离 BS 价格时，存在套利空间**——这就是 hedge funds 用的"vol arb"策略。

---

#### 第三章　BS 的局限性（加密市场特别明显）

**3.1 恒定 σ 假设在加密完全失效**

承自 Wikipedia "Black–Scholes model" + 《风险量化 VaR/CVaR/ES》§4：

- BS 假设 σ 在期权存续期内不变——这在传统股票市场勉强成立（σ 相对稳定）
- **加密市场 σ 跳变频繁**：BTC 日波动率从 30% 跳到 80% 经常发生（regime 切换）；同一份 BTC 期权在 1 周内"重新定价"几次是常态
- **30-day 历史波动率 → BS σ** 是常见近似，但永远偏离 IV

**3.2 无跳跃假设**

- BS 假设标的服从 GBM——价格连续，没有跳空
- **加密市场跳空常见**：24/7 + 杠杆级联 → 单分钟 -10% 不是"罕见事件"
- 跳跃导致极端行情期权定价严重偏低（BS 给出"看起来便宜"但实际很贵的 put）

**3.3 美式期权处理不直接**

- BS 只解欧式——美式提前行权的可能性需要 binominal tree 或 finite difference
- **加密期权大多是欧式**（Deribit 是欧式现金结算）→ 这条对加密市场影响较小

**3.4 对加密的修正方向**

- **Heston 1993 模型**：用随机波动率（Stochastic Volatility），让 σ 服从均值回归过程。比 BS 更适合加密。
- **Merton 1976 跳跃扩散模型**：加 Poisson 跳跃项。处理加密的"清算级联跳空"。
- **Local Volatility（Dupire 1994）**：让 σ 随 S 和 T 变化——这就是后面要讲的 IV Surface。
- **SABR / Heston + 跳跃 / rough volatility**：更高级的加密期权定价模型。

承自《自动量化项目架构初稿》§7 + 本文：项目 Phase 1/2 不做期权（费率套利更稳）；Phase 3 引入 Long Put 时，**BS 公式对单点定价够用**（不需要 Heston 复杂度），但 IV 反推必须用市场实际成交价 → 避免"BS 给出理论价 ≠ 市场实际价"。

---

### 第三部分　隐含波动率 IV 与波动率曲面（必含）

#### 第四章　IV 的定义 + 历史 σ vs IV + IV Surface

**4.1 隐含波动率 IV 的一句话定义**

> IV = 把当前期权市场价格代入 BS 公式，反推出来的 σ（"市场共识的未来波动率"）

承自 Wikipedia "Implied volatility" + Vollib `implied_volatility` API：

```python
from vollib.black_scholes import black_scholes
from vollib.black_scholes.implied_volatility import implied_volatility

# 已知：期权市场价格 + 其他输入 → 反推 IV
flag, S, K, t, r = 'c', 65000, 70000, 0.05, 0.04
market_price = 2500
iv = implied_volatility(market_price, S, K, t, r, flag)
# iv ≈ 0.62 = 62%
```

**4.2 历史波动率 vs IV**

| 指标 | 含义 | 时间方向 | 用途 |
| --- | --- | --- | --- |
| **历史波动率（HV / realized vol）** | 过去 N 天收益的标准差 | backward-looking | 校准 BS σ；HV < IV → 卖出期权；HV > IV → 买入期权 |
| **隐含波动率（IV）** | 未来 N 天收益的市场预期（risk-neutral 测度） | forward-looking | 期权定价；情绪指标；"市场认为未来波动有多大" |
| **预期波动率（forecast vol）** | GARCH 等模型对未来 σ 的预测 | forward-looking | 校准 BS σ；与 IV 差距是"vol arb"的 source |

**4.3 波动率曲面（IV Surface）**

承自 Wikipedia "Volatility smile" + Wikipedia "Implied volatility surface" + Amberdata "The Smile" 2026：

- **定义**：IV 是 (行权价 K, 到期日 T) 的二元函数 → 形成三维曲面
- **曲面 3 个轴**：
  - X 轴：行权价（moneyness = K/S）
  - Y 轴：到期时间（time to maturity）
  - Z 轴：IV 值（颜色或高度）
- **典型形状**：右侧高（OTM Put IV > ATM IV > OTM Call IV）——因为 put 是保险，市场愿为下跌保护付高价

**4.4 IV Smile（波动率微笑）**

承自 Wikipedia "Volatility smile" + ResearchGate 2021 Bitcoin IV 实证：

- **定义**：固定到期日，IV 随行权价 K 变化的曲线；ATM 时 IV 最低，OTM Put 和 OTM Call 时 IV 更高 → 形成"微笑"
- **出现时间**：1987 黑色星期一后首次观察到（之前 IV 几乎是常数）
- **加密版本**：BTC/ETH 期权 IV smile 实证（ResearchGate 2021）显示明显的 forward volatility skew——put wing 比 call wing 高很多
- **本质**：OTM Put 长期被市场"过度买入"——是"保险溢价"

**4.5 IV Skew（波动率偏斜）**

承自 Amberdata "The Smile" 2026-08 + ResearchGate 2021：

- **定义**：IV smile 不对称——put wing 高于 call wing 的程度
- **传统市场**：1987 后 35 年 equity index skew 长期为负（put IV > call IV）——市场愿为"crash protection"付高价
- **加密特殊性**：Amberdata 2026-08 报告 BTC 25-delta RR = -3.46 vol points（在 90 天分布的 98 百分位）——put bid 强烈；但近 2 年加密市场出现"positive skew 区间"（call wing 高于 put wing）——这是结构性的差异，反映加密参与者结构不同（散户 + 长期持有者 + 公司财务买入 call）
- **2026-08 解读**：BTC 25-delta RR 在 7D/30D/60D/90D 都为负（-4.15 / -3.46 / -3.71 / -3.79）→ put wing 仍主导，但"近期正 skew 的回归" 是结构性现象（参与者结构差异 → 不只是叙事）

---

#### 第五章　VIX + 加密市场的 IV

**5.1 VIX 是什么**

承自 Wikipedia "VIX" + CBOE VIX Methodology PDF：

> VIX 是 CBOE 计算的**标普 500 指数未来 30 天预期年化波动率**——基于一系列 SPX 看跌/看涨期权价格反推。

**CBOE VIX 计算公式（简化）**：

```
σ² = (2/T) · Σᵢ (ΔKᵢ / Kᵢ²) · e^(RT) · Q(Kᵢ) − (1/T) · (F/K₀ − 1)²

VIX = σ × 100
```

其中：
- T = 到期时间（年化）
- ΔKᵢ = 邻近 strike 间距
- Q(Kᵢ) = 该 strike 的期权 mid 报价
- K₀ = 第一个 ≤ forward price F 的 strike
- F = 由期权价格反推的 forward price

**5.2 VIX 的含义**

- **VIX 是 30 天预期年化标准差**：例如 VIX = 20 意味着市场预期标普 500 未来 30 天的年化波动率约 20%
- **"恐慌温度计"**：VIX > 30 = 市场恐慌；VIX < 15 = 市场自满
- **历史峰值**（Wikipedia "VIX"）：
  - 2008-10-24：59.89（雷曼危机后）
  - 2020-03-12：75.47（COVID 流动性危机，**超过 Black Monday 1987**）
  - 2020-03-16：82.69（更高）

**5.3 VVIX（Vol of Vol）**

- VVIX = 测量 VIX 自身的预期波动率——"恐惧的加速度"
- **类比**：VIX 是速度，VVIX 是加速度；正常速度不可怕，加速才可怕
- 由 CBOE 2012 年推出

**5.4 Deribit DVOL（加密版 VIX）**

承自 Deribit 官方 statistics 页面（实时数据）：

- DVOL = Deribit 计算的 BTC/ETH 30 天预期年化 IV——同样基于期权价格反推
- **当前水平**（2026-08-30 数据快照）：
  - BTC DVOL ≈ 30-60%（近期大部分时间 40-60%）
  - ETH DVOL ≈ 50-80%（ETH 比 BTC 略高）
- **与 VIX 对比**：加密 DVOL 是 SPX VIX 的 2-4 倍——反映加密市场结构性高波动

**5.5 加密 IV 的三大结构性特征**

承自 Deribit statistics + FalconX 2025 + Amberdata 2026 + ResearchGate 2021 BTC IV 实证：

| 特征 | 加密 IV | 传统 IV（SPX） | 原因 |
| --- | --- | --- | --- |
| **水平** | BTC 30-90% / ETH 50-100% | 12-25% | 加密资产本身波动大 + 24/7 + 高杠杆 |
| **skew 不对称** | 长期 put wing > call wing（put 溢价） | 同样 put wing > call wing，但更稳定 | 加密清算级联是真实风险 → put 是真保险 |
| **term structure** | 短端（近月）远高于长端（远月） | 短端有时高有时低（contango/backwardation） | 加密短期事件密集 + 24% 月波动比长期更可预测 |

**5.6 BTC ATM IV 历史范围（Deribit 数据，2025）**

承自 FalconX 2025-10 报告：

> "If there's been one constant in 2025 options chatter, it is that BTC implied volatility has stayed remarkably low."

- **2025 大部分时间 BTC 30d ATM IV 维持在 30-50%**——对加密来说"异常低"
- BTC 与 ETH IV 在 2024 年中开始分化（分叉）——此前两者高度相关
- 含义：BTC 期权市场可能进入"压缩 IV"阶段，IV 卖出策略（covered call、cash-secured put）在 BTC 上的收益预期下降

---

### 第四部分　Greeks（期权风险敏感度，必含）

#### 第六章　Delta / Gamma / Theta / Vega / Rho

**6.1 Greeks 的一句话定义**

> Greeks 是期权价格对各输入变量的一阶（或二阶）偏导数——衡量期权价格对每个变量的敏感度。

承自 Wikipedia "Greeks (finance)" + Investopedia "Option Greeks" + public.com "Options Greek Trading"：

| Greek | 符号 | 数学定义 | 含义 |
| --- | --- | --- | --- |
| **Delta** | Δ | ∂V / ∂S | 期权价格对标的价格的敏感度 |
| **Gamma** | Γ | ∂Δ / ∂S = ∂²V / ∂S² | Delta 对标的价格的敏感度（二阶导数） |
| **Theta** | Θ | ∂V / ∂t | 期权价格对时间的敏感度（时间衰减） |
| **Vega** | ν / V | ∂V / ∂σ | 期权价格对波动率的敏感度 |
| **Rho** | ρ | ∂V / ∂r | 期权价格对利率的敏感度 |

**6.2 Delta（Δ）—— 方向敏感度**

承自 Wikipedia "Greeks (finance)" + Investopedia：

- **Call Delta**：0（OTM Call）到 +1（深度 ITM Call）；ATM Call 通常 ≈ +0.5
- **Put Delta**：0（深度 OTM Put）到 -1（深度 ITM Put，绝对值）；ATM Put 通常 ≈ -0.5
- **几何直觉**：Delta = 期权到期被行权的概率（在 risk-neutral 测度下）；也是"对冲需要的股票份数"
- **Delta hedging**：每持 1 份多头期权，做 Δ 份空头股票 → 组合 delta 中性

**6.3 Gamma（Γ）—— Delta 的变化率**

承自 Wikipedia "Greeks (finance)"：

- **公式**：Γ = ∂Δ / ∂S = ∂²V / ∂S² > 0（多头期权 gamma 为正）
- **几何直觉**：Delta 变化的速度；ATM 期权 Gamma 最高
- **临近到期**：ATM 期权 Gamma → 无穷大（"pin risk"——Delta 在到期日瞬间从 0 跳到 1 或反之）
- **多头 Gamma 含义**：标的涨，Delta 增大（自动增加对冲的股票空头），标的大涨 → 进一步加空 → 反复"低买高卖"——这是"gamma scalping"的基础

**6.4 Theta（Θ）—— 时间衰减**

承自 Investopedia "Option Greeks" Theta 章节 + public.com：

- **时间价值随时间消耗**——期权价值 = 内在价值 + 时间价值；到期日只剩内在价值
- **Theta 通常为负**（持多头每天亏时间价值）；**持空头 theta 为正**——期权卖方的"朋友"
- **Theta 加速规律**：临近到期时 ATM 期权的 Theta 绝对值快速增大——最后 30 天 theta 消耗占整个存续期的 50%+
- **加密特殊性**：24/7 让 theta 持续消耗（无市场关闭），但 theta 衰减率不变——只是"日历时间"与"交易时间"一致

**6.5 Vega（ν）—— 波动率敏感度**

承自 Investopedia "Vega" + public.com "Vega: Sensitivity to Volatility"：

- **公式**：ν = ∂V / ∂σ；通常 ν > 0（IV ↑ → 期权价格 ↑）
- **量级**：ATM Vega 典型值 ≈ 0.05-0.15（每 1% IV 变化对应期权价格变化 5-15 美分/股）
- **规律**：
  - ATM 期权 vega 最大（vega 集中在 ATM）
  - 长期期权 vega 大于短期期权（30 天 vega > 7 天 vega）
  - 临近到期 ATM vega → 0（gamma 趋向无穷大但 vega 趋向 0）
- **波动率交易的核心**：做多/做空 Vega = "long vol / short vol" 策略的数学表达

**6.6 Rho（ρ）—— 利率敏感度**

承自 Wikipedia "Greeks (finance)" + public.com "Rho: The Interest Rate Sensitivity"：

- **公式**：ρ = ∂V / ∂r（每 1% 利率变化对应期权价格变化）
- **量级**：Rho 通常是 Greeks 中最小的——因为利率变化对期权定价的影响远小于标的价格或波动率
- **长期期权 rho 较高**：30 天 ATM Call rho ≈ 0.05；3 个月 ATM Call rho ≈ 0.15
- **2022 加息周期**：美联储快速加息使长期利率敏感性成为重要因素——传统 rho "长期被忽视" 在那个时期一度热门

**6.7 Greeks 的"角色分工"**

- **方向风险**：Δ（你是否能承受方向暴露？）
- **波动率风险**：ν（你是否能承受 IV 跳变？）
- **时间风险**：Θ（你是否能承受时间衰减？）
- **曲率风险**：Γ（你是否能承受 delta 跳变？）
- **宏观风险**：ρ（你是否能承受利率跳变？）

| Greek | 多头 Long Call | 多头 Long Put | 空头 Short Call | 空头 Short Put |
| --- | --- | --- | --- | --- |
| **Delta (Δ)** | +（0 到 +1） | −（−1 到 0） | −（−1 到 0） | +（0 到 +1） |
| **Gamma (Γ)** | + | + | − | − |
| **Theta (Θ)** | −（亏时间） | −（亏时间） | +（赚时间） | +（赚时间） |
| **Vega (ν)** | +（IV 涨赚） | + | − | − |
| **Rho (ρ)** | +（Call 受利率影响正） | −（Put 受利率影响负） | − | + |

---

#### 第七章　Greeks 的组合管理（Delta hedging / Gamma scalping）

**7.1 Delta Hedging（动态 delta 复制）**

承自 Wikipedia "Delta neutral" + 《做市策略与 Avellaneda-Stoikov》§1.6 复制组合：

- **基本公式**：每持 1 份期权多头，做 Δ 份股票空头 → 组合 delta = 0
- **动态 rebalance**：标的 S 变化 → Δ 变化 → 调整空头股数 → 保持 delta 中性
- **频率**：1 次/秒（HFT）、1 次/分钟（活跃做市）、1 次/小时（日级对冲）、1 次/日（long-term 对冲）
- **对冲成本**：每次 rebalance 都有 spread + 滑点 + 手续费 → 频率越高，成本越高

**7.2 Gamma Scalping（"靠波动赚钱"）**

承自 Investopedia "Gamma scalping" + Wikipedia "Greeks (finance)"：

- **前提**：持有多头期权（gamma > 0）+ delta 中性组合
- **机制**：标的价格波动时 → delta 不再中性 → 平掉 delta 暴露 → 价格反弹 → 再平衡 → **低买高卖**
- **本质**：用多头期权 gamma 暴露，从标的波动中赚"自动低买高卖"的钱
- **关键**：需要**足够大的波动**才能覆盖 rebalance 成本
- **加密应用**：Deribit 期权做市商常用此策略——持 ATM 跨式多头（long straddle）+ 动态 delta 对冲

**7.3 Delta-Gamma-Vega 中性（做市商的目标态）**

- 理想做市商：组合 Δ = Γ = ν = 0 → 完全对冲所有风险，只赚 Θ（时间价值）
- **加密做市商现实**：因为加密 IV 高、波动大，组合经常偏离中性——需要持续调整

**7.4 Theta 与 Vega 的"平衡"**

承自《做市策略与 Avellaneda-Stoikov》§1.4：

- 卖出期权 = short theta + short vega → 赚时间衰减，但 IV 上涨时亏
- 买入期权 = long theta + long vega → 付时间衰减，但 IV 下跌时赚（long vega 不利）、方向有利时赚
- **加密做市商**：多数是 net long vega（因为加密 IV 长期偏高，预期 IV 会回归），但靠 maker rebate + spread 覆盖 theta 成本

---

### 第五部分　期权策略详解（必含）

#### 第八章　Covered Call / Protective Put / Spreads / Straddle / Iron Condor

**8.1 Covered Call（备兑看涨）**

承自 Investopedia "Covered Call" + CMC Markets：

- **结构**：持币（多头现货）+ Short Call（同一 strike）
- **收益结构**：
  - 上行：封顶在 K + 权利金（标的涨过 K 也只能按 K 卖）
  - 下行：仍持有币的全部下行风险（只是多了权利金作为缓冲）
- **适用**：长期看好 + 短期波动小（"愿意以 K 卖出，但先收一笔权利金"）
- **风险**：放弃上行 upside

**8.2 Protective Put（保护性看跌）**

承自 CMC Markets "The Protective Put" + Investopedia：

> "If prices rise, it expires worthless and you keep the gains minus the premium. If prices fall sharply, the put increases in value, offsetting losses below the strike. The maximum loss is floored at a known level."

- **结构**：持币 + Long Put
- **收益结构**：
  - 上行：保留全部 upside（put 过期作废，最多亏权利金）
  - 下行：封顶在 K − 权利金（最大损失已知）
- **本质**：**给币买保险**——就像给房子买火灾险
- **适用**：长期看好但怕短期崩

**8.3 Bull Call Spread（牛市价差）**

- **结构**：买 ATM Call + 卖 OTM Call（同到期日）
- **收益**：低成本 + 上行封顶（最大收益 = K2 − K1 − 权利金净支出）
- **适用**：温和看涨

**8.4 Bear Put Spread（熊市价差）**

- **结构**：买 ATM Put + 卖 OTM Put
- **收益**：下行收益封顶 + 权利金净收入小于纯 Long Put
- **适用**：温和看跌

**8.5 Long Straddle / Strangle（跨式 / 宽跨式）**

- **Long Straddle**：同时买 ATM Call + ATM Put（同 strike K = S₀）
- **Long Strangle**：同时买 OTM Call（K > S₀）+ OTM Put（K < S₀）——权利金比 straddle 便宜，但需要更大波动才能盈亏平衡
- **收益结构**：押注**大波动**——不论方向，只要 |S_T − S₀| > 权利金净支出就赚
- **加密应用**：常用，因为加密波动大 → 押注"波动会继续"的胜率比传统市场高
- **风险**：如果市场平静（vol 压缩、regime 切换到 sideways），时间价值快速衰减 → 亏损

**8.6 Iron Condor（铁鹰）**

承自 Investopedia "Iron Condor" + Optionalpha：

> "A more sophisticated options strategy, the iron condor is a risk-defined way to profit from low volatility by selling an out-of-the-money (OTM) put spread and an OTM call spread, collecting a net credit upfront."

- **结构**：
  - 卖 OTM Put（K1，较低）
  - 买 OTM Put（K2，K1 < K2）
  - 卖 OTM Call（K3，较高）
  - 买 OTM Call（K4，K3 < K4）
  - 形成"两个 spread 的组合"
- **收益**：在 K2 到 K3 之间最大收益（净收入的权利金）；最大亏损 = max(K2 − K1, K4 − K3) − 净权利金
- **适用**：低波动 + 横盘市场
- **Deribit 真实交易数据**（2025-08）：Iron condor 占 Deribit block trades 仅 0.1%（call spread 5.6%、call calendar spread 94%）——加密市场做 iron condor 的少，因为"低波动"假设不成立

**8.7 策略选择的"决策树"**

```
你的市场观点：
├── 强烈看涨 → Long Call
├── 温和看涨 + 愿意封顶 → Bull Call Spread
├── 强烈看跌 → Long Put
├── 温和看跌 → Bear Put Spread
├── 押注大波动（方向不明）→ Long Straddle / Strangle
├── 押注低波动 → Iron Condor / Short Straddle（高风险）
├── 长期持币 + 短期锁定下行 → Protective Put
└── 长期持币 + 短期愿意卖出 → Covered Call
```

---

### 第六部分　加密期权市场（必含）

#### 第九章　Deribit + 其他交易所 + 加密 vs 传统

**9.1 Deribit 主导加密期权市场**

承自 Deribit 官方页面 + FalconX 2025-10 报告 + cfc-stmoritz.com 行业综述：

- **市场份额**：Deribit 占加密期权 ~85% open interest（cf-stmoritz 报道）；24h 期权成交量常见 $1B+
- **2025 年总成交量**：$1,875B（$1.875 万亿）
- **机构主导**：~80% 成交量来自机构客户（market makers、hedge funds、quant firms）
- **BTC 期权 open interest**：~$30B 名义价值（2026-08 数据快照）
- **24h 数据（snapshot）**：
  - 24h Put Volume: 3,018 BTC 名义 / 24h Call Volume: 4,413 BTC 名义 → Put/Call ratio 0.68（看涨略多）
  - Open interest: Call 253,073 BTC / Put 142,270 BTC → Call/Put OI ratio ≈ 1.78

**9.2 IBIT 期权（BlackRock 比特币 ETF）崛起**

承自 FalconX 2025-10 报告：

> "IBIT options briefly eclipsed Deribit's just after launch... IBIT has been changing hands at a solid $2–3 billion a day, within shooting range of Deribit's $3–4 billion daily average."

- **IBIT 期权** = BlackRock 比特币现货 ETF 的期权——2024-11 推出后快速崛起
- **特点**：美式行权、受 SEC 监管、与现货 ETF 联动；**对比 Deribit**：put/call 比更低（0.3 vs 0.5-0.6）→ IBIT 期权更偏 call 主导（美国投资者更看涨）
- **意义**：传统金融市场（TradFi）正在用 IBIT 期权重新定义加密期权定价——未来 IBIT 可能蚕食 Deribit 份额

**9.3 其他加密期权交易所**

承自 cfc-stmoritz.com + 行业调研：

| 交易所 | 特点 | 目标用户 |
| --- | --- | --- |
| **Deribit** | 85% 份额，机构主导，欧式现金结算 | 机构 + 专业散户 |
| **OKX** | 综合衍生品平台，期权 + 永续 + 现货 | 散户 |
| **Bybit** | 期权 + 永续 + 现货 | 散户 |
| **Bit.com** | 老牌加密期权所 | 机构 |
| **Binance** | 期权已上线但份额较小 | 散户 |
| **dYdX / GMX / Hegic** | DEX 期权（永续 + 看涨） | DeFi 原生 |

**9.4 加密期权 vs 传统期权的差异**

承自《市场微观结构与滑点建模》§6 + Deribit 官方 + FalconX 2025：

| 维度 | 传统期权（SPX） | 加密期权（BTC/ETH Deribit） |
| --- | --- | --- |
| **交易时间** | 9:30-16:00 ET（盘前盘后受限） | **24/7 全天候** |
| **结算** | 实物交割（部分）或现金 | 现金结算（Deribit） |
| **行权方式** | 美式（多数股票）/ 欧式（指数） | **欧式**（Deribit）/ 美式（CME） |
| **波动率水平** | 12-25% 年化 | 30-90% 年化 |
| **IV skew** | 长期负（put > call） | **短期正/负波动**——参与结构变化导致 |
| **流动性** | 深（top 100 strike 价差 0.01%） | **浅**（top strike 价差 0.5-2%） |
| **清算** | 集中清算所（OCC） | 交易所内清算（无 CCP） |
| **监管** | SEC / CFTC | **相对宽松**（CFTC 部分监管） |
| **合约面值** | 100 股 / $100 | 0.01-1 BTC / 0.1-10 ETH（合约可调） |

**9.5 加密期权常用工具**

承自行业调研 + Wikipedia "Deribit" 引用：

| 工具 | URL | 用途 |
| --- | --- | --- |
| **Deribit Insights** | insights.deribit.com | Deribit 官方研究博客 |
| **Greeks.live** | greeks.live | 中文加密期权社区 + 数据 |
| **Laevitas** | laevitas.ch | 加密期权/期货数据 + 仪表板 |
| **Amberdata** | amberdata.io | 加密市场结构 + 期权分析（承自本文 §5.6） |
| **CoinGlass** | coinglass.com | 期权 + 永续数据（多所聚合） |
| **Genisis Volatility** | genesisvolatility.com | 加密期权 IV 历史数据库 |
| **The Block** | theblock.co | 加密市场综合数据 |

**9.6 Deribit DVOL（加密 VIX）的实时读数**

承自 Deribit statistics 页面（2026-08-30 快照）：

- **ATM Implied Volatility** 实时图：
  - BTC DVOL：最近 30 天大部分时间 30-50%（2026-08-30 快照）
  - ETH DVOL：50-70%
  - 与 2024-2025 年同期的 60-90% 相比，已显著回落
- **DVOL 含义**：30 天预期年化波动率 = 市场认为"未来 30 天 BTC 日波动率 ≈ DVOL / √12 ≈ 9-14%"
- **DVOL vs realized vol**：2025 年 DVOL 大部分时间高于 realized vol 5-10 个 vol points——这给 vol 卖出策略提供正期望空间

---

### 第七部分　项目应用映射（必含）

#### 第十章　100 USDT 项目的期权定位（Phase 1 不做 / Phase 2 实验 / Phase 3 集成）

**10.1 Phase 1（起步，唯一先做费率套利）**

承自《自动量化项目架构初稿》L5 阶段 1 + 本文原则 7：

**不做期权的理由**：
1. **资金门槛**：100 USDT 本金无法买 BTC 期权（最便宜的 OTM Put 一张 $500-2000 名义 → 1-5% 名义）
2. **学习成本高**：BS 公式、IV 反推、Greeks 管理——投入产出比不如费率套利
3. **策略定位不匹配**：费率套利是"确定性收益"；Long Put 是"概率保险"——两者期望差异大
4. **风控更简单**：费率套利有 delta 中性 + funding 保护；期权引入 gamma/vega 多维度风险

**10.2 Phase 2（中期，Phase 1 稳定 3 个月后）**

**实验性期权对冲**：
- **资金分配**：3-5 USDT 实验性 Long Put（占总资金 3-5%）
- **目的**：验证期权买/卖流程、IV 反推精度、Greeks 计算准确性、对冲效果
- **建议起点**：买 30 天 OTM Put（行权价低于现价 10-15%）——成本约 1-3% 名义本金
- **风险**：Phase 2 期权头寸最大亏损 = 权利金（即 3-5 USDT 全部归零）——已知最大损失

**10.3 Phase 3（成熟期，Phase 2 稳定 6 个月后）**

**集成对冲层**：
- **持币 + Long Put 的反脆弱组合**：
  - 持币 = 持续积累 BTC/ETH（Phase 3 注资 ≥$5k 后开始）
  - 月度买 30 天 OTM Put = 持续"保险"
  - **每月成本**：约 1-3% 名义本金 → 100 USDT 折算 1-3 USDT/月
  - **保护效果**：BTC 跌到 OTM Put 行权价时，本金有保险
- **regime-dependent 动态对冲**：承自《市场 Regime 检测与牛熊识别》§9
  - **bull / sideways regime**：可以减小 put 比例（市场情绪好，少买保险）
  - **bear / crisis regime**：放大 put 比例（市场危险，多买保险）
  - **自动触发**：当 HMM 隐藏状态切到 bear/crisis 且概率 > 0.7 → 触发加仓 put
- **ES 反向校准的期权对应**（承自《风险量化 VaR/CVaR/ES》§7）：
  - "日亏 -2% 熔断" = Long Put 行权价 92 USDT 的"自动止损保险"
  - **白旗熔断 -20 USDT** = "组合归零保险"——但实际工程上 Long Put 能在价格触达 K 时自动行权，比熔断更可靠

**10.4 关键设计：期权的"信息价值"**

承自《系统思维与反脆弱》§9 + 本文原则 4：

> 期权不只是"金融工具"——它是"信息工具"。Long Put 让项目能回答：
> "**如果明天崩盘，我能承受多少？我愿意花多少钱买这个保护？**"

这种"显式回答"比"含糊的熔断线"更接近反脆弱的"凸性结构"——因为期权费是"知道最大损失"的代价，熔断线是"不知道会不会触发"的赌注。

---

#### 第十一章　L1 风控 + L5 策略层 + L6 监控的期权扩展

**11.1 L1 风控的期权对应**

承自《自动量化项目架构初稿》L1 + 本文：

| L1 禁止项 | 期权对应 |
| --- | --- |
| **禁高杠杆方向单 (>2x)** | 禁止 Naked Short Call（无限亏损 = 高杠杆）；允许 Covered Call、Long Put |
| **禁网格 / 马丁裸跑** | 隐含——Short Straddle / Iron Condor 类似"无对冲卖保险" |
| **禁单所集中** | 期权持有本身不集中风险；但**对手方风险**（Deribit 出事）需要分散 |
| **人工干预需二次确认** | 期权平仓 → 二次确认 |

**新增禁止项（Phase 3）**：
- 禁 Naked Short Call（无标的覆盖 = 无限亏损）
- 禁 Naked Short Put（无现金覆盖 = 接近无限亏损）
- 禁 Single Leg 大额期权（< $50 名义不反对，> $1000 名义必须 spread 或组合对冲）

**11.2 L5 策略层新增 P3-A / P3-B**

承自《自动量化项目架构初稿》L5 + 本文：

**P3-A：Phase 3 集成 Deribit 期权（对冲）**
```
P3-A 任务清单：
  - Deribit API 接入（REST + WS 双通道）
  - IV 反推工具（py_vollib 或 vollib）
  - Greeks 计算工具链
  - 期权行权价选择规则（基于 portfolio drawdown）
  - 月度自动买 Put 脚本（cron）
  - 期权与现货的对账（多产品对账）
```

**P3-B：regime-dependent 动态对冲**
```
P3-B 任务清单：
  - 集成 Regime 检测模块的输出
  - 当 regime = bear 且概率 > 0.7 → 触发加仓 Put（自动开仓 0.5 USDT 名义 30d OTM Put）
  - 当 regime = crisis 且概率 > 0.6 → 触发加仓 Put（自动开仓 1 USDT 名义 14d ATM Put）
  - 当 regime = bull → 减小 Put 比例（不平仓，但下个月不再买新 Put）
```

**11.3 L6 监控层新增期权告警**

承自《自动量化项目架构初稿》L6 + 本文：

| 告警项 | 阈值 | 动作 |
| --- | --- | --- |
| **期权到期日临近** | T < 3 天 | 通知：决定是否平仓或滚动 |
| **Delta 偏离** | abs(delta) > 0.5（单份期权） | 触发对冲（用现货调整） |
| **Vega 暴露** | abs(vega) > 预算上限 | 触发 IV 跳变告警 |
| **IV 跳变** | DVOL 24h 变化 > 10 vol points | 触发期权头寸评审 |
| **Put/Call ratio 极端** | > 1.5 或 < 0.3 | 告警（市场情绪极端） |
| **持仓名义价值** | > 总资金 10% | 触发减仓评审 |

---

#### 第十二章　投资组合视角（持币 + Long Put = 反脆弱）

承自《系统思维与反脆弱》§6 凸性 + 《杠铃配置模板》+ 《反脆弱决策清单》§F：

**12.1 单一组合的凸性比较**

| 组合 | 下行 | 上行 | f''(payoff) | 凸性 |
| --- | --- | --- | --- | --- |
| **纯持币（HODL）** | 无封顶（跌无底） | 无封顶（涨无顶） | 0 | 线性 |
| **纯费率套利（同所做 funding_arb）** | 几乎无下行（funding 反转有熔断） | 有限上行（funding × 名义 × 时间） | ~0 | 接近线性 |
| **持币 + Long Put（Protective Put）** | 封顶在 K − 权利金 | 无封顶 | > 0 | **凸性** |
| **持币 + Covered Call** | 仍持币全部下行 | 封顶在 K + 权利金 | < 0 | **凹性** |
| **跨式多头（Long Straddle）** | 最多亏权利金 | 理论上无顶 | > 0 | **强凸性** |

**12.2 凸性组合的"显式答案"**

承自本文原则 4 + 《系统思维与反脆弱》§6：

> 持币 + Long Put = **Taleb 推荐的"反脆弱平衡"**的工程实现。
> 
> - 持币 = "凸性暴露"（无封顶 upside）= 长期增长
> - Long Put = "凸性保险"（下行有底）= 不被尾部归零
> - 组合 = "永远活着"的反脆弱结构

**12.3 与传统投资组合理论的对照**

- **Black-Litterman / Mean-Variance**：在 markowitz 框架下，持币 + Long Put 是"非有效组合"（不在 efficient frontier 上）——因为保险"降低期望收益"
- **反脆弱视角**：保险"降低期望收益"换来"不被尾部归零"——**长期复利的关键是"不被归零"，不是"期望收益最大化"**（承自《复利与非线性回报》§2.1 Buffett 1965-2024 19.9% 是"几乎每年正收益"，不是"最大化期望"）
- **结论**：反脆弱组合 vs markowitz 组合的差异 = **"活得久" vs "赚得多"**——长期看前者更优

**12.4 期权对冲的"组合保险"（CPPI）类比**

- **CPPI（Constant Proportion Portfolio Insurance）**：动态调整股票/债券比例，让组合价值不低于 floor
- **Long Put 类似**：floor = K − 权利金；超过 floor 的部分保留 upside
- **差异**：CPPI 是"连续 rebalance"，Long Put 是"事前一次性付费"——后者更简单但成本可能更高（取决于 IV）

---

### 第八部分　实战工具与回测（必含）

#### 第十三章　Python 工具栈 + BS 公式伪代码

**13.1 Python 期权工具栈对比**

承自 fast-vollib arXiv 2604.27210 + vollib GitHub + Implementing QuantLib + Investopedia：

| 工具 | 用途 | 优势 | 劣势 |
| --- | --- | --- | --- |
| **vollib / py_vollib** | BS、Black、BSM 定价 + IV 反推 + Greeks | API 简单、文档全 | 仅支持欧式 |
| **QuantLib** | 行业标准期权/利率衍生品库 | 支持美式、亚式、随机波动率 | 学习曲线陡 |
| **fast-vollib** | PyTorch/JAX/CUDA 加速 BS 定价 | 批量计算快 | 新库（2026） |
| **mibian** | BS、CRR、GBS 定价 + Greeks | 简单 | 文档少 |
| **options-pricing-model** | BS + 二叉树 + Monte Carlo | 全 | 性能一般 |
| **arch** | GARCH 波动率预测 | 与 BS 配合做 σ 输入 | 仅波动率 |
| **Deribit API** | 实时期权数据 + 交易 | 直接获取 IV + Greeks + OI | 仅 Deribit |

**13.2 BS 公式的 Python 实现（最小骨架）**

承自 vollib GitHub 范例 + fast-vollib arXiv：

```python
import numpy as np
from scipy.stats import norm

def bs_call(S, K, T, r, sigma):
    """Black-Scholes 欧式 Call 定价"""
    d1 = (np.log(S / K) + (r + 0.5 * sigma**2) * T) / (sigma * np.sqrt(T))
    d2 = d1 - sigma * np.sqrt(T)
    return S * norm.cdf(d1) - K * np.exp(-r * T) * norm.cdf(d2)

def bs_put(S, K, T, r, sigma):
    """Black-Scholes 欧式 Put 定价（or use Put-Call Parity）"""
    d1 = (np.log(S / K) + (r + 0.5 * sigma**2) * T) / (sigma * np.sqrt(T))
    d2 = d1 - sigma * np.sqrt(T)
    return K * np.exp(-r * T) * norm.cdf(-d2) - S * norm.cdf(-d1)

def implied_vol(market_price, S, K, T, r, flag='c'):
    """反推 IV（用 Brent 求解）"""
    from scipy.optimize import brentq
    def objective(sigma):
        if flag == 'c':
            return bs_call(S, K, T, r, sigma) - market_price
        else:
            return bs_put(S, K, T, r, sigma) - market_price
    return brentq(objective, 1e-6, 5.0)  # σ ∈ [0, 500%]

# Greeks（closed form）
def greeks_call(S, K, T, r, sigma):
    """Call 期权的 Delta / Gamma / Theta / Vega / Rho"""
    d1 = (np.log(S / K) + (r + 0.5 * sigma**2) * T) / (sigma * np.sqrt(T))
    d2 = d1 - sigma * np.sqrt(T)
    
    delta = norm.cdf(d1)
    gamma = norm.pdf(d1) / (S * sigma * np.sqrt(T))
    theta = -(S * norm.pdf(d1) * sigma) / (2 * np.sqrt(T)) - r * K * np.exp(-r * T) * norm.cdf(d2)
    vega = S * norm.pdf(d1) * np.sqrt(T)
    rho = K * T * np.exp(-r * T) * norm.cdf(d2)
    
    # 年化 theta（per year）；per day = theta / 365
    return {'delta': delta, 'gamma': gamma, 'theta': theta/365, 'vega': vega/100, 'rho': rho/100}
```

**13.3 Deribit 实时数据接入示例**

```python
import requests

def get_deribit_btc_options():
    """拉取 Deribit BTC 全部期权"""
    url = "https://www.deribit.com/api/v2/public/get_instruments"
    params = {"currency": "BTC", "kind": "option", "expired": False}
    r = requests.get(url, params=params).json()
    return r['result']

def get_deribit_btc_index():
    """拉取 BTC 指数价格"""
    url = "https://www.deribit.com/api/v2/public/get_index_price"
    params = {"index_name": "btc_usd"}
    r = requests.get(url, params=params).json()
    return r['result']['index_price']

def get_deribit_dvol():
    """拉取 BTC DVOL（实时 VIX-style 指数）"""
    url = "https://www.deribit.com/api/v2/public/get_volatility_index_data"
    params = {"currency": "BTC", "resolution": "60"}
    r = requests.get(url, params=params).json()
    return r['result']['data']  # [[timestamp, open, high, low, close], ...]
```

---

#### 第十四章　回测注意事项（IV 稳定性、mispricing、流动性、对冲频率）

**14.1 IV 稳定性的检验**

承自《回测方法论深化与 CPCV》§6 + 本文：

- **rolling IV 计算**：30 天滚动 DVOL 均值 + 标准差 → IV 稳定性判据
- **稳定性检验**：用 Ljung-Box test / Augmented Dickey-Fuller test 测 IV 序列是否平稳
- **结论**：加密 DVOL 长期非平稳（σ 跳变频繁）→ 回测必须 regime-aware（不能用单一 IV 参数）

**14.2 实际价格 vs BS 价格的 mispricing 统计**

- **回测模拟**：用 BS 公式 + 历史 σ 算"理论价"，对比实际期权市场成交价
- **mispricing 分布**：通常右偏（实际价 > BS 理论价，因为 BS 假设无跳跃）
- **加密特殊性**：加密期权 mispricing 比传统更大（BS 假设的恒定 σ 在加密不成立）→ 用 BS 理论价做回测会系统性低估实际权利金成本

**14.3 流动性成本（bid-ask spread + slippage）**

承自《市场微观结构与滑点建模》§1 + 本文：

- **加密期权 bid-ask spread**：Deribit ATM 期权 spread 约 0.5-2%（vs SPX ATM 期权 0.01%）
- **小资金回测的 spread 成本**：100 USDT 名义 × 1% spread = 1 USDT 往返成本
- **结论**：加密期权回测必须把 spread 作为"最低成本"——任何回测期望 < 1% 名义的策略都不可行

**14.4 期权对冲（delta hedging）的实际频率**

承自 §7.1 + 《做市策略与 Avellaneda-Stoikov》§3.4：

| 频率 | 适用 | 成本 |
| --- | --- | --- |
| 实时（每笔 tick） | HFT、做市商 | 极高（每秒多次 rebalance） |
| 1 秒级 | 主动做市商 | 高 |
| 1 分钟级 | 散户 deltahedging | 中（每次 0.5-2 USDT 名义成本） |
| 1 小时级 | 长期对冲 | 低 |
| 1 天级 | 季度对冲 | 极低（但 gamma 暴露大） |

**14.5 期权策略回测的 5 关卡 + 加密 RST**

承自《回测方法论深化与 CPCV》§7.2 + 本文：

| 关卡 | 期权专用修正 |
| --- | --- |
| ① 未来函数 + purged k-fold | 期权有到期日 → purge 时间必须覆盖到期后 IV 调整期（≥ 7 天） |
| ② CPCV PBO/DSR | regime-aware CPCV（按 IV regime 分桶：高 IV / 中 IV / 低 IV） |
| ③ 多周期 WFO | 加 (T−t) 滚动窗口（time decay 敏感） |
| ④ 块状 bootstrap MC | IV 时间序列 block_size = 7d（IV 周内有结构） |
| ⑤ 加密 RST | 加 BTC -30% + funding 反转 + IV 跳 30 vol points + ADL 触发 |

**任一关 FAIL 不准进入 paper trading**——期权策略在加密 RST 下的脆弱性比费率套利大得多（期权有 gamma 风险 = 价格跳变时 delta 跳变）。

**14.6 加密期权 paper trading 的特别注意**

- **流动性低** → paper trading 用 mid price 容易高估 fill rate；用 (bid + ask) / 2 + 模拟 fill rate < 70%
- **IV 跳变** → paper trading 用"昨日 IV"做 BS 定价会偏离实际；应该用"实时 IV 反推"
- **到期日 pin risk** → 回测必须处理"到期日附近的 delta 跳变"

---

## 自测

1. **（多空头风险结构）** Long Call / Short Call / Long Put / Short Put 各自的"下行风险 + 上行收益"是什么？哪个是凸性？哪个是凹性？为什么金融机构长期卖期权？2008 危机里 AIG 卖出 CDS 的尾部归零说明了什么？
2. **（BS 公式）** 写出 BS Call 公式 + d₁/d₂ + Put-Call Parity。5 大假设是什么？为什么"恒定 σ 假设"在加密市场完全不成立？BS 价格的几何直觉是什么？（为什么 C = S·N(d₁) − K·e^(−rT)·N(d₂)？）
3. **（IV 与 Skew）** IV 是什么？与历史波动率的区别是什么？什么是 IV Surface？什么是 IV Skew？加密市场 IV 的三大结构性特征是什么（与传统市场对比）？
4. **（Greeks）** Delta / Gamma / Theta / Vega / Rho 各自的定义 + 几何直觉 + 多空头方向。对一个 ATM Long Call 30 天到期：delta、gamma、theta、vega 的大致数量级是多少（用 BS 公式代入 S=100、K=100、T=30/365、r=0.04、σ=0.50 算一下）？
5. **（策略选择）** 长期持币 + 怕短期崩盘 → 哪个策略？长期持币 + 愿意在 X 价卖出 → 哪个策略？押注"未来 30 天 BTC 大波动" → 哪个策略？押注"未来 30 天 BTC 低波动" → 哪个策略？各策略的 max gain / max loss / 适用场景是什么？
6. **（加密市场）** Deribit 占加密期权市场份额多少？Deribit 24h 期权成交量、年度总量、BTC DVOL 当前水平、Deribit 与 IBIT 期权的 put/call ratio 差异是什么？加密期权 vs 传统期权的 5 大差异？
7. **（项目 Phase 定位）** 100 USDT 项目的 Phase 1/2/3 各自与期权的关系是什么？Phase 3 "持币 + Long Put" 的反脆弱数学基础是什么？月成本大概多少？regime-dependent 动态对冲的工程入口在哪？
8. **（风控映射）** 架构初稿 L1 的"白旗熔断 -20 USDT" 在期权视角下是什么？"日亏 -2% 熔断" 在期权视角下是什么？保护性 Put vs 熔断线的差异是什么？Phase 3 新增 L1 禁止项是什么？
9. **（VIX 与情绪）** VIX 的几何含义是什么（"30 天预期年化标准差"）？Deribit DVOL 与 VIX 的区别？2025 年 BTC DVOL 平均水平？DVOL 30+ vs 60+ 的市场含义？
10. **（实战工具）** 用 py_vollib 实现 BS 定价 + IV 反推需要哪几行代码？Deribit 实时数据接入需要哪个 API？加密期权回测的 5 关卡 + 特殊修正是什么？

---

## 资料来源（Tier 分级列表）

### Tier 1：原始论文（直接访问摘要/转述，未读全文）

- **Black, F. & Scholes, M. (1973)**. *The Pricing of Options and Corporate Liabilities.* Journal of Political Economy 81(3): 637-654. —— BS 公式原始论文；通过 Wikipedia "Black–Scholes model" 完整摘要 + Gregory Gundersen 2024 公式推导 + Investopedia 历史叙述交叉验证
- **Merton, R. C. (1973)**. *Theory of Rational Option Pricing.* Bell Journal of Economics and Management Science 4(1): 141-183. —— 提出 "Black-Scholes options pricing model" 术语；与 Scholes 共获 1997 诺奖
- **Merton, R. C. (1976)**. *Option Pricing When Underlying Stock Returns Are Discontinuous.* Journal of Financial Economics 3: 125-144. —— 跳跃扩散模型，BS 的扩展
- **Heston, S. L. (1993)**. *A Closed-Form Solution for Options with Stochastic Volatility with Applications to Bond and Currency Options.* Review of Financial Studies 6(2): 327-343. —— 随机波动率模型，加密的修正方向
- **Dupire, B. (1994)**. *Pricing with a Smile.* Risk Magazine 7(1): 18-20. —— Local volatility 与 IV Surface 的关系
- **Hull, J. C. (2017 第 10 版 / 2022 第 11 版)**. *Options, Futures, and Other Derivatives.* —— 第 15-17 章 BS、第 19 章 Greeks、第 26 章 IV Surface 的标准教材；**未直接读原书章节**，以 Wikipedia + Investopedia 二手转述

### Tier 2：监管/标准与行业白皮书（直接访问）

- **Cboe Global Markets (2024)** *Cboe Volatility Index Methodology (VIX)*. —— VIX 计算公式完整公开，URL: https://cdn.cboe.com/resources/indices/Volatility_Index_Methodology_Cboe_Volatility_Index.pdf
- **Cboe Global Markets (2019)** *Cboe Volatility Index Whitepaper*. —— VIX 历史与设计哲学
- **Deribit (2024-2026 实时)** *BTC Options Statistics*. —— DVOL、open interest、volume 数据，URL: https://www.deribit.com/statistics/BTC/metrics/options
- **Deribit (2025)** *Trading Volume $1,875B (2025)*. —— Deribit 官方公布的 2025 年度成交量

### Tier 3：综述与平台文档（直接访问 + 二手转述）

#### 概念与公式

- **Wikipedia "Black–Scholes model"** —— BS 5 大假设、公式、Put-Call Parity、风险中性测度推导
- **Wikipedia "Put–call parity"** —— 4 种等价形式、套利论证
- **Wikipedia "Greeks (finance)"** —— Δ/Γ/Θ/ν/ρ 完整定义、二阶 Greeks（Vanna/Charm/Veta/Color）、多资产期权（Correlation Delta）
- **Wikipedia "VIX"** —— CBOE Volatility Index 概念、1993 起源、2003 方法论重设计、2020-03 历史峰值 75.47、VVIX（Vol of Vol）
- **Wikipedia "Implied volatility"** + **Wikipedia "Volatility smile"** —— IV Surface、smile、skew、forward skew
- **Wikipedia "Option (finance)"** —— Call/Put、欧式/美式、行权方式
- **Wikipedia "Protective put"** + **Wikipedia "Covered call"** + **Wikipedia "Iron condor"** + **Wikipedia "Straddle"** + **Wikipedia "Strangle"** —— 5 种基础期权策略的定义与收益结构
- **Gregory Gundersen (2024)** *An Intuitive Explanation of Black–Scholes*. URL: https://gregorygundersen.com/blog/2024/09/28/black-scholes/ —— BS 的 Feynman-Kac 直觉推导（Tier 3 但质量极高）
- **Investopedia "Black-Scholes Model"** + **"Put-Call Parity"** + **"Option Greeks"** + **"Iron Condor"** + **"Protective Put"** + **"Covered Call"** —— 概念入门

#### 行业与平台

- **Investopedia "Option Greeks"** + **public.com "Options Greek Trading"** + **Optiver "Option Greeks"** —— Greeks 解释（含 vega 0.12 → 12 美分的具体例子）
- **Deribit 官方主页 + Insights blog** —— Deribit 历史、85% 市场份额、机构主导
- **FalconX (2025-10-02)** *Inside the Crypto Options Boom: Three Significant Shifts*. —— IBIT vs Deribit 对比、BTC IV 2025 长期低位（30-50%）、put/call ratio 对比（Deribit 0.5-0.6 vs IBIT 0.3）、BTC vs ETH IV 2024 中开始分化
- **Amberdata (2026-08-03)** *The Smile: Why Crypto's Skew Tells a Story Equity Markets Never Will*. —— BTC 25-delta RR = -3.46 vol points（90 天分布 98 百分位）；7D/30D/60D/90D RR 均负；fly/ATM 在 90 百分位
- **SpotGamma (2026-08-06)** *MSTR Volatility Skew*. —— MSTR call skew 在 2026 消失；IV 80-150% 期间
- **cf-stmoritz.com** *Crypto Options - A Fast-Growing Market*. —— Deribit ~85% OI、~80% 机构、期权占加密衍生品 ~3%
- **ResearchGate (2021)** *Implied volatility estimation of bitcoin options and the stylized facts of option pricing*. —— BTC IV smile 实证：forward volatility skew 明显；smile 在短到期最显著
- **Springer (2024)** *Deterministic modelling of implied volatility in cryptocurrency options*. —— IV smile 与 multiple resolution momentum indicator + 非线性 ML 回归
- **Amberdata Blog** + **medium @amberdata** —— 加密 IV 与传统市场的结构性差异
- **Deribit statistics dashboard**（2026-08-30 实时快照）—— 24h put volume 3,018 BTC vs call 4,413 BTC；open interest call 253,073 BTC vs put 142,270 BTC；BTC OI 名义 $30.88B；block trades breakdown（call spread 5.6% + call calendar spread 94% + iron condor 0.1%）

#### Python 工具

- **vollib GitHub**（github.com/vollib/py_vollib）—— Black-Scholes + Black + BSM 定价 + IV 反推 + Greeks
- **Implementing QuantLib Blog (2023-11)** *The Black-Scholes model in QuantLib*. —— QuantLib 行业标准期权库的使用
- **arXiv 2604.27210 (2026)** *fast-vollib: A Fast Implied Volatility Library for Python with PyTorch, JAX, and CUDA Fused-Kernel Backends*. —— PyTorch/JAX/CUDA 加速 BS + IV + Greeks

### Tier 1/2/3 综合使用说明

- 本条目中**所有 BS 公式与 Greeks 定义**来自 Tier 1 论文摘要或 Wikipedia/Investopedia 标准形式；**未直接读** Hull《Options, Futures, and Other Derivatives》原书章节、Black-Scholes 1973 原文 PDF、Merton 1973 原文 PDF。
- **加密期权市场数据**（Deribit 市场份额、24h 成交量、BTC DVOL、put/call ratio）来自 Tier 2 Deribit 官方 statistics 页面 + Tier 3 FalconX 2025-10 报告 + cfc-stmoritz 行业综述。
- **加密 IV skew 数据**（BTC 25-delta RR、IV surface 形状、forward skew）来自 Tier 3 Amberdata 2026-08 + ResearchGate 2021 BTC IV 实证 + FalconX 2025-10 + SpotGamma 2026-08。
- **VIX 数据**（30 天预期年化 σ 公式、2020-03 历史峰值 75.47、VVIX 概念）来自 Tier 2 Cboe VIX Methodology PDF + Tier 3 Wikipedia VIX 页面。
- **Python 工具栈**（vollib、QuantLib、fast-vollib、Deribit API）来自 Tier 3 vollib GitHub + fast-vollib arXiv + Implementing QuantLib blog + Deribit 官方 API 文档。
- **项目应用映射**（Phase 1/2/3 期权定位、L1 禁止项扩展、ES 反向校准的期权对应）来自《自动量化项目架构初稿》（同目录）+ 《风险量化 VaR/CVaR/ES》§7 + 《系统思维与反脆弱》§6 + 本文作者综合判断。
- **数据点**（Deribit $1.875B 2025 成交量、24h snapshot 数据、BTC DVOL 当前 30-50%）以 2026-08-30 行情为准；**费率、IV 随市场变化，上线前以 Deribit 实时数据为准**。

---

## 与本条交叉验证的其他外脑条目

- **《系统思维与反脆弱》**（外脑 思维模型 2026-08-30）§6 凸性 —— Long Call/Long Put = 凸性组合的纯数学表达
- **《复利与非线性回报》**（外脑 思维模型 2026-08-30）§2 期权 = 凸性 + §9.3 金融凸性例子 —— 本文思维层基础
- **《杠铃配置模板》**（外脑 思维模型 2026-08-30）—— 90 USDT 费率套利 + 10 USDT 期权对冲实验 = 项目级杠铃
- **《反脆弱决策清单》**（外脑 思维模型 2026-08-30）§F 重大金额交易的反例 —— 期权对冲的决策清单
- **《自动量化项目架构初稿》**（外脑 量化交易 2026-08-30）—— L1/L5/L6 是本文 §10/§11 的工程依据；risk.yaml 是 Phase 3 集成期权时的扩展源
- **《自动量化加密货币的成功与失败》**（外脑 量化交易 2026-08-30）§3.5 账户级对冲 + §9 funding arb 细节 —— 本文核心引用
- **《市场微观结构与滑点建模》**（外脑 量化交易 2026-08-30）—— IV 反映预期滑点；期权费包含逆向选择
- **《风险量化 VaR/CVaR/ES》**（外脑 量化交易 2026-08-30）§7 L1 熔断线反向校准 —— 保护性 Put = 给 ES 熔断线"买保险"
- **《市场 Regime 检测与牛熊识别》**（外脑 量化交易 2026-08-30）—— regime 切换 → 动态调整 put 比例的工程入口
- **《做市策略与 Avellaneda-Stoikov》**（外脑 量化交易 2026-08-30）第 11 章 加密 maker 返佣算账 —— 期权做市的库存管理（gamma scalping）
- **《回测方法论深化与 CPCV》**（外脑 量化交易 2026-08-30）§6/§7.2 —— 期权回测的特殊性（IV 稳定性、mispricing、regime-aware CPCV）
- **《统计套利与 Pairs Trading》**（外脑 量化交易 2026-08-30）—— 期权组合对冲的扩展
- **《行为金融与交易者心理偏差》**（外脑 量化交易 2026-08-30）§6 —— 处置效应 + 心理止损 = 期权卖方的"心理债"
