# ③ Python 实战：策略 t 检验 + 多次检验仿真

> 三个实验：给真实的沪深 300 均线策略做 t 检验；跑 1000 个随机策略看假阳性；换 50 个回测窗口演示 p-hacking。

复制到 Jupyter / VS Code 直接能跑。依赖：`pip install numpy pandas yfinance scipy matplotlib`。

> 实操原则：先跑通（哪怕硬编码），再优化（参数化/函数化），最后才追求精度。散户量化最大的坑是没跑通就开始优化。

## 完整代码

```python
# day_013_hypothesis.py — 假设检验在量化策略中的实战
import numpy as np, pandas as pd, yfinance as yf
import scipy.stats as stats
import matplotlib.pyplot as plt

# ============ 1. 真实策略 t 检验：沪深 300 均线策略 ============
# 拉沪深 300ETF（510300）近 3 年日线
px = yf.download('510300.SS', period='3y', auto_adjust=True)['Close'].squeeze()
ret = px.pct_change().dropna()

# 简单均线策略：收盘价站上 20 日均线则持有，跌破则空仓（信号滞后一天避免未来函数）
ma20 = px.rolling(20).mean()
signal = (px > ma20).shift(1).fillna(False).astype(int)
strat_ret = (signal * ret).dropna()

# t 检验：H₀ = 策略日均收益 = 0
t_stat, p_val = stats.ttest_1samp(strat_ret, popmean=0)
n = len(strat_ret)
annualized = strat_ret.mean() * 252
sharpe = strat_ret.mean() / strat_ret.std() * np.sqrt(252)

result = pd.DataFrame({
    '指标': ['样本量(天)', '日均收益(%)', '年化(%)', 'Sharpe', 't 统计量', 'p 值'],
    '数值': [n, strat_ret.mean() * 100, annualized * 100, sharpe, t_stat, p_val],
})
print('=== 沪深三百 20 日均线策略 t 检验 ===')
print(result.to_string(index=False))
verdict = '显著' if p_val < 0.05 else '不显著(可能纯运气)'
print(f'\n判断:{verdict}(α=0.05)')

# ============ 2. 多次检验仿真：1000 个随机策略，看几个「假阳性」 ============
np.random.seed(42)
n_strategies, n_days = 1000, 252
p_values = []
for _ in range(n_strategies):
    fake_ret = np.random.normal(0, 0.01, n_days)  # 真实均值就是 0，纯噪声
    _, p = stats.ttest_1samp(fake_ret, 0)
    p_values.append(p)
p_values = np.array(p_values)

false_positives = (p_values < 0.05).sum()
bonferroni = (p_values < 0.05 / n_strategies).sum()
print(f'\n=== 1000 个随机策略多次检验 ===')
print(f'按 α=0.05 假阳性数 = {false_positives}(理论 50)')
print(f'Bonferroni 修正(α=0.00005)假阳性数 = {bonferroni}')

# ============ 3. p-hacking 演示：同一策略，换不同回测窗口 ============
print(f'\n=== p-hacking 演示:同一策略,换不同回测窗口 ===')
windows = [strat_ret.iloc[i:i + 150] for i in range(0, len(strat_ret) - 150, 10)]
p_per_window = [float(stats.ttest_1samp(w, 0).pvalue) for w in windows if len(w) == 150]
min_p_w = min(p_per_window)
print(f'最小 p 值 = {min_p_w:.4f} (在 {len(p_per_window)} 个窗口中)')
print(f'按 α=0.05 数 = {sum(p < 0.05 for p in p_per_window)}/{len(p_per_window)}')
print(f'⚠ 只报告最小 p 值就是 p-hacking,真实显著性远低于这个数字')

# ============ 4. 可视化：p 值分布 + 策略累计收益 ============
fig, axes = plt.subplots(1, 2, figsize=(14, 5))
axes[0].hist(p_values, bins=50, color='steelblue', edgecolor='black')
axes[0].axvline(0.05, color='red', linestyle='--', linewidth=2, label='α=0.05')
axes[0].set_title(f'1000 个随机策略 p 值分布 · 假阳性 {false_positives} 个')
axes[0].set_xlabel('p 值')
axes[0].set_ylabel('频数')
axes[0].legend()

axes[1].plot(strat_ret.cumsum().values, color='steelblue')
axes[1].set_title(f'沪深三百均线策略累计收益(t={t_stat:.2f}, p={p_val:.4f})')
axes[1].set_xlabel('交易日')
axes[1].set_ylabel('累计收益')
axes[1].grid(True, alpha=0.3)

plt.tight_layout()
plt.savefig('day013_hypothesis.png', dpi=120)
print('\n✓ 图已保存到 day013_hypothesis.png')
```

## 逐段解释

**第 1 段 · 真实策略 t 检验。** 拉沪深 300ETF 三年数据，构造最简单的 20 日均线策略（线上持有、线下空仓）。注意 `shift(1)`：今天的信号明天才执行，避免「看着今天的收盘价做今天的决定」这种未来函数。然后 `ttest_1samp(strat_ret, 0)` 一行完成检验，输出 t 和 p。

**第 2 段 · 多次检验仿真。** 生成 1000 组**真实均值为 0** 的纯噪声「策略」，每组都做 t 检验。由于它们本质上全是运气，所有 p < 0.05 的结果都是假阳性。理论期望是 1000 × 0.05 = 50 个。再用 Bonferroni（阈值 0.05/1000）看假阳性剩几个。

**第 3 段 · p-hacking 演示。** 把同一个均线策略切成一堆 150 天的滑动窗口，每个窗口单独做 t 检验，然后看：总有一个窗口 p 值特别小。如果你只把这个窗口拿出来说「策略显著」，这就是教科书级的 p-hacking。

**第 4 段 · 可视化。** 左图：1000 个随机策略的 p 值分布——理论上它是均匀分布（H₀ 为真时的数学性质），所以约 5% 会落在红线左边。右图：真实策略的累计收益曲线，标题里带 t 和 p。

## 样本输出（课程样本）

第 1 段的 t、p 取决于你运行时的最新三年数据（以实际输出为准），表格形如：

```text
=== 沪深三百 20 日均线策略 t 检验 ===
        指标          数值
   样本量(天)      约 730
  日均收益(%)      随数据而定
     年化(%)       随数据而定
      Sharpe       随数据而定
    t 统计量       随数据而定
       p 值        随数据而定
```

第 2、3 段固定随机种子，课程给出的关键读数：

```text
=== 1000 个随机策略多次检验 ===
按 α=0.05 假阳性数 ≈ 50(理论 50)
Bonferroni 修正(α=0.00005)假阳性数 ≈ 0

=== p-hacking 演示:同一策略,换不同回测窗口 ===
最小 p 值 << 0.05（几十个窗口中总有「幸运窗口」）
⚠ 只报告最小 p 值就是 p-hacking,真实显著性远低于这个数字
```

课件图表分析的三个要点：

| 发现 | 数字 | 含义 |
| --- | --- | --- |
| 随机策略 p 值均匀分布 | ~50/1000 个 p<0.05 | 假阳性是数学必然，不是意外 |
| Bonferroni 修正 | 假阳性压到 ~0 | 但代价是可能误杀真信号，可用 FDR 折中 |
| Renaissance 标准 | t > 5（p < 5×10⁻⁷） | 比学术 0.05 严 100 万倍，30 年无年度亏损 |

> [!IMPORTANT]
> **评估一个策略必须三件套：Sharpe + p 值 + power（样本量够不够）。** 一个 Sharpe 1.5 的策略如果只有 6 个月样本，p 值可能 > 0.1——你不能确定它是真本事。光看年化收益和回撤曲线，是在自欺欺人。
