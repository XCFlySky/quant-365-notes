# ③ Python 实战：三派的「味道」

> 三段代码，分别给你统计套利、高频做市、因子投资的「最小可运行样本」。纯演示，感受三派各自的思维方式。

```python
# day_003_three_schools.py — 三派标志性代码对照
import numpy as np
import pandas as pd
import statsmodels.api as sm
import statsmodels.tsa.stattools as ts

np.random.seed(2026)

# === 统计套利派：配对交易的核心 —— 协整检验 ===
# 制造两只「本来应该一起走」的股票，价差围着均值波动
n = 500
common  = np.cumsum(np.random.randn(n) * 0.5)   # 共同趋势
stock_a = 50 + common + np.random.randn(n) * 0.6        # 模拟工行
stock_b = 30 + 0.6 * common + np.random.randn(n) * 0.6  # 模拟建行

# 用线性回归找「对冲比例」，再看价差是否平稳
model = sm.OLS(stock_a, sm.add_constant(stock_b)).fit()
hedge_ratio = float(model.params[1])
spread = stock_a - hedge_ratio * stock_b
_, p_value, *_ = ts.adfuller(spread)
print(f'[统计套利] 对冲比例 = {hedge_ratio:.3f}')
print(f'[统计套利] ADF 检验 p 值 = {p_value:.4f} -> ' + ('可做配对' if p_value < 0.05 else '不显著'))

# === 高频做市派：订单簿不平衡信号（伪代码）===
book = {'bid_size_1': 1200, 'ask_size_1': 800}
def order_flow_imbalance(b):
    return (b['bid_size_1'] - b['ask_size_1']) / (b['bid_size_1'] + b['ask_size_1'])
ofi = order_flow_imbalance(book)
print(f'[高频做市] OFI = {ofi:+.3f} -> ' + ('短期看涨' if ofi > 0.1 else '观望'))
print('[高频做市] 真实场景需要 FPGA + 同机房，散户无法参与')

# === 因子投资派：多因子等权打分选股 ===
factors = pd.DataFrame({
    'value':    np.random.randn(500),   # 价值：1/PE 的 rank
    'momentum': np.random.randn(500),   # 动量：过去 12-1 月收益
    'quality':  np.random.randn(500),   # 质量：ROE rank
    'low_vol':  np.random.randn(500),   # 低波：负的波动 rank
})
score = factors.mean(axis=1)            # 四因子等权合成
top_30 = score.nlargest(30).index.tolist()
print(f'[因子投资] 从 500 只里选出前 30 只，共 {len(top_30)} 只')
print(f'[因子投资] 头部得分均值 = {score.iloc[top_30].mean():+.3f}, '
      f'尾部 = {score.nsmallest(30).mean():+.3f}')
```

## 三段代码分别告诉你什么

**1. 统计套利 —— 关键是「价差平稳」**

两只同趋势的模拟股票（想象工行和建行），回归出对冲比例后算价差，用 ADF 检验判断价差是否平稳。课程样本结果：**p 值 ≈ 0.000 → 价差平稳，可做配对**。橡皮筋还在，暂时被拉远，过段时间会弹回来——这就是入场条件。

**2. 高频做市 —— 信号你也能算，但你跑不赢**

OFI（订单簿不平衡）= 买一档和卖一档挂单量的差值比，样本中 OFI = +0.20 → 买强看涨。公式两行就能写完，但要赚这个钱需要 **< 1 毫秒延迟和 FPGA 硬件**——信号到你电脑上时已经过期一万年了。

**3. 因子投资 —— 统计学的力量**

四个因子等权合成打分，500 只里选前 30。样本中头部均值 **+1.05** vs 尾部 **−1.00**，差距明显。不需要快，不需要贵，需要的是**纪律和样本量**。

> [!TIP]
> 三派一段代码就能尝出味道：统计套利赌「回归」，高频赌「速度」，因子赌「广度 × 时间」。问问自己：哪一个赌桌上，你有真实的筹码？
