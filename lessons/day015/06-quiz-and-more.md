# ⑥ 自测题与延伸阅读

> 这几道题没有标准答案，目的是逼你用自己的话把知识讲清楚。答不上来的，回到对应小节再看一遍。

## 自测题

### Q1：你看到一篇研究说「用沪深300价格直接做回归得到 R² = 0.95」，你怎么判断这个结论的可信度？

<details>
<summary>💡 答题思路提示（先自己想，再点开）</summary>

核心词：**伪回归**。

沪深300 价格是非平稳序列（课程实测 ADF p = 0.5953）。两个非平稳序列哪怕完全独立，普通回归也有 75% 概率跑出 R² 高、p < 0.05 的「显著」结果（Granger 1974）。所以第一反应应该是质疑：为什么不做对数收益？正确的流程是先各自做 ADF，非平稳就转收益率，想做配对就补 Engle-Granger 协整 + spread ADF。回「[① 平稳性](lessons/day015/01-stationarity.md)」复习。

</details>

### Q2：ADF 检验 p = 0.06，你应该说「平稳」还是「不平稳」？为什么？

<details>
<summary>💡 答题思路提示（先自己想，再点开）</summary>

说**不平稳**。

ADF 的原假设 H₀ 是「序列非平稳（有单位根）」，p = 0.06 > 0.05，无法拒绝 H₀——注意这不是「证明了不平稳」，而是「证据不足以拒绝不平稳」。实务上按非平稳处理：该差分差分，该转对数收益就转。门槛是纪律，0.06 和 0.04 没有本质区别，别在边界上讨价还价。回「[① 平稳性](lessons/day015/01-stationarity.md)」复习。

</details>

### Q3：中信和华泰按教科书应该协整，但实测 p = 0.2852 不成立。如果你硬上配对仓位，会发生什么？

<details>
<summary>💡 答题思路提示（先自己想，再点开）</summary>

spread 的 ADF p = 0.1191，不平稳——意味着价差**没有回归均值的机制**。

硬上配对的剧本是：spread 越漂越远，你设的 ±2σ 入场信号触发后，价格不回来反而继续走扩，「回归均值」的机会始终不出现，持仓越亏越多。底层原因：2024 国九条新规影响不对称 + 中信（投行/并购）与华泰（资管/金融科技）业务路径分化 + 市值差异（约 4500 亿 vs 约 1800 亿）。教训：协整检验不通过，hedge ratio 算得再漂亮也不能交易。回「[② 差分与协整](lessons/day015/02-diff-and-cointegration.md)」复习。

</details>

## 一页总结（建议截图，3 天后再看一遍）

1. 时间序列 = 按时间顺序的数据，内部结构不能打乱；
2. 平稳性三要求：均值不变 / 方差不变 / 协方差只跟时间差有关；
3. ADF 检验：H₀ 是「不平稳」，p < 0.05 拒绝 → 平稳；
4. 实测沪深300：价格 ADF p = 0.60 不平稳 / 收益 ADF p = 0.0000 平稳——教科书完美对照；
5. 协整：y 和 x 都不平稳，但 y − βx 平稳 → 配对交易的理论根基；
6. Engle-Granger 检验 = 对 OLS 残差做 ADF，p < 0.05 协整成立；
7. 实测中信 vs 华泰（2024–2025）协整 p = 0.2852 不成立——教科书直觉被现实推翻，必须实测；
8. 复检纪律：每季度 coint 一次，关系不显著立即清仓。

> [!TIP]
> 今天的三行代码（ADF → 差分 → coint）是后面所有时间序列内容的入场券。判断「数据能不能用」应该变成肌肉记忆。

## 延伸阅读

- Engle & Granger《Co-integration and Error Correction: Representation, Estimation, and Testing》（Econometrica 1987）——协整理论奠基，2003 年诺贝尔奖核心论文；
- Dickey & Fuller《Distribution of the Estimators for Autoregressive Time Series with a Unit Root》（JASA 1979）——ADF 检验原始论文；
- Granger《Spurious Regressions in Econometrics》（JoE 1974）——揭示伪回归危害的奠基论文；
- Hamilton《Time Series Analysis》（Princeton 1994）——时间序列经典教材，平稳性 / 单位根 / 协整数学最严谨；
- Vidyamurthy《Pairs Trading: Quantitative Methods and Analysis》（Wiley 2004）——配对交易工程师必读，从协整理论到实盘代码；
- Gatev / Goetzmann / Rouwenhorst《Pairs Trading: Performance of a Relative-Value Arbitrage Rule》（RFS 2006）——配对交易实证研究经典；
- statsmodels.tsa.stattools 文档（adfuller / coint / kpss）——Python 量化时间序列标配。

## 下一课预告

**Day 016 · 自相关与预测** —— 讲完平稳性，看时间序列的下一个核心问题：一个序列跟自己「过去版本」的关系。ACF / PACF / Ljung-Box 检验 / Hurst 指数 / 短期反转 vs 长期动量。实测有反直觉：近十年标普500月度其实是动量（+0.10）不是反转，黄金和美元的 Hurst 都小于 0.5，呈反转特性。
