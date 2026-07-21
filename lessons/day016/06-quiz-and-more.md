# ⑥ 自测题与延伸阅读

> 这几道题没有标准答案，目的是逼你用自己的话把知识讲清楚。答不上来的，回到对应小节再看一遍。

## 自测题

### Q1：为什么价格的自相关必然接近 1，而收益的自相关才是有意义的？

<details>
<summary>💡 答题思路提示（先自己想，再点开）</summary>

核心词：**非平稳的累加过程**。

今天的价格 ≈ 昨天的价格 + 一个小变化，价格序列是收益一路累加出来的，相邻两天天然高度相似——所以 ACF 接近 1 是数学必然，不含任何可交易信息。而收益是差分后的「变化幅度」，它的自相关回答的是「今天的变化能不能预测明天的变化」——这才是动量 / 反转策略的原材料。回「[① 自相关与 Ljung-Box](lessons/day016/01-acf-and-ljungbox.md)」和「[Day 015 · 平稳性](lessons/day015/01-stationarity.md)」复习。

</details>

### Q2：标普500 日收益 ρ(r) = 0.014 看似没信号，但 ρ(|r|) = 0.124 显著大于零。你怎么用这两个数字？

<details>
<summary>💡 答题思路提示（先自己想，再点开）</summary>

把两个数字分到两个不同的策略篮子：

- ρ(r) = 0.014 ≈ 0 → **方向预测放弃**：别做「猜明天涨跌」的策略，美股大盘效率太高；
- ρ(|r|) = 0.124 > 0 → **波动率预测可用**：今天波动大，明天大概率波动也大。可以做波动率目标仓位（波动高降仓、低加仓）、VaR 用 GARCH 动态波动率、期权端做波动率交易。

一句话：方向猜不到，波动猜得到——这两件事必须分开做。回「[② 波动率聚集与 Hurst](lessons/day016/02-volatility-and-hurst.md)」复习。

</details>

### Q3：教科书说「短期反转」，但实测标普近十年月度 +0.10 是动量。你应该信教科书还是信实测？为什么？

<details>
<summary>💡 答题思路提示（先自己想，再点开）</summary>

信**实测**——但加上两个限定词：信「最近 3–5 年」的实测，并且「每季度复检」。

理由：教科书的「短期反转」结论建立在 1980–90 年代美股数据上，而近十年美股是长牛 + FAANG 拉动，趋势力量压过均值回归，市场结构已经变了。任何统计规律都有时效性，实测才是真理。但也别走到另一个极端——+0.10 也只是近十年的结论，所以策略上线后要滚动重测 Hurst + 月度 ρ，关系反转就出场。回「[② 波动率聚集与 Hurst](lessons/day016/02-volatility-and-hurst.md)」复习。

</details>

## 一页总结（建议截图，3 天后再看一遍）

1. 自相关 = 序列跟自己滞后版本的相关性；ACF 看所有 lag 的总相关，PACF 去除中间环节看直接关系；
2. Ljung-Box 检验：p < 0.05 → 不是白噪声 → 有可预测信号；
3. 收益自相关 ≈ 0（本身不可预测），但 |收益| 自相关 > 0（波动率高度可预测）——这就是波动率聚集；
4. 实测：标普 ρ(r) = 0.014 / 美元指数 ρ(|r|) = 0.365——美股效率最高，美元波动率记忆最强；
5. Hurst 指数：> 0.5 趋势 / = 0.5 随机 / < 0.5 反转；
6. 反直觉实测：近十年标普月度 +0.10 是动量（非反转）/ 黄金 H = 0.434 反转（非趋势）/ 美元 H = 0.409 反转；
7. ARCH/GARCH 是波动率预测标配，Engle 2003 年诺贝尔奖核心贡献；
8. 教训：任何动量 / 反转策略必须用最近 3–5 年实测验证，教科书结论易失效。

> [!TIP]
> 到今天为止，时间序列的两块地基都打完了：[Day 015](lessons/day015/README.md) 的「数据能不能用」（平稳性）+ 今天的「数据里有没有信号」（自相关）。后面的 ARIMA、GARCH、配对交易全是这两块地基上的楼。

## 延伸阅读

- Bachelier《Théorie de la spéculation》（1900 年博士论文）——随机游走和金融数学的起源；
- Mandelbrot《The Variation of Certain Speculative Prices》（JoB 1963）——波动率聚集首次记录；
- Mandelbrot《The (Mis)Behavior of Markets》（Basic Books 2004）——写给非学术读者的厚尾 / 分形 / 波动率聚集科普，可读性极强；
- Engle《Autoregressive Conditional Heteroscedasticity with Estimates of the Variance of United Kingdom Inflation》（Econometrica 1982）——ARCH 模型，2003 年诺贝尔奖核心；
- Bollerslev《Generalized Autoregressive Conditional Heteroscedasticity》（JoE 1986）——GARCH 模型；
- Tsay《Analysis of Financial Time Series》——自相关 / ACF / PACF / GARCH 篇章是金融时序的标准答案；
- Jegadeesh & Titman《Returns to Buying Winners and Selling Losers》（JF 1993）——动量策略经典。

## 下一课预告

**Day 017 · 敬请期待** —— 下一批课程正在排期中。建议趁这个间隙把 Day 015–016 的代码换成你自己关注的标的跑一遍：先 ADF 看平稳性，再 ACF + Ljung-Box + Hurst 看信号——两张「体检表」就是你以后分析任何时间序列的 standard 开场。
