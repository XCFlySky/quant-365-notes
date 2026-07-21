# ③ Python 实战：ADF + 中信/华泰协整检验

> 复制到 Jupyter / VS Code 直接能跑，`pip install yfinance statsmodels matplotlib` 缺什么装什么。原则：先跑通，再优化，最后才追求精度。

## 完整代码

```python
# day_015_stationarity.py — 价格 vs 收益率 ADF + 中信/华泰券商对子协整
import numpy as np
import pandas as pd
import yfinance as yf
import matplotlib.pyplot as plt
from statsmodels.tsa.stattools import adfuller, coint
from statsmodels.regression.linear_model import OLS
import statsmodels.api as sm

# ============ 1. 拉数据 ============
tickers = {
    '沪深300': '510300.SS',
    '中信证券': '600030.SS',
    '华泰证券': '601688.SS',
}
raw = yf.download(list(tickers.values()), period='5y', auto_adjust=True)['Close']
# yfinance 按字母序返回列 → 用 dict 显式按 ticker 名字映射，避免列错位
raw = pd.DataFrame({name: raw[ticker] for name, ticker in tickers.items()})
raw = raw.dropna()
ret = raw.pct_change().dropna()          # 简单收益；对数收益用 np.log(raw).diff()
print(f'数据样本量: {len(raw)} 个交易日')

# ============ 2. 价格 vs 收益率 ADF 对比 ============
print('\n=== 沪深300 价格 vs 收益率 ADF 检验 ===')
for label, series in [('沪深300 价格', raw['沪深300']),
                      ('沪深300 日收益率', ret['沪深300'])]:
    stat, p, *_ = adfuller(series, autolag='AIC')
    verdict = '平稳' if p < 0.05 else '非平稳(有单位根)'
    print(f'  {label:<20} | ADF stat = {stat:.3f}  p = {p:.4f} → {verdict}')

# ============ 3. 中信 vs 华泰 协整检验（配对交易理论根基）============
print('\n=== 中信证券 vs 华泰证券 协整检验 ===')
y, x = raw['中信证券'], raw['华泰证券']
for name, series in [('中信证券价格', y), ('华泰证券价格', x)]:
    stat, p, *_ = adfuller(series, autolag='AIC')
    print(f'  {name:<15} ADF p = {p:.4f}（预期 > 0.05 即非平稳）')

# Engle-Granger 协整检验
score, pval, _ = coint(y, x)
print(f'\nEngle-Granger 协整检验 t = {score:.3f}, p = {pval:.4f}')
print('协整成立(可做配对)' if pval < 0.05 else '协整不成立(不能配对)')

# 用 OLS 拟合 hedge ratio：中信 ≈ α + β × 华泰
X_with_const = sm.add_constant(x)
model = OLS(y, X_with_const).fit()
alpha, beta = model.params.iloc[0], model.params.iloc[1]
spread = y - beta * x - alpha
print(f'\nhedge ratio: 中信 ≈ {alpha:.2f} + {beta:.3f} × 华泰')
print(f'spread 均值 = {spread.mean():.3f}, 标准差 = {spread.std():.3f}')

# spread 自身做 ADF —— 协整成立意味着 spread 必须平稳
sp_stat, sp_p, *_ = adfuller(spread, autolag='AIC')
print(f'spread ADF p = {sp_p:.4f} → ' +
      ('平稳(可做配对)' if sp_p < 0.05 else '不平稳(配对失效)'))

# ============ 4. 可视化 ============
fig, axes = plt.subplots(2, 2, figsize=(14, 9))

ax = axes[0, 0]
ax.plot(raw.index, raw['沪深300'], color='steelblue')
ax.set_title('沪深300 价格 — 非平稳(均值随时间漂移)')
ax.set_ylabel('价格'); ax.grid(True, alpha=0.3)

ax = axes[0, 1]
ax.plot(ret.index, ret['沪深300'] * 100, color='tab:green', linewidth=0.5)
ax.axhline(0, color='black', linewidth=0.5)
ax.set_title('沪深300 日收益率 — 平稳(均值近零)')
ax.set_ylabel('日收益 (%)'); ax.grid(True, alpha=0.3)

ax = axes[1, 0]
ax.plot(raw.index, raw['中信证券'], color='tab:orange', label='中信证券')
ax.plot(raw.index, raw['华泰证券'], color='tab:blue', label='华泰证券')
ax.set_title(f'中信 vs 华泰券商对子 · 协整 p = {pval:.3f}')
ax.set_ylabel('价格'); ax.legend(); ax.grid(True, alpha=0.3)

ax = axes[1, 1]
ax.plot(spread.index, spread, color='tab:red')
ax.axhline(spread.mean(), color='black', linestyle='--',
           label=f'均值 {spread.mean():.2f}')
ax.axhline(spread.mean() + 2 * spread.std(), color='gray', linestyle=':',
           label='±2σ')
ax.axhline(spread.mean() - 2 * spread.std(), color='gray', linestyle=':')
ax.set_title(f'spread = 中信 − {beta:.2f}×华泰 − {alpha:.1f} · ADF p = {sp_p:.3f}')
ax.set_ylabel('spread'); ax.legend(); ax.grid(True, alpha=0.3)

plt.tight_layout()
plt.savefig('day015_stationarity.png', dpi=120)
print('\n✓ 图已保存到 day015_stationarity.png')
```

## 逐段解释

**第 1 段 · 拉数据**：用 yfinance 拉 5 年日线收盘价。注意 yfinance 返回的列按字母序排列，直接用中文名索引会错位，所以用 dict 显式映射一遍——这是实盘代码里很常见的坑。

**第 2 段 · 价格 vs 收益率 ADF**：`adfuller(series, autolag='AIC')` 一行出结果，只看返回的第二个值（p 值）。p < 0.05 → 平稳。

**第 3 段 · 协整三步走**：

1. 中信、华泰各自做 ADF——两个都非平稳，才满足协整前提；
2. `coint(y, x)` 做 Engle-Granger 检验，p < 0.05 才算协整成立；
3. 用 OLS 拟合 hedge ratio（β），构造 spread = y − β·x − α，**再对 spread 本身做 ADF**——spread 平稳才是真正能交易的信号。

**第 4 段 · 可视化**：四张图——价格（漂移）、收益率（绕零波动）、两只股票走势、spread 与 ±2σ 通道。

## 课程样本输出（2024–2025 实测）

```text
=== 沪深300 价格 vs 收益率 ADF 检验 ===
  沪深300 价格     | ADF p = 0.5953 → 非平稳(有单位根)
  沪深300 日收益率 | ADF p = 0.0000 → 平稳

=== 中信证券 vs 华泰证券 协整检验 ===
Engle-Granger 协整检验 t = -2.486, p = 0.2852
协整不成立(不能配对)

hedge ratio: 中信 ≈ 4.67 + 1.162 × 华泰
spread 均值 ≈ 0, 标准差 = 1.645
spread ADF p = 0.1191 → 不平稳(配对失效)
```

| 检验 | p 值 | 解读 |
| --- | --- | --- |
| 沪深300 价格 ADF | 0.5953 | 远超 0.05 → 非平稳，均值随时间漂移 |
| 沪深300 收益率 ADF | 0.0000 | 极小 → 平稳，围绕零均值波动 |
| 中信 vs 华泰协整 | 0.2852 | 不成立——头部券商协整失效，反直觉 |
| spread ADF | 0.1191 | 不平稳 → 硬上配对会越亏越多 |

> [!IMPORTANT]
> 读输出时盯住一条纪律：**hedge ratio 算得出来 ≠ 配对做得成**。β 只是 OLS 的数学拟合结果，spread 不平稳，这个 β 就没有交易价值。看到任何「股价直接回归」的研究材料，默认让对方先解释为什么不做对数收益。
