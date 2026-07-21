# ③ Python 实战：R² 三视角

> 三个实验：给 4 只 A 股做「β + R²」画像；往模型里塞噪声看 Adj R² 如何打假；切训练/测试集看样本外 R² 如何暴露 β 漂移。

复制到 Jupyter / VS Code 直接能跑。依赖：`pip install numpy pandas yfinance scikit-learn matplotlib`。

> 实操原则：先跑通（哪怕硬编码），再优化（参数化/函数化），最后才追求精度。散户量化最大的坑是没跑通就开始优化。

## 完整代码

```python
# day_014_r2_strategy_eval.py — R² 在策略评估的三个真实场景
import numpy as np, pandas as pd, yfinance as yf
import matplotlib.pyplot as plt
from sklearn.linear_model import LinearRegression
from sklearn.metrics import r2_score

# ============ 1. 拉 4 只标的 + 沪深300 三年数据 ============
tickers = {
    '沪深300': '510300.SS',
    '平安银行': '000001.SZ',   # 高 β + 中 R²（跟大盘）
    '紫金矿业': '601899.SS',   # 高 β + 低 R²（商品周期）
    '长江电力': '600900.SS',   # 低 β + 低 R²（防御独立）
    '招商银行': '600036.SS',   # 比较组
}
raw = yf.download(list(tickers.values()), period='3y', auto_adjust=True)['Close']
# yfinance 按字母序返回列 → 用 dict 显式按名字映射，避免列错位
raw = pd.DataFrame({name: raw[ticker] for name, ticker in tickers.items()})
ret = raw.pct_change().dropna()
mkt = ret['沪深300']
print(f'数据样本量:{len(ret)} 个交易日')

# ============ 2. 三只股票对沪深300 单因子回归 — 算 β 和 R² ============
print('\n=== 三种股票画像 · 单因子回归 ===')
rows = []
for name in ['平安银行', '紫金矿业', '长江电力', '招商银行']:
    y = ret[name].values.reshape(-1, 1)
    X = mkt.values.reshape(-1, 1)
    model = LinearRegression().fit(X, y)
    beta = float(model.coef_[0][0])
    r2 = float(model.score(X, y))  # model.score 直接返回 R²
    pic = '跟随大盘' if r2 > 0.5 else ('半独立' if r2 > 0.25 else '基本独立')
    rows.append({'股票': name, 'β': round(beta, 3), 'R²': round(r2, 3), '画像': pic})
df1 = pd.DataFrame(rows)
print(df1.to_string(index=False))

# ============ 3. Adjusted R² 演示 — 加垃圾变量怎么暴露 ============
print('\n=== Adj R² 揭穿垃圾因子 ===')
np.random.seed(42)
n = len(ret)
y_t = ret['平安银行'].values
X1 = ret[['沪深300']].values                                # 1 个真因子
X2 = np.column_stack([X1, np.random.normal(0, 0.01, n)])    # +1 个噪声
X6 = np.column_stack([X1, np.random.normal(0, 0.01, (n, 5))])   # +5 个噪声
X21 = np.column_stack([X1, np.random.normal(0, 0.01, (n, 20))]) # +20 个噪声

def adj_r2(X, y):
    m = LinearRegression().fit(X, y)
    r2_v = m.score(X, y)
    n_, k = X.shape
    adj = 1 - (1 - r2_v) * (n_ - 1) / (n_ - k - 1)
    return r2_v, adj

results_adj = [adj_r2(X, y_t) for X in (X1, X2, X6, X21)]
df2 = pd.DataFrame({
    '模型': ['只用大盘 (1)', '+1 噪声 (2)', '+5 噪声 (6)', '+20 噪声 (21)'],
    'R²': [round(r, 4) for r, _ in results_adj],
    'Adj R²': [round(a, 4) for _, a in results_adj],
})
print(df2.to_string(index=False))
print('R² 一直涨 / Adj R² 加垃圾后停涨甚至降 → Adj R² 是诚实者')

# ============ 4. in-sample vs out-of-sample R² 过拟合检测 ============
print('\n=== in/out R² 过拟合检测 ===')
np.random.seed(7)
X_pool = np.column_stack([mkt.values, np.random.normal(0, 0.01, (n, 24))])
split = int(n * 0.7)   # 前 70% 训练，后 30% 测试
X_tr, X_te = X_pool[:split], X_pool[split:]
y_tr, y_te = y_t[:split], y_t[split:]

# 模型 A：只用大盘 1 个真因子
m_clean = LinearRegression().fit(X_tr[:, :1], y_tr)
r2_in_c = m_clean.score(X_tr[:, :1], y_tr)
r2_out_c = r2_score(y_te, m_clean.predict(X_te[:, :1]))

# 模型 B：大盘 + 24 个垃圾因子
m_over = LinearRegression().fit(X_tr, y_tr)
r2_in_o = m_over.score(X_tr, y_tr)
r2_out_o = r2_score(y_te, m_over.predict(X_te))

df3 = pd.DataFrame({
    '模型': ['A · 只用大盘', 'B · 加 24 垃圾'],
    'R²_in': [round(r2_in_c, 4), round(r2_in_o, 4)],
    'R²_out': [round(r2_out_c, 4), round(r2_out_o, 4)],
    '落差%': [
        round((1 - r2_out_c / r2_in_c) * 100, 1) if r2_in_c > 0 else 0,
        round((1 - r2_out_o / r2_in_o) * 100, 1) if r2_in_o > 0 else 0,
    ],
})
print(df3.to_string(index=False))
print('A 模型 R²_in/R²_out 接近 → 稳健;B 模型 R²_in 高 R²_out 暴跌 → 过拟合')

# ============ 5. 可视化 ============
fig, axes = plt.subplots(1, 3, figsize=(18, 5))

# 左：三只股票散点 + 拟合线，「棍子贴紧度」肉眼可见
colors = {'平安银行': 'tab:blue', '紫金矿业': 'tab:orange', '长江电力': 'tab:green'}
for name, color in colors.items():
    axes[0].scatter(mkt.values * 100, ret[name].values * 100,
                    alpha=0.3, color=color, s=10, label=name)
xline = np.linspace(mkt.min(), mkt.max(), 50)
for name, color in colors.items():
    m = LinearRegression().fit(mkt.values.reshape(-1, 1), ret[name].values)
    axes[0].plot(xline * 100, m.predict(xline.reshape(-1, 1)) * 100,
                 color=color, linewidth=2)
axes[0].set_xlabel('沪深300 日收益 (%)')
axes[0].set_ylabel('个股日收益 (%)')
axes[0].set_title('三种 β + R² 画像 · 棍子贴紧度肉眼可见')
axes[0].legend()
axes[0].grid(True, alpha=0.3)
axes[0].axhline(0, color='gray', alpha=0.5)
axes[0].axvline(0, color='gray', alpha=0.5)

# 中：加噪声后 R² vs Adj R² 走势
n_vars = [1, 2, 6, 21]
r2_vals = [r for r, _ in results_adj]
adj_vals = [a for _, a in results_adj]
axes[1].plot(n_vars, r2_vals, 'o-', color='tab:red', linewidth=2,
             markersize=10, label='普通 R²')
axes[1].plot(n_vars, adj_vals, 's-', color='tab:blue', linewidth=2,
             markersize=10, label='Adj R²')
axes[1].set_xlabel('总变量数(只有 1 个真因子)')
axes[1].set_ylabel('R²')
axes[1].set_title('Adj R² 揭穿垃圾 · R² 一直涨 Adj R² 拒绝跟涨')
axes[1].legend()
axes[1].grid(True, alpha=0.3)

# 右：训练/测试 R² 对比柱状图
labels = ['A · 只用大盘\n(简洁)', 'B · 加 24 噪声\n(过拟合)']
x = np.arange(2)
w = 0.35
axes[2].bar(x - w / 2, [r2_in_c, r2_in_o], w, label='R²_in (训练)', color='tab:blue')
axes[2].bar(x + w / 2, [r2_out_c, r2_out_o], w, label='R²_out (测试)', color='tab:red')
axes[2].set_xticks(x)
axes[2].set_xticklabels(labels)
axes[2].set_ylabel('R²')
axes[2].set_title('in vs out R² 过拟合检测 · 模型 B 落差暴露问题')
axes[2].legend()
axes[2].grid(True, alpha=0.3, axis='y')
axes[2].axhline(0, color='black', linewidth=0.5)

plt.tight_layout()
plt.savefig('day014_r2_eval.png', dpi=120)
print('\n✓ 图已保存到 day014_r2_eval.png')
```

## 逐段解释

**第 1~2 段 · 个股画像。** 拉三年日线算日收益，每只股票对沪深 300 做单因子回归。`LinearRegression().fit()` 的系数就是 β，`model.score(X, y)` 直接返回 R²。注意代码里专门用 dict 重排了列——yfinance 多只下载时按字母序返回，直接用位置索引会错位，这是真实项目里很常见的坑。

**第 3 段 · Adj R² 打假。** 只有沪深 300 一个真因子，其余 1/5/20 个变量全是纯噪声。普通 R² 数学上必然单调上涨，Adj R² 则对没有贡献的变量进行惩罚。`adj_r2` 函数就是课件公式 Adj R² = 1 − (1−R²)(n−1)/(n−k−1) 的直译。

**第 4 段 · 样本外检测。** 前 70% 数据训练、后 30% 测试。模型 A 只用真因子，模型 B 再塞 24 个噪声。注意样本外 R² 要用 `sklearn.metrics.r2_score(真实值, 预测值)` 显式算，不能用 `score()`（那是样本内的）。

**第 5 段 · 可视化。** 三张图分别对应三个实验：散点贴合度、R² vs Adj R² 走势、训练/测试 R² 对比。

## 样本输出（课程样本）

```text
=== 三种股票画像 · 单因子回归 ===
    股票      β     R²      画像
  平安银行  0.753  0.411  半独立
  紫金矿业  0.986  0.240  基本独立
  长江电力  0.123  0.022  基本独立
  招商银行  0.673  0.314  半独立

=== Adj R² 揭穿垃圾因子 ===
          模型      R²   Adj R²
   只用大盘 (1)  0.4105   0.4097
    +1 噪声 (2)  0.4108   0.4091
    +5 噪声 (6)  0.4210   0.4161
   +20 噪声(21)  0.4277   0.4105

=== in/out R² 过拟合检测 ===
          模型   R²_in   R²_out   落差%
  A · 只用大盘  0.5200  -0.1829  135.1
 B · 加 24 垃圾  0.5293  -0.1921  136.3
```

三个关键读数：

| 发现 | 数字 | 含义 |
| --- | --- | --- |
| 高 β 低 R² | 紫金 β=0.986 / R²=0.240 | β≈1 但大盘只解释 24%，β 会骗人 |
| Adj R² 诚实 | 加 20 噪声后 R²=0.4277 / Adj R²=0.4105 | R² 一路涨，Adj R² 回到基线不动 |
| 样本外负 R² | A: 0.52 → −0.18；B: 0.53 → −0.19 | 两模型一样烂 → 根源是 β 漂移，不是过拟合 |

> [!IMPORTANT]
> 最后一行是全课最反直觉的结论：**B 加 24 个垃圾后只比 A 差 0.0092**。如果糟糕的根源是过拟合，B 应该烂得多；两者几乎一样烂，说明是**市场结构变了（β 漂移）**——这连最简洁的模型也防不住，只能靠滚动重估和样本外监控来应对。
