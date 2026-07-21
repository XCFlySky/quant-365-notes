# ③ Python 实战：5 种收益率算法

> 用 5 个真实资产（招商银行、美团、微软、黄金 ETF、比特币）把本课的 5 种算法全部跑一遍。代码复制到 Jupyter / VS Code 就能运行。

## 先记住一条实操原则

> [!TIP]
> **先跑通 → 再优化 → 最后追求精度。**
> 散户量化最大的坑，是代码还没跑通就开始优化参数。先让程序从头到尾跑完，哪怕写得很笨。

## 5 种算法速查表

| # | 算法 | 代码 | 用途 |
| --- | --- | --- | --- |
| 1 | 简单收益 | `df.pct_change()` | 展示给客户看 |
| 2 | 对数收益 | `np.log(df / df.shift(1))` | 建模、算夏普 |
| 3 | 累计收益（简单） | `(1 + r).cumprod() - 1` | 画净值曲线 |
| 4 | 累计收益（对数） | `log_r.cumsum()` | 画对数净值曲线 |
| 5 | 年化 | 收益 `(1+r)^N − 1`，波动 `σ × √N` | 跨周期比较 |

## 完整代码

```python
# day_005_returns_math.py — 招行/美团/微软/GLD/BTC 五资产收益率全套
import pandas as pd
import numpy as np
import yfinance as yf
import matplotlib.pyplot as plt

# 5 个资产：A股、港股、美股、黄金、比特币
tickers = {
    '招商银行 A': '600036.SS',
    '美团 H':     '3690.HK',
    'MSFT US':    'MSFT',
    '黄金 ETF':   'GLD',
    '比特币':     'BTC-USD',
}

# 下载近 3 年复权收盘价（auto_adjust=True 自动用复权价）
df = yf.download(list(tickers.values()), period='3y',
                 auto_adjust=True, progress=False)['Close']
df.columns = list(tickers.keys())
df = df.dropna()

# ---- 5 种收益率算法 ----
simple_ret = df.pct_change().dropna()          # ① 简单收益
log_ret    = np.log(df / df.shift(1)).dropna() # ② 对数收益
cum_simple = (1 + simple_ret).cumprod() - 1    # ③ 累计（简单）
cum_log    = log_ret.cumsum()                  # ④ 累计（对数）

# ⑤ 年化（注意：都是复利/平方根算法，不是简单乘法）
ann_ret = (1 + simple_ret.mean())**252 - 1     # 年化收益
ann_vol = simple_ret.std() * np.sqrt(252)      # 年化波动率
sharpe  = ann_ret / ann_vol                    # 简化夏普（未扣无风险利率）

result = pd.DataFrame({
    '3年累计简单(%)': (cum_simple.iloc[-1] * 100).round(1),
    '3年累计对数(%)': (cum_log.iloc[-1] * 100).round(1),
    '年化收益(%)':   (ann_ret * 100).round(1),
    '年化波动(%)':   (ann_vol * 100).round(1),
    'Sharpe':        sharpe.round(2),
})
print(result.to_string())

# ---- 陷阱演示：月均×12 vs 正确年化 ----
monthly   = simple_ret.resample('ME').apply(lambda x: (1 + x).prod() - 1)
wrong_ann = monthly.mean() * 12                # ❌ 错误示范
right_ann = (1 + monthly.mean())**12 - 1       # ✅ 正确做法
print('\n[陷阱演示] 月平均×12 vs 正确年化')
for name in monthly.columns:
    print(f'  {name}: 错误 {wrong_ann[name]*100:+.1f}% / 正确 {right_ann[name]*100:+.1f}%')

# ---- 画 3 年累计净值曲线 ----
(1 + simple_ret).cumprod().plot(figsize=(12, 5), title='3 年累计净值')
plt.tight_layout()
plt.savefig('day005_cumret.png', dpi=120)
```

## 逐段解释

**1. 数据下载**：`auto_adjust=True` 是关键——它让 yfinance 直接返回复权价，把分红再投资算进去。这是美股/港股数据最常见的错误来源。

**2. 两种日收益**：`pct_change()` 就是 (今天−昨天)/昨天；`np.log(df / df.shift(1))` 就是 ln(今天/昨天)。对比一下两者的数值，你会发现日级别几乎一样——印证了「短期两种收益差别很小」。

**3. 累计收益**：简单收益要**累乘**（`cumprod`），对数收益只要**累加**（`cumsum`）——这就是「对数收益可加」在代码里的体现。

**4. 年化**：收益用复利 `(1+r)^252 − 1`，波动用 `σ × √252`，正好对应上一节的两条规则。

**5. 陷阱演示**：把月收益分别用「×12」和「(1+r)^12 − 1」算年化，打印出来对比——你会亲眼看到差距有多大。

## 跑完看什么

- **对数 ≠ 简单**：时间越长、涨幅越大，「3年累计简单」和「3年累计对数」两列差得越多；
- **看 Sharpe 而不是看年化收益**：年化高收益 + 高波动，可能不如年化中等 + 低波动；
- **陷阱演示的输出**：把「错误 vs 正确」两个数字记在心里，以后看到别人的年化数据就有免疫力了。

## 偷懒神器：QuantStats

不想自己造轮子？开源库 QuantStats 一行搞定：

```python
import quantstats as qs
qs.reports.full(simple_ret['MSFT US'])   # 自动产出 30+ 指标 + 双视图
```

它内部全部用对数收益做计算、用简单收益做展示——正是这节课教的最佳实践。
