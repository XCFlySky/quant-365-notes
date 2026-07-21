# ③ Python 实战：四个标的的厚尾真相

> 沪深三百 / 标普 / 纳斯达克 / 比特币，直方图 + 正态拟合 + Q-Q 图 + 极端日清单，一次把「正态假设」按在地上摩擦。

> [!TIP]
> 实操原则：**先跑通（哪怕硬编码），再优化（参数化/函数化），最后才追求精度**。散户量化最大的坑是没跑通就开始优化。

## 准备工作

```bash
pip install numpy pandas yfinance matplotlib scipy
```

## 第一步：逐个下载，避免交易日历互相污染

```python
# day_010_normal_distribution.py — 沪深三百/标普/纳斯达克/比特币 厚尾真相
import numpy as np
import pandas as pd
import yfinance as yf
import matplotlib.pyplot as plt
from scipy import stats

tickers = {
    '沪深三百': '000300.SS',
    '标普五百': '^GSPC',
    '纳斯达克': '^IXIC',
    '比特币':   'BTC-USD',
}

# 每个标的单独下载，避免不同交易日历强行 dropna 把样本清空
returns = {}
for name, sym in tickers.items():
    raw = yf.download(sym, period='5y', auto_adjust=True, progress=False)
    if raw is None or raw.empty:
        print(f'× {name} 拿不到数据'); continue
    s = raw['Close']
    if isinstance(s, pd.DataFrame):
        s = s.iloc[:, 0]
    r = s.pct_change().dropna() * 100          # 百分比收益
    if len(r) < 100:
        print(f'× {name} 样本太少 {len(r)}'); continue
    returns[name] = r
    print(f'✓ {name}: {len(r)} 天 ({r.index.min().date()} ~ {r.index.max().date()})')

names = list(returns.keys())
```

**解释：** 四个标的有三个交易日历（A 股 / 美股 / 加密 7×24），必须各下各的。这是 Day 009 学到的教训的直接应用。

## 第二步：描述统计——均值/标准差/偏度/峰度

```python
print('=== 五年日收益描述统计(单位:%) ===')
print(f'{"标的":<10}{"均值":>8}{"标准差":>10}{"偏度":>8}{"峰度":>10}')
for name in names:
    r = returns[name]
    print(f'{name:<10}{r.mean():>8.3f}{r.std():>10.2f}'
          f'{stats.skew(r):>8.2f}{stats.kurtosis(r):>10.2f}')
```

**样本输出（课程样本）：**

```text
标的        均值      标准差    偏度      峰度
沪深三百     0.001      1.11    0.21      7.09
标普五百     0.051      1.06    0.16      7.17
纳斯达克     0.057      1.41    0.21      5.54
比特币       0.131      2.49    0.30      3.44
```

> [!NOTE]
> 正态分布的偏度 = 0、超额峰度 = 0。四个标的峰度全部显著大于 0——**全部厚尾**，没有一个像教科书。

## 第三步：68-95-99.7 法则实测 vs 理论

```python
print('=== 68-95-99.7 法则:实测 vs 理论 ===')
print(f'{"标的":<10}{"|z|<1":>10}{"|z|<2":>10}{"|z|<3":>10}')
print(f'{"理论值":<10}{"68.3%":>10}{"95.4%":>10}{"99.7%":>10}')
for name in names:
    r = returns[name]
    z = (r - r.mean()) / r.std()        # 标准化：每个收益是几个 σ
    p1 = (np.abs(z) < 1).mean() * 100
    p2 = (np.abs(z) < 2).mean() * 100
    p3 = (np.abs(z) < 3).mean() * 100
    print(f'{name:<10}{p1:>9.1f}%{p2:>9.1f}%{p3:>9.1f}%')
```

**样本输出（课程样本）：**

```text
标的        |z|<1      |z|<2      |z|<3
理论值       68.3%      95.4%      99.7%
沪深三百      76.9%      95.5%      98.8%
标普五百      75.5%      95.1%      98.9%
纳斯达克      74.3%      95.2%      99.0%
比特币        77.0%      94.7%      98.5%
```

**怎么读：** ±1σ、±2σ 两列跟理论几乎一致（中段成立）；±3σ 一列全部低于 99.7%——**缺口全在尾巴上**。

## 第四步：极端日清单——实测 vs 正态预测

```python
print('=== 超过 3σ 与 5σ 的天数 ===')
for name in names:
    r = returns[name]
    z = (r - r.mean()) / r.std()
    n_total = len(z)
    n3 = int((np.abs(z) > 3).sum())                  # 实测 >3σ 天数
    n5 = int((np.abs(z) > 5).sum())                  # 实测 >5σ 天数
    e3 = n_total * 2 * (1 - stats.norm.cdf(3))       # 正态预测 >3σ
    e5 = n_total * 2 * (1 - stats.norm.cdf(5))       # 正态预测 >5σ
    print(f'{name:<10}{n3:>6}天{e3:>10.1f}天{n5:>6}天{e5:>12.4f}天')

# 各标的五年最大五次单日跌幅
for name in names:
    r = returns[name]
    worst = r.nsmallest(5)
    print(f'\n{name}:')
    for d_, v in worst.items():
        zv = (v - r.mean()) / r.std()
        print(f'  {d_.date()}: {v:+.2f}% ({zv:+.2f}σ)')
```

**样本输出（课程样本）：**

```text
标的        >3σ 实测   正态预测    >5σ 实测     正态预测
沪深三百        15天     3.3天        4天      0.0007天
标普五百        14天     3.4天        3天      0.0007天
纳斯达克        13天     3.4天        2天      0.0007天
比特币          15天     2.7天        1天      0.0006天

沪深三百 最大五次跌幅:
  2024-10-09: -7.05% (-6.38σ)
  2025-04-07: -7.05% (-6.37σ)
  2022-04-25: -4.94% (-4.47σ)
  ...
```

> [!WARNING]
> 标普五年里 >5σ 实测 **3 天**，正态预测 **0.0007 天**——实际比理论多几千倍。这不是统计噪声，是模型错了。

## 第五步：直方图 + 正态拟合 + Q-Q 图

```python
n = len(names)
fig, axes = plt.subplots(2, n, figsize=(4*n, 7))
for i, name in enumerate(names):
    r = returns[name]
    mu, sigma = float(r.mean()), float(r.std())
    # 上排：直方图 + 正态拟合曲线
    ax = axes[0, i]
    ax.hist(r, bins=80, density=True, alpha=0.55, color='#3aa0ff', label='实际')
    x = np.linspace(float(r.min()), float(r.max()), 500)
    ax.plot(x, stats.norm.pdf(x, mu, sigma), 'r-', lw=1.6, label='正态拟合')
    ax.axvline(mu - 3*sigma, ls='--', color='gray', lw=0.8)
    ax.axvline(mu + 3*sigma, ls='--', color='gray', lw=0.8)
    ax.set_title(f'{name} 日收益分布'); ax.legend(fontsize=8); ax.grid(alpha=0.3)
    # 下排：Q-Q 图
    ax2 = axes[1, i]
    stats.probplot(r, dist='norm', plot=ax2)
    ax2.set_title(f'{name} Q-Q 图')
    ax2.get_lines()[1].set_color('red')
    ax2.grid(alpha=0.3)
plt.tight_layout(); plt.savefig('day010_normal_qq.png', dpi=120)
```

**怎么读图：** 直方图上红色正态曲线在两侧尾巴处「够不着」实际数据；Q-Q 图两端翘起离开 45° 红线——厚尾实锤。

## 第六步：中心极限定理直觉实验

```python
ref_name = '沪深三百' if '沪深三百' in returns else names[0]
rr = returns[ref_name].values
print('=== 中心极限定理:聚合周期越长,越像正态(不重叠分块) ===')
for window in [1, 5, 21, 63]:
    n_blocks = len(rr) // window
    blocks = rr[:n_blocks * window].reshape(n_blocks, window).sum(axis=1)
    z = (blocks - blocks.mean()) / blocks.std()
    print(f'  {window:>2} 日聚合: 样本 {n_blocks:>4d} 偏度 {stats.skew(z):+.2f} 峰度 {stats.kurtosis(z):+.2f}')
```

**课程实测结论：** 沪深三百日收益峰度 ≈ 4.5–7，5 日聚合后掉到 ≈ 2，21 日聚合掉到 ≈ 0.7——CLT 真的在工作。**但散户的止损、保证金、开仓平仓全是日级的事，月度像正态救不了你。**
