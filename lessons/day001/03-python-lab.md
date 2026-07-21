# ③ Python 实战：Hello Quant

> 用 30 行代码体验量化的完整闭环：数据 → 模型 → 评估。复制到 Jupyter / VS Code 直接能跑。

## 这段代码在干什么

做一个「最简动量策略」：在茅台、腾讯、苹果三只股票里，**每月买入过去 20 天涨得最多的那只**，对比「三只等权拿着不动」的基准，看谁赚得多。

```python
# day_001_hello_quant.py — 用 30 行代码体验量化的「数据→模型→评估」三件套
import pandas as pd
import numpy as np
import yfinance as yf
import matplotlib.pyplot as plt

# 1. 数据层：同时拉 A股/港股/美股 5 年日线（后复权 + 含分红）
tickers = {'600519.SS': '茅台', '0700.HK': '腾讯', 'AAPL': '苹果'}
df = yf.download(list(tickers), period='5y', auto_adjust=True)['Close']
df.columns = [tickers[c] for c in df.columns]
df = df.ffill().dropna(how='all')  # 跨市场交易日不齐，前向填充
df = df.dropna()

# 2. 模型层：计算月度动量（过去 20 日累计收益）
ret_20d = df.pct_change(20)
monthly_mom = ret_20d.resample('ME').last()

# 3. 评估层：每月持有动量最强的那只，记录下个月收益
next_month_ret = df.pct_change(20).shift(-20).resample('ME').last()
winners = monthly_mom.dropna().idxmax(axis=1)
rets = []
for date, sym in winners.items():
    if date in next_month_ret.index:
        rets.append(next_month_ret.loc[date, sym])
rets = pd.Series(rets, index=winners.index[:len(rets)]).dropna()

# 4. 对比基准（三只等权）
bench = next_month_ret.mean(axis=1).reindex(rets.index)
ann_strat = (1 + rets).prod() ** (12 / len(rets)) - 1
ann_bench = (1 + bench).prod() ** (12 / len(bench)) - 1
print(f'最简动量策略年化：{ann_strat:.2%}')
print(f'三市场等权基准年化：{ann_bench:.2%}')

# 5. 画累计净值对比图
(1 + rets).cumprod().plot(label='动量策略', color='#15803d', linewidth=2)
(1 + bench).cumprod().plot(label='等权基准', color='#88817a', linewidth=2, linestyle='--')
plt.title('Hello Quant: 5 年最简动量策略 vs 三市场等权基准')
plt.legend(); plt.grid(alpha=0.3)
plt.savefig('day001_hello.png', dpi=120, bbox_inches='tight')
```

## 逐段解释

**1. 数据层**：`auto_adjust=True` 用复权价；三个市场交易日不同步（A股休市美股开门），用 `ffill()` 前向填充对齐——这就是「数据层」要处理的典型脏活。

**2. 模型层**：`pct_change(20)` 是过去 20 个交易日的累计涨幅，规则一句话说完：谁涨得多买谁。

**3. 评估层**：`idxmax(axis=1)` 每月选出冠军股，`shift(-20)` 拿到「未来 20 天」的收益来检验这个月的选择对不对。

**4. 基准对比**：策略好不好不能自嗨，必须有个对照组——这里是「三只等权躺平」。

## 跑出来的真实结果（课程样本）

| 指标 | 动量策略 | 等权基准 |
| --- | --- | --- |
| 年化收益 | **+10.29%** | +2.96% |
| 5 年累计 | +63% | +16% |

但先别激动，三个必须泼的冷水：

> [!WARNING]
> 1. **没算费用**：实盘佣金、印花税、滑点会吃掉一部分超额，+10.29% 可能变成 +6~7%；
> 2. **样本太小**：3 只股票 5 年，统计上不显著，只是演示概念，真用要在 200+ 只股票池验证；
> 3. **回撤巨大**：2022–2024 期间策略最大跌幅接近 −40%——账户从 100 万跌到 60 万，普通人未必拿得住。

> [!TIP]
> 这就是量化的第一课：**代码 30 行就能跑，但「跑出来的数字」和「能赚的钱」之间，隔着费用、样本和心理三道关。**
