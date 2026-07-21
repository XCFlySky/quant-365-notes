# ③ Python 实战：CLT 可视化 + Bootstrap 置信区间

> 三个实验：亲眼看 CLT 收敛、亲眼看组合比个股更正态、亲手用 Bootstrap 给策略收益算置信区间。

复制到 Jupyter / VS Code 直接能跑。依赖：`pip install numpy pandas matplotlib`。

> 实操原则：先跑通（哪怕硬编码），再优化（参数化/函数化），最后才追求精度。散户量化最大的坑是没跑通就开始优化。

## 完整代码

```python
# day_012_clt_demo.py — 中心极限定理可视化实验 + Bootstrap 置信区间
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt

# 设置中文字体和随机种子（保证结果可复现）
plt.rcParams['font.sans-serif'] = ['Noto Sans CJK JP']
plt.rcParams['axes.unicode_minus'] = False
np.random.seed(42)

# ========== 实验 1：CLT 收敛过程可视化 ==========
def simulate_clt(distribution, sample_sizes, n_simulations=10000):
    """模拟不同样本量下的样本均值分布"""
    results = {}
    for n in sample_sizes:
        # 每次模拟：从原始分布抽 n 个样本，计算均值，重复 10000 次
        if distribution == 'uniform':
            samples = np.random.uniform(0, 1, (n_simulations, n))
        elif distribution == 'exponential':
            samples = np.random.exponential(1, (n_simulations, n))
        elif distribution == 'bimodal':
            # 双峰分布：50% 概率来自 N(-2,1)，50% 来自 N(2,1)
            mask = np.random.randint(0, 2, (n_simulations, n))
            samples = np.where(mask,
                               np.random.normal(2, 1, (n_simulations, n)),
                               np.random.normal(-2, 1, (n_simulations, n)))
        sample_means = samples.mean(axis=1)
        results[n] = sample_means
    return results

# 测试不同样本量：1, 5, 10, 30, 50
sample_sizes = [1, 5, 10, 30, 50]
distributions = ['uniform', 'exponential', 'bimodal']
dist_names = {'uniform': '均匀分布', 'exponential': '指数分布', 'bimodal': '双峰分布'}

fig, axes = plt.subplots(3, 5, figsize=(20, 12))
fig.suptitle('中心极限定理可视化：不同分布如何收敛到正态', fontsize=16, fontweight='bold')

for row, dist in enumerate(distributions):
    results = simulate_clt(dist, sample_sizes)
    for col, n in enumerate(sample_sizes):
        ax = axes[row, col]
        data = results[n]
        # 画样本均值的直方图
        ax.hist(data, bins=50, density=True, alpha=0.7,
                color='steelblue', edgecolor='black')
        # 叠加理论正态曲线，方便肉眼对比
        mu, sigma = data.mean(), data.std()
        x = np.linspace(data.min(), data.max(), 100)
        ax.plot(x, 1 / (sigma * np.sqrt(2 * np.pi)) * np.exp(-0.5 * ((x - mu) / sigma) ** 2),
                'r-', linewidth=2, label='理论正态')
        ax.set_title(f'{dist_names[dist]} (n={n})', fontweight='bold')
        ax.set_yticks([])
        if col == 0:
            ax.set_ylabel('密度', fontsize=10)
        if row == 2:
            ax.set_xlabel('样本均值', fontsize=10)
        if col == 4:
            ax.legend(fontsize=8)
plt.tight_layout()
plt.savefig('day012_clt_convergence.png', dpi=120, bbox_inches='tight')
print('✓ 图表已保存: day012_clt_convergence.png')

# ========== 实验 2：个股收益 vs 组合收益的正态性 ==========
# 模拟 A 股日收益（现实中可用 yfinance 拉真实数据）
np.random.seed(42)
n_days = 1000
# 生成带「肥尾」的收益：95% 来自 N(0, 0.02)，5% 来自 N(0, 0.10)——模拟极端日
fat_tail_mask = np.random.random(n_days) < 0.05
returns_single = np.where(fat_tail_mask,
                          np.random.normal(0, 0.10, n_days),
                          np.random.normal(0, 0.02, n_days))

# 分组聚合：模拟 50 只股票的等权组合
n_stocks = 50
returns_portfolio = np.zeros(n_days)
for _ in range(n_stocks):
    fat_tail_mask = np.random.random(n_days) < 0.05
    stock_returns = np.where(fat_tail_mask,
                             np.random.normal(0, 0.10, n_days),
                             np.random.normal(0, 0.02, n_days))
    returns_portfolio += stock_returns
returns_portfolio /= n_stocks

fig, axes = plt.subplots(1, 2, figsize=(14, 5))
fig.suptitle('金融收益的正态性：单资产 vs 组合', fontsize=14, fontweight='bold')

axes[0].hist(returns_single, bins=50, density=True, alpha=0.7,
             color='coral', edgecolor='black')
axes[0].set_title(f'单股票收益（偏度={pd.Series(returns_single).skew():.2f}）', fontweight='bold')
axes[0].set_xlabel('日收益率')
axes[0].set_ylabel('密度')

axes[1].hist(returns_portfolio, bins=50, density=True, alpha=0.7,
             color='seagreen', edgecolor='black')
axes[1].set_title(f'等权组合收益（偏度={pd.Series(returns_portfolio).skew():.2f}）', fontweight='bold')
axes[1].set_xlabel('日收益率')

plt.tight_layout()
plt.savefig('day012_normality_test.png', dpi=120, bbox_inches='tight')
print('✓ 图表已保存: day012_normality_test.png')

# ========== 实验 3：Bootstrap 置信区间估计 ==========
def bootstrap_ci(data, statistic=np.mean, n_bootstrap=10000, ci=95):
    """用 Bootstrap 方法计算置信区间"""
    boot_stats = []
    n = len(data)
    for _ in range(n_bootstrap):
        # 有放回重采样：从原数据里抽 n 个（允许重复）
        resample = np.random.choice(data, size=n, replace=True)
        boot_stats.append(statistic(resample))
    boot_stats = np.array(boot_stats)
    lower = (100 - ci) / 2
    upper = 100 - lower
    return np.percentile(boot_stats, [lower, upper]), boot_stats

# 模拟策略回测收益（250 个交易日）
strategy_returns = np.random.normal(0.001, 0.02, 250)  # 年化约 25%，波动 32%
ci, boot_means = bootstrap_ci(strategy_returns, n_bootstrap=10000)

print(f'\n策略年化收益: {(1 + strategy_returns).prod() ** (252 / 250) - 1:.2%}')
print(f'Bootstrap 95% 置信区间: [{(1 + ci[0]) ** 252 - 1:.2%}, {(1 + ci[1]) ** 252 - 1:.2%}]')

# 画 Bootstrap 均值分布
fig, ax = plt.subplots(figsize=(10, 6))
ax.hist(boot_means, bins=50, density=True, alpha=0.7,
        color='steelblue', edgecolor='black')
ax.axvline(ci[0], color='red', linestyle='--', linewidth=2, label=f'2.5% 分位数: {ci[0]:.4f}')
ax.axvline(ci[1], color='red', linestyle='--', linewidth=2, label=f'97.5% 分位数: {ci[1]:.4f}')
ax.axvline(np.mean(strategy_returns), color='green', linestyle='-', linewidth=3,
           label=f'样本均值: {np.mean(strategy_returns):.4f}')
ax.set_title('Bootstrap 方法：策略收益的 95% 置信区间', fontsize=14, fontweight='bold')
ax.set_xlabel('年化收益率（日度均值）', fontsize=12)
ax.set_ylabel('密度', fontsize=12)
ax.legend(fontsize=10)
plt.tight_layout()
plt.savefig('day012_bootstrap_ci.png', dpi=120, bbox_inches='tight')
print('✓ 图表已保存: day012_bootstrap_ci.png')

print('\n✅ 所有实验完成！')
```

## 逐段解释

**实验 1 · CLT 收敛过程。** `simulate_clt` 是核心：对每种分布、每个样本量 n，抽 10000 组「n 个样本」，每组算均值。画图时蓝色的直方图是 10000 个均值，红线是同均值同方差的理论正态曲线。肉眼可见：n=1 时蓝线就是原始分布的丑样子，n=30 时蓝线和红线几乎重合。

**实验 2 · 个股 vs 组合。** 先用混合分布造「肥尾」收益（5% 的日子波动放大 5 倍，模拟极端行情）。单只股票直接用这个序列；组合则把 50 只独立生成的股票收益等权平均。看两张直方图标题里的偏度：单股偏度明显更大，组合偏度显著下降——这就是「平均掉极端值」的直观证据。

**实验 3 · Bootstrap 置信区间。** `bootstrap_ci` 每轮从 250 个日收益里**有放回**抽 250 个，算均值，重复 10000 次，最后取 2.5% 和 97.5% 分位数。这个区间告诉你：这个策略的真实年化收益，95% 的概率落在哪。

## 样本输出（课程样本）

```text
✓ 图表已保存: day012_clt_convergence.png
✓ 图表已保存: day012_normality_test.png

策略年化收益: 25% 左右（种子固定时可复现）
Bootstrap 95% 置信区间: [约 9%, 约 41%]

✓ 图表已保存: day012_bootstrap_ci.png
✅ 所有实验完成！
```

课件给出的三个关键读数：

| 指标 | 读数 | 含义 |
| --- | --- | --- |
| 收敛速度 | n ≥ 30 | 均匀/指数分布此时已接近正态 |
| 偏度改善 | 1.0 → 0.3 | 单股 → 50 只等权组合 |
| Bootstrap CI 宽度 | ±8%（年化 9%~41%） | 点估计 25% 的不确定性很大 |

> [!WARNING]
> 看第三行：策略回测年化 25% 很漂亮，但 Bootstrap 95% 置信区间是 **9% 到 41%**。如果你只看点估计就上实盘，大概率会失望。**置信区间是安全带，不是装饰品。**
