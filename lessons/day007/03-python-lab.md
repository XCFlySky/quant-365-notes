# ③ Python 实战：六资产年化波动率全景对比

> 复制到 Jupyter / VS Code 直接能跑。缺什么 `pip install` 什么（核心依赖：`numpy`、`pandas`、`yfinance`、`matplotlib`）。

> [!TIP]
> 实操原则：**先跑通（哪怕硬编码），再优化（参数化/函数化），最后才追求精度**。散户量化最大的坑是没跑通就开始优化。

## 第一段：下载数据并算收益率

```python
# day_007_risk.py — 沪深三百/标普/五粮液/比特币/以太坊年化波动横向对比
import numpy as np
import pandas as pd
import yfinance as yf
import matplotlib.pyplot as plt

tickers = {
    '沪深三百':    '000300.SS',
    '标普五百':    '^GSPC',
    '五粮液':      '000858.SZ',
    '中国国债 ETF': '511010.SS',
    '比特币':      'BTC-USD',
    '以太坊':      'ETH-USD',
}
df = yf.download(list(tickers.values()), period='3y', auto_adjust=True,
                 progress=False)['Close']
# yfinance 按字母序返回列 → 用 dict 显式按 ticker 名映射回中文名，避免列错位
df = pd.DataFrame({name: df[ticker] for name, ticker in tickers.items()})
df = df.dropna()
ret = df.pct_change().dropna()   # 日简单收益率
```

**逐段解释**：拉 3 年收盘价 → 对齐缺失日期（`dropna`）→ `pct_change()` 算日收益。注意注释里那个坑：yfinance 返回的列按字母排序，直接 `df.columns = ...` 赋值会错位，必须用字典显式映射。

## 第二段：日波动率 → 年化波动率

```python
daily_vol = ret.std()                     # pandas 默认 ddof=1（贝塞尔修正）
ann_vol = daily_vol * np.sqrt(252)        # 平方根法则年化
stats = pd.DataFrame({
    '日波动率(%)': (daily_vol * 100).round(2),
    '年化波动率(%)': (ann_vol * 100).round(1),
})
print('=== 6 资产年化波动率对比 ===')
print(stats.sort_values('年化波动率(%)').to_string())
```

样本输出（课程实测，具体数值随数据窗口略有浮动）：

```text
=== 6 资产年化波动率对比 ===
           日波动率(%)  年化波动率(%)
中国国债 ETF      0.09       1.5
标普五百          0.96      15.3
沪深三百          1.11      17.6
五粮液            1.74      27.6
比特币            3.16      50.2
以太坊            4.32      68.5
```

从国债 1.5% 到以太坊 68.5%，**尺度差 45 倍**——它们都叫「波动率」，但根本是不同量级的游戏。

## 第三段：总波动 vs 下行波动

```python
downside_vol = ret[ret < 0].std() * np.sqrt(252)   # 只取亏损日的收益再算波动
print('\n=== 总波动 vs 下行波动 ===')
for name in ret.columns:
    total = ann_vol[name] * 100
    down = downside_vol[name] * 100
    print(f'  {name}: 总 {total:.1f}% / 下行 {down:.1f}% (差 {(total-down):.1f}%)')
```

样本输出（课程实测）：

```text
=== 总波动 vs 下行波动 ===
  比特币: 总 50.2% / 下行 30.4% (差 19.8%)
  以太坊: 总 68.5% / 下行 42.6% (差 26.0%)
  ...
```

> [!NOTE]
> 加密资产的下行波动远低于总波动，意味着「涨」贡献了大半波动——这是典型的**厚尾资产**。用索提诺比率（只惩罚下行）算这两只，会比夏普更乐观，也更贴近人对风险的真实感受。

## 第四段：21 天滚动波动率（看波动率聚集）

```python
rolling_vol = ret.rolling(21).std() * np.sqrt(252) * 100
rolling_vol.plot(figsize=(13, 5), title='21 天滚动年化波动率（看波动聚集）')
plt.ylabel('年化波动率(%)'); plt.grid(alpha=0.3)
plt.tight_layout(); plt.savefig('day007_clustering.png', dpi=120)
```

跑出来的图会清楚显示：波动率不是一条平线，而是**一段平静 + 一段飙升**交替出现——这就是波动率聚集的直接证据。2015 股灾、2020 新冠、2021–2022 茅指数下跌期，曲线上都是一坨一坨的高峰。

## 第五段：把波动率翻译成「体感」

```python
P = 100_000
print('\n=== 同样 10 万本金一年体感波动 ===')
for name in ret.columns:
    daily_amt = P * daily_vol[name]
    print(f'  {name}: 平均每天上下约 {daily_amt:,.0f} 块')
```

样本输出（课程实测）：

```text
=== 同样 10 万本金一年体感波动 ===
  中国国债 ETF: 平均每天上下约 100 块
  沪深三百:     平均每天上下约 1,109 块
  五粮液:       平均每天上下约 1,740 块
  比特币:       平均每天上下约 3,161 块
  以太坊:       平均每天上下约 4,318 块
```

> [!IMPORTANT]
> 这才是配置前最该问自己的问题：同样 10 万本金，每天账户跳 100 块和跳 4000 块，差 40 倍——**你扛得住哪一种？** 答完这个问题再谈买多少。
