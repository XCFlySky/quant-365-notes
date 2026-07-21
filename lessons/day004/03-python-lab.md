# ③ Python 实战：一段代码看 5 类资产

> 用 yfinance 把 A 股、港股、美股、黄金、比特币的 5 年走势拉到同一张图上，顺便复习 Day 005 的收益率算法。

```python
# day_004_asset_classes.py — 一段代码看 5 类资产 5 年走势
import pandas as pd
import yfinance as yf
import matplotlib.pyplot as plt

# 5 类资产代表：A股 / 港股 / 美股 / 黄金 / 加密
tickers = {
    'A股(沪深300)': '000300.SS',
    '港股(恒指)':   '^HSI',
    '美股(SP500)':  '^GSPC',
    '黄金':         'GC=F',
    '比特币':       'BTC-USD',
}

data = {}
for name, ticker in tickers.items():
    df = yf.download(ticker, period='5y', auto_adjust=True, progress=False)
    data[name] = df['Close']
df = pd.concat(data, axis=1).dropna()
norm = df / df.iloc[0]          # 标准化到起点 = 1

# 计算 5 年累计收益和年化波动率
ret_5y  = (norm.iloc[-1] - 1) * 100
vol_ann = df.pct_change().std() * (252 ** 0.5) * 100

result = pd.DataFrame({
    '5年累计收益(%)': ret_5y.round(1),
    '年化波动率(%)':  vol_ann.round(1),
    'Sharpe(粗估)':   (ret_5y / 5 / vol_ann).round(2),
})
print(result.to_string())

# 画对比图
norm.plot(figsize=(12, 5), title='5 年累计净值对比 (起点=1)')
plt.savefig('day004_assets.png', dpi=120, bbox_inches='tight')
```

## 跑出来的结果（课程样本）

| 资产 | 5 年累计 | 年化波动 | 粗估 Sharpe |
| --- | --- | --- | --- |
| 黄金 | **+154%** | 约 19% | **1.65** |
| 美股 SP500 | +72% | — | 0.83 |
| 比特币 | +35% | **59%** | 很低 |
| A 股沪深300 | 约 −5% | — | 负 |
| 港股恒指 | 约 −10% | — | 负 |

## 这张图告诉我们三件事

**1. 光看回报会被骗**

比特币 5 年 +35% 看似不差，但年化波动 59%——账户随时可能腰斩。黄金的 +154% 波动只有 19%。**同样的收益，波动小的那个「含金量」高得多**——这就是为什么必须看夏普而不是绝对回报。

**2. A 股/港股这五年不好过**

A 股 −5%、港股 −10%，美股 +72%——不是 A 股公司差，是宏观周期 + 流动性环境的差异。这告诉你**全球配置的重要性**。

**3. 黄金的避险价值**

美元贬值 + 地缘风险年代，黄金跑出年化 31%。这不是常态（过去 50 年均值约 7%），但说明：**合适的宏观环境下，避险资产也能跑赢权益**。

> [!TIP]
> 顺手复习：代码里 `pct_change().std() × √252` 就是 Day 005 的年化波动率公式——前后的课都是串着的。
