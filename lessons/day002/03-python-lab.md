# ③ Python 实战：1000 次模拟看清两种风格

> 不争辩，直接模拟：让「主观投资者」和「量化策略」各活 1000 次、每次 20 年，看谁的结局更可控。

## 模拟设定

- **主观**：每年 5 个大决策。熟悉领域 60% 胜率（+6%），陌生领域 45% 胜率（−5%）；
- **量化**：每年 1000 次小决策，每次 51% 胜率，盈亏各 ±0.1%。

```python
# day_002_quant_vs_discretionary.py — 模拟两种风格的长期收益分布
import numpy as np
import matplotlib.pyplot as plt

np.random.seed(42)
N_YEARS = 20
N_TRIALS = 1000

# 主观投资：每年 5 个大决策，熟悉领域 60% 胜率，陌生领域 45% 胜率
def discretionary_year():
    rets = []
    for _ in range(5):
        familiar = np.random.rand() < 0.5
        win_p = 0.60 if familiar else 0.45
        win = np.random.rand() < win_p
        rets.append(0.06 if win else -0.05)   # +6% / -5% 每个决策
    return float(np.sum(rets))

# 量化策略：每年 1000 次小决策，每次 51% 胜率
def quant_year():
    trades = np.random.choice([0.001, -0.001], size=1000, p=[0.51, 0.49])
    return float(np.sum(trades))

# 跑 1000 次 20 年模拟
disc  = [(np.prod([1 + discretionary_year() for _ in range(N_YEARS)])) - 1 for _ in range(N_TRIALS)]
quant = [(np.prod([1 + quant_year()        for _ in range(N_YEARS)])) - 1 for _ in range(N_TRIALS)]

print(f'主观: 中位数 {np.median(disc)*100:.1f}%,  标准差 {np.std(disc)*100:.1f}%')
print(f'量化: 中位数 {np.median(quant)*100:.1f}%,  标准差 {np.std(quant)*100:.1f}%')
print(f'主观: 最差 5% 收益 {np.percentile(disc,5)*100:.1f}%')
print(f'量化: 最差 5% 收益 {np.percentile(quant,5)*100:.1f}%')

# 画分布对比
plt.figure(figsize=(10, 5))
plt.hist(np.clip(disc, -1, 5),  bins=50, alpha=0.55, label='主观投资 (5 决策/年)',  color='#b91c1c')
plt.hist(np.clip(quant, -1, 5), bins=50, alpha=0.55, label='量化策略 (1000 决策/年)', color='#15803d')
plt.axvline(0, color='#888', linestyle='--', alpha=0.5)
plt.legend(loc='upper right'); plt.title('20 年累计收益分布 · 1000 次模拟')
plt.xlabel('20 年累计收益(截断到 -100% ~ +500%)'); plt.ylabel('模拟次数')
plt.grid(alpha=0.3)
plt.savefig('day002_compare.png', dpi=120, bbox_inches='tight')
```

## 跑出来的结果（课程样本）

| 指标 | 主观投资 | 量化策略 |
| --- | --- | --- |
| 中位数（20 年累计） | **+82.9%** | +46.4% |
| 标准差 | **116.7%**（极度发散） | 20.7%（非常集中） |
| 最差 5% 情景 | **−22.8%**（20 年白干还亏钱） | +17.3%（最差也是赚） |

## 怎么读这个结果

1. **主观中位数高，但下行也凶**：每 20 个主观投资者里就有 1 个，20 年后还是亏的；
2. **量化收益低但波动小**：1000 次小决策的中心极限效应，让分布窄得多——这就是「大数定律」在赚钱上的样子；
3. **量化的真正优势是下行可控**：对管大钱的机构，「最差 5% 情景」比中位数更重要。

> [!TIP]
> 这个模拟就是 Day 002 的精华：**主观赌「深度正确」，量化赌「次数足够多」**。看懂这张分布图，你就懂了为什么机构愿意为量化付那么多管理费。
