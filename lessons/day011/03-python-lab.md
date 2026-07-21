# ③ Python 实战：t 拟合 + 凯利仿真 + 杠铃压力测试

> 厚尾应对三件套，全部用真实数据 + 蒙特卡洛仿真跑一遍。复制到 Jupyter / VS Code 直接能跑。

> [!TIP]
> 实操原则：**先跑通（哪怕硬编码），再优化（参数化/函数化），最后才追求精度**。散户量化最大的坑是没跑通就开始优化。

## 准备工作

```bash
pip install numpy pandas yfinance matplotlib scipy
```

## 第一步：拉数据 + 学生 t 拟合

```python
# day_011_fat_tails.py — 厚尾应对:t 分布拟合 + 凯利公式 + 杠铃仿真
import numpy as np
import pandas as pd
import yfinance as yf
import matplotlib.pyplot as plt
from scipy import stats

rng = np.random.default_rng(42)   # 固定随机种子，结果可复现

# === 1) 取标普五百五年日收益(承接第十课的标的) ===
raw = yf.download('^GSPC', period='5y', auto_adjust=True, progress=False)
s = raw['Close']
if isinstance(s, pd.DataFrame):
    s = s.iloc[:, 0]
ret = (s.pct_change().dropna() * 100).values   # 百分比收益
n = len(ret)
print(f'标普五百:{n} 个交易日,均值 {ret.mean():.3f}%,标准差 {ret.std():.2f}%')
print(f'  实测峰度 {stats.kurtosis(ret):.2f}(正态 = 0)')

# === 2) 学生 t 分布拟合:估计自由度 ν ===
# 自由度越小 → 尾巴越厚;ν → ∞ 时 t 退化为正态
df_t, loc_t, scale_t = stats.t.fit(ret)
mu_n, sigma_n = ret.mean(), ret.std()
print('=== 正态 vs 学生 t 分布拟合 ===')
print(f'  正态:    均值 {mu_n:+.3f}, 标准差 {sigma_n:.3f}')
print(f'  学生 t:  自由度 ν = {df_t:.2f}, 位置 {loc_t:+.3f}, 规模 {scale_t:.3f}')
print(f'  → ν = {df_t:.1f} 意味着尾巴比正态厚得多(ν 越小越厚)')
```

**样本输出（课程样本）：**

```text
标普五百:1274 个交易日,均值 0.051%,标准差 1.06%
  实测峰度 7.16(正态 = 0)

=== 正态 vs 学生 t 分布拟合 ===
  正态:    均值 +0.051, 标准差 1.061
  学生 t:  自由度 ν = 3.94, 位置 +0.076, 规模 0.755
  → ν = 3.9 意味着尾巴比正态厚得多(ν 越小越厚)
```

**解释：** `stats.t.fit` 一行搞定拟合，返回自由度、位置、规模三个参数。ν ≈ 3.94 就是 Day 011 的标题数字——标普的真实尾巴要用 ν=4 的 t 分布才兜得住。

## 第二步：极端日——实测 vs 正态 vs t

```python
# === 3) 极端日:实测 vs 正态预测 vs t 预测 ===
print('=== 五倍标准差以上 — 三种估计的 PK ===')
print(f'{"阈值":<8}{"实测天数":>10}{"正态预测":>12}{"t 分布预测":>14}')
for k in [3, 4, 5, 6]:
    actual = int((np.abs((ret - mu_n) / sigma_n) > k).sum())
    p_norm = 2 * (1 - stats.norm.cdf(k))               # 正态双侧尾部概率
    p_t = 2 * stats.t.sf(k * sigma_n / scale_t, df=df_t)  # t 双侧尾部概率
    print(f'{k}σ {actual:>10}{n*p_norm:>12.3f}{n*p_t:>14.3f}')
```

**样本输出（课程样本）：**

```text
阈值      实测天数      正态预测      t 分布预测
3σ            14       3.440         17.836
4σ             5       0.081          6.586
5σ             3       0.001          2.925
6σ             1       0.000          1.482
```

> [!NOTE]
> 逐行看：t 分布每一行都贴着实测走，正态每一行都差几个数量级。这张表就是「为什么金融风控用 t 替代正态」的全部理由。

## 第三步：历史黑天鹅日历

```python
# === 4) 黑天鹅日历(过去三十年最有名的 8 次极端日) ===
events = [
    ('1987-10-19', '黑色星期一',  '标普 -20.5%',  '程序化交易踩踏'),
    ('1997-10-27', '亚洲金融危机', '道指 -7.2%',   '泰铢崩盘传染'),
    ('2008-10-15', '雷曼连锁',    '标普 -9.0%',   '次贷蔓延'),
    ('2010-05-06', '闪崩',        '道指日内 -9%', '大单乌龙 + HFT'),
    ('2015-08-24', 'A股股灾延烧', '道指 -3.6%',   '人民币贬值传染'),
    ('2018-02-05', 'VIX 爆雷',    '标普 -4.1%',   '波动率空头被强平'),
    ('2020-03-16', '新冠熔断',    '标普 -12.0%',  '疫情封控冲击'),
    ('2022-05-12', 'Luna 归零',   '比特币 -19%',  '算法稳定币崩塌'),
]
for d_, name, loss, why in events:
    print(f'  {d_}  {name:<10}  {loss:<12}  {why}')
```

**解释：** 近 30 年 8 次系统级黑天鹅，平均 3–4 年一次——完全不是教科书说的「几百年一遇」。这张表建议打印出来贴在屏幕上。

## 第四步：凯利公式仿真

```python
# === 5) 凯利公式仿真:全压 vs 凯利 vs 半凯利 vs 保守 ===
# 经典硬币赌博:赢概率 p=0.55,赢则 +1 倍本金,输则全亏
# 凯利最优仓位 f* = (p*b - q) / b,b=1 时 = 2p-1 = 0.10
print('=== 凯利公式仿真(1000 次抛硬币,胜率 55%)===')
n_trials, p_win = 1000, 0.55
f_kelly = 2 * p_win - 1          # = 0.10
strategies = {
    '全压 (f=1.0)':   1.0,
    '半压 (f=0.5)':   0.5,
    '凯利 (f=0.10)':  f_kelly,
    '半凯利 (f=0.05)': f_kelly / 2,
    '保守 (f=0.02)':  0.02,
}
for name, f in strategies.items():
    finals = []
    for _ in range(200):                    # 每个策略跑 200 条独立路径
        wealth = 1.0
        for _ in range(n_trials):
            won = rng.random() < p_win
            wealth = wealth * (1 + f) if won else wealth * (1 - f)
            if wealth < 1e-9: wealth = 0; break   # 破产
        finals.append(wealth)
    finals = np.array(finals)
    print(f'  {name:<14} 中位终值 {np.median(finals):>10.2f}'
          f'  均值 {finals.mean():>12.2f}  破产率 {(finals==0).mean()*100:>5.1f}%')
```

**样本输出（课程样本）：**

```text
全压 (f=1.0)     中位终值       0.00  均值         0.00  破产率 100.0%
半压 (f=0.5)     中位终值       0.00  均值         0.00  破产率 100.0%
凯利 (f=0.10)    中位终值     136.05  均值     10695.39  破产率   0.0%
半凯利 (f=0.05)  中位终值      63.62  均值       156.45  破产率   0.0%
保守 (f=0.02)    中位终值       6.56  均值         7.46  破产率   0.0%
```

> [!WARNING]
> 注意凯利均值（10695）远大于中位数（136）——少数极端幸运路径拉高了均值。**看中位数，别看均值**，这才是「大多数人会经历的结局」。

## 第五步：杠铃 vs 单一策略压力测试

```python
# === 6) 杠铃策略:90% 国债 + 10% 高赔率期权(模拟一年) ===
n_days, n_sim = 252, 5000
bond_daily = 0.0001                    # 年化约 2.5%
barbell_finals, single_finals = [], []
for _ in range(n_sim):
    # 杠铃:期权每天大概率亏,小概率(0.5%)赚 20 倍,散开到全年
    opt_outcomes = np.where(rng.random(n_days) < 0.005, 20.0, -1.0)
    bond_part = (1 + bond_daily) ** n_days
    opt_part = np.prod(1 + opt_outcomes / 252)
    barbell = 0.9 * bond_part + 0.1 * opt_part
    # 单一中等风险:每日收益 N(0.0003, 0.012)(年化约 7.5%, 波动 19%)
    daily_ret = rng.normal(0.0003, 0.012, n_days)
    single = np.prod(1 + daily_ret)
    barbell_finals.append(barbell); single_finals.append(single)

barbell_finals = np.array(barbell_finals)
single_finals = np.array(single_finals)
print(f'  杠铃策略:中位终值 {np.median(barbell_finals):.3f}'
      f'  均值 {barbell_finals.mean():.3f}'
      f'  最差 5% 分位 {np.percentile(barbell_finals,5):.3f}')
print(f'  单一策略:中位终值 {np.median(single_finals):.3f}'
      f'  均值 {single_finals.mean():.3f}'
      f'  最差 5% 分位 {np.percentile(single_finals,5):.3f}')
```

**样本输出（课程样本）：**

```text
杠铃策略:中位终值 0.963  均值 0.964  最差 5% 分位 0.960
单一策略:中位终值 1.065  均值 1.085  最差 5% 分位 0.778
```

**怎么读：** 单一策略中位数略高（1.065 vs 0.963），但最差 5% 分位是 0.778——一年亏 22%；杠铃最差 5% 分位 0.960，国债保底几乎不亏。**平时亏一点小钱，换黑天鹅日活着，这就是反脆弱的价格。**
