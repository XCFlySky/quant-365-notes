# ③ Python 实战：三种贝塔人格一次算清楚

> 工行 / 隆基绿能 / 英伟达，真实数据跑回归，β、α、R²、残差全部出来。复制到 Jupyter / VS Code 直接能跑。

> [!TIP]
> 实操原则：**先跑通（哪怕硬编码），再优化（参数化/函数化），最后才追求精度**。散户量化最大的坑是没跑通就开始优化。

## 准备工作

```bash
pip install numpy pandas yfinance matplotlib scikit-learn
```

## 第一步：拉数据（注意 yfinance 的坑）

```python
# day_009_linear_regression.py — 工行/隆基/英伟达 三种贝塔人格
import numpy as np
import pandas as pd
import yfinance as yf
import matplotlib.pyplot as plt
from sklearn.linear_model import LinearRegression

tickers = {
    '工商银行': '601398.SS',
    '隆基绿能': '601012.SS',
    '英伟达':   'NVDA',
    '沪深三百': '000300.SS',
    '标普五百': '^GSPC',
}

df_raw = yf.download(list(tickers.values()), period='3y',
                     auto_adjust=True, progress=False)['Close']

# ⚠ 关键坑：不要直接 df.columns = list(tickers.keys())
# yfinance 经常按字母序返回列，强行改名会把数据张冠李戴
# 正确做法：用字典按 ticker 名显式映射
df = pd.DataFrame({name: df_raw[ticker] for name, ticker in tickers.items()})

# 中美分组各自 dropna：两边节假日不同，混在一起会把样本清掉
cn_df = df[['工商银行', '隆基绿能', '沪深三百']].dropna()
us_df = df[['英伟达', '标普五百']].dropna()

cn_ret = cn_df.pct_change().dropna()   # 日收益
us_ret = us_df.pct_change().dropna()
print(f'A 股样本: {len(cn_ret)} 天 / 美股样本: {len(us_ret)} 天')
```

**逐段解释：**

- `auto_adjust=True` 拿到的是复权价，避免除权除息日的假跳跃污染回归；
- 用字典 `{name: df_raw[ticker]}` 显式映射列名，是因为 yfinance 返回的列顺序不可控（常常按字母序），直接改 `columns` 是经典翻车姿势；
- A 股和美股交易日历不同（春节 vs 感恩节），必须分开 `dropna`，否则交集会把样本清得所剩无几。

## 第二步：封装一个回归函数

```python
def regress(stock_ret, market_ret, name, market_name):
    X = market_ret.values.reshape(-1, 1)   # 自变量：市场日收益
    y = stock_ret.values                   # 因变量：个股日收益
    model = LinearRegression().fit(X, y)
    beta = float(model.coef_[0])           # 斜率 = 贝塔
    alpha_daily = float(model.intercept_)  # 截距 = 日阿尔法
    alpha_annual = alpha_daily * 252       # 年化阿尔法
    # 手动算 R²：1 − 残差平方和/总平方和
    y_pred = model.predict(X)
    ss_res = ((y - y_pred) ** 2).sum()
    ss_tot = ((y - y.mean()) ** 2).sum()
    r2 = 1 - ss_res / ss_tot
    print(f'{name} vs {market_name}:')
    print(f'  贝塔 β = {beta:+.3f}')
    print(f'  阿尔法(日) α = {alpha_daily*100:+.4f}% / 年化 ≈ {alpha_annual*100:+.2f}%')
    print(f'  决定系数 R² = {r2:.3f}\n')
    return beta, alpha_daily, r2

print('=== A 股标的 vs 沪深三百 ===')
regress(cn_ret['工商银行'], cn_ret['沪深三百'], '工商银行', '沪深三百')
regress(cn_ret['隆基绿能'], cn_ret['沪深三百'], '隆基绿能', '沪深三百')
print('=== 美股 vs 标普五百 ===')
regress(us_ret['英伟达'], us_ret['标普五百'], '英伟达', '标普五百')
```

**样本输出（课程样本，数字随数据窗口变化）：**

```text
A 股样本: 742 天 / 美股样本: 770 天
=== A 股标的 vs 沪深三百 ===
工商银行 vs 沪深三百:
  贝塔 β = +0.182
  阿尔法(日) α = +0.1063% / 年化 ≈ +26.80%
  决定系数 R² = 0.023

隆基绿能 vs 沪深三百:
  贝塔 β = +1.436
  阿尔法(日) α = -0.1275% / 年化 ≈ -32.12%
  决定系数 R² = 0.392

=== 美股 vs 标普五百 ===
英伟达 vs 标普五百:
  贝塔 β = +2.118
  阿尔法(日) α = +0.1405% / 年化 ≈ +35.41%
  决定系数 R² = 0.420
```

> [!WARNING]
> 看到工行 β=0.18 先别怀疑代码——这是**真实数据揭穿教科书**的经典瞬间（详见 [④ 真实市场案例](lessons/day009/04-market-cases.md)）。教科书说银行 β 在 0.6–0.8，实测因为国家队维稳，工行跟沪深三百几乎脱钩（R² 只有 0.02）。

## 第三步：画散点图 + 回归线

```python
fig, axes = plt.subplots(1, 3, figsize=(15, 5))
pairs = [
    (cn_ret, '工商银行', '沪深三百', '低贝塔 · 银行稳健'),
    (cn_ret, '隆基绿能', '沪深三百', '中高贝塔 · 新能源放大镜'),
    (us_ret, '英伟达',   '标普五百', '高贝塔 · 美股 AI 加强版'),
]
for ax, (rr, s, m, label) in zip(axes, pairs):
    x = rr[m].values * 100   # 换成百分比较好读
    y = rr[s].values * 100
    ax.scatter(x, y, alpha=0.4, s=12)
    coef = np.polyfit(x, y, 1)              # 一次多项式拟合 = 线性回归
    xs = np.array([x.min(), x.max()])
    ax.plot(xs, np.polyval(coef, xs), 'r-', lw=2, label=f'β={coef[0]:.2f}')
    ax.axhline(0, color='gray', lw=0.5)
    ax.axvline(0, color='gray', lw=0.5)
    ax.set_xlabel(f'{m} 日收益(%)'); ax.set_ylabel(f'{s} 日收益(%)')
    ax.set_title(label)
    ax.grid(True, alpha=0.3); ax.legend()
plt.tight_layout()
plt.savefig('day009_regression.png', dpi=120)
```

**逐段解释：**

- `np.polyfit(x, y, 1)` 是最快的一行回归，和 `LinearRegression` 结果一致，画图时用它更顺手；
- 三张图并排放，一眼对比三种「贝塔人格」：工行的点云几乎水平（低 β 低 R²），隆基的线陡但点散（高 β 低 R²），英伟达又陡又相对集中。

## 第四步：看残差——跑赢还是跑输大市

```python
print('=== 工行最近 5 天残差(跑赢/输大市) ===')
X = cn_ret['沪深三百'].values.reshape(-1, 1)
y = cn_ret['工商银行'].values
model = LinearRegression().fit(X, y)
resid = y - model.predict(X)               # 残差 = 实际 − 预测
tail_dates = cn_ret.index[-5:]
for d, r in zip(tail_dates, resid[-5:]):
    direction = '跑赢' if r > 0 else '跑输'
    print(f'  {d.date()}: 残差 {r*100:+.3f}% ({direction} 大市 {abs(r)*100:.3f}%)')
```

**样本输出（课程样本）：**

```text
=== 工行最近 5 天残差(跑赢/输大市) ===
  2026-04-24: 残差 -0.042% (跑输 大市 0.042%)
  2026-04-27: 残差 -1.036% (跑输 大市 1.036%)
  2026-04-28: 残差 +0.209% (跑赢 大市 0.209%)
  2026-04-29: 残差 -1.103% (跑输 大市 1.103%)
  2026-04-30: 残差 -0.363% (跑输 大市 0.363%)
```

> [!NOTE]
> 单日残差正负交替很正常——噪声。**统计套利用的是「连续多天同号残差」**：连续 10 天正残差，才可能是真信号。这就是把残差从「解释工具」变成「交易信号」的关键一步。
