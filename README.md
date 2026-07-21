# 量化金融 365 · 学习笔记

> 用大白话重述每一课，把「看似数学、实为地基」的知识点真正搞懂。

这套笔记配合《量化金融 365》课程使用，每天一课，共 365 课。笔记不是课件的复制粘贴，而是**听完课之后，用自己的话再讲一遍**——目标是：能给一个完全不懂的朋友讲清楚。

## 📖 笔记怎么用

1. **先听课，再看笔记**：笔记的作用是复习和查漏补缺，不是替代视频；
2. **每课一个章节**：左侧边栏按天排列，一课拆成「概述 → 概念 → 代码 → 案例 → 避坑 → 自测」六个小节；
3. **自测题最后做**：答不上来的，回到对应小节再看一遍；
4. **代码全部可跑**：复制到 Jupyter / VS Code 里直接运行，缺什么库就 `pip install` 什么。

## 📚 课程目录

**第一阶段 · 量化基础（P1）**

<a class="lesson-card" href="#/lessons/day001/README">
  <span class="lc-tag">DAY 001 / 365 · 已更新 ✅</span>
  <div class="lc-title">课程总览：365 天能学到什么</div>
  <div class="lc-desc">量化 = 数据+模型+工程 · 11 阶段路线图 · 每天 1% 的复利 · Hello Quant 第一段代码</div>
</a>

<a class="lesson-card" href="#/lessons/day002/README">
  <span class="lc-tag">DAY 002 / 365 · 已更新 ✅</span>
  <div class="lc-title">量化交易 vs 主观交易</div>
  <div class="lc-desc">巴菲特 vs Simons · 两种 alpha 的来源 · 1000 次蒙特卡洛模拟 · 混合派心法</div>
</a>

<a class="lesson-card" href="#/lessons/day003/README">
  <span class="lc-tag">DAY 003 / 365 · 已更新 ✅</span>
  <div class="lc-title">量化的三大流派</div>
  <div class="lc-desc">统计套利 / 高频做市 / 因子投资 · 散户选哪派 · 三派标志性代码</div>
</a>

<a class="lesson-card" href="#/lessons/day004/README">
  <span class="lc-tag">DAY 004 / 365 · 已更新 ✅</span>
  <div class="lc-title">金融市场基础速览</div>
  <div class="lc-desc">股票/债券/期货/期权/加密 · T+0 vs T+1 · 散户的真实通道与主战场选择</div>
</a>

<a class="lesson-card" href="#/lessons/day005/README">
  <span class="lc-tag">DAY 005 / 365 · 已更新 ✅</span>
  <div class="lc-title">收益率的几种算法</div>
  <div class="lc-desc">简单收益 vs 对数收益 · 年化的正确姿势 · 分红与复权 · 5 种算法的 Python 实战</div>
</a>

<a class="lesson-card" href="#/lessons/day006/README">
  <span class="lc-tag">DAY 006 / 365 · 已更新 ✅</span>
  <div class="lc-title">复利与贴现</div>
  <div class="lc-desc">复利公式与 72 法则 · NPV 与 IRR · 税/通胀/费率三大复利杀手 · 识破年金话术</div>
</a>

<a class="lesson-card" href="#/lessons/day007/README">
  <span class="lc-tag">DAY 007 / 365 · 已更新 ✅</span>
  <div class="lc-title">风险与方差</div>
  <div class="lc-desc">风险=波动？ · 年化波动率 · 下行风险与最大回撤 · 波动率聚集</div>
</a>

<a class="lesson-card" href="#/lessons/day008/README">
  <span class="lc-tag">DAY 008 / 365 · 已更新 ✅</span>
  <div class="lc-title">协方差与相关性</div>
  <div class="lc-desc">相关 ≠ 因果 · 分散化的数学原理 · 协方差矩阵 · Spearman 抗异常值</div>
</a>

<a class="lesson-card" href="#/lessons/day009/README">
  <span class="lc-tag">DAY 009 / 365 · 已更新 ✅</span>
  <div class="lc-title">线性回归与最小二乘</div>
  <div class="lc-desc">最小二乘法 · β 与 α 的含义 · 回归结果怎么读 · 回归的三个陷阱</div>
</a>

<a class="lesson-card" href="#/lessons/day010/README">
  <span class="lc-tag">DAY 010 / 365 · 已更新 ✅</span>
  <div class="lc-title">正态分布与金融</div>
  <div class="lc-desc">68-95-99.7 法则 · 偏度与峰度 · QQ 图 · 中段可信、尾部留一手</div>
</a>

<a class="lesson-card" href="#/lessons/day011/README">
  <span class="lc-tag">DAY 011 / 365 · 已更新 ✅</span>
  <div class="lc-title">厚尾与黑天鹅</div>
  <div class="lc-desc">肥尾分布 · 黑天鹅事件 · Kelly 公式 · 杠铃策略</div>
</a>

<a class="lesson-card" href="#/lessons/day012/README">
  <span class="lc-tag">DAY 012 / 365 · 已更新 ✅</span>
  <div class="lc-title">中心极限定理在金融</div>
  <div class="lc-desc">大数定律 · 抽样分布 · Bootstrap · 为什么小样本结论不可信</div>
</a>

<a class="lesson-card" href="#/lessons/day013/README">
  <span class="lc-tag">DAY 013 / 365 · 已更新 ✅</span>
  <div class="lc-title">假设检验入门</div>
  <div class="lc-desc">原假设与 p 值 · t 检验 · 多重检验陷阱 · 策略显著性判断</div>
</a>

<a class="lesson-card" href="#/lessons/day014/README">
  <span class="lc-tag">DAY 014 / 365 · 已更新 ✅</span>
  <div class="lc-title">R² 与策略评估</div>
  <div class="lc-desc">R² 的含义与误用 · 样本内 vs 样本外 · 过拟合识别</div>
</a>

<a class="lesson-card" href="#/lessons/day015/README">
  <span class="lc-tag">DAY 015 / 365 · 已更新 ✅</span>
  <div class="lc-title">时间序列与平稳性</div>
  <div class="lc-desc">平稳性 · ADF 检验 · 差分 · 协整与配对交易基础</div>
</a>

<a class="lesson-card" href="#/lessons/day016/README">
  <span class="lc-tag">DAY 016 / 365 · 已更新 ✅</span>
  <div class="lc-title">自相关与预测</div>
  <div class="lc-desc">ACF 与 Ljung-Box 检验 · 波动聚集 · Hurst 指数 · 收益不可预测但波动可预测</div>
</a>

<a class="lesson-card" href="#/">
  <span class="lc-tag">DAY 017 / 365 · 待更新 ⏳</span>
  <div class="lc-title">敬请期待</div>
  <div class="lc-desc">下一批：Day 017–032（预告）</div>
</a>

## 🧭 笔记的编写约定

每课笔记都遵循同样的结构，方便形成阅读习惯：

| 小节 | 内容 | 作用 |
| --- | --- | --- |
| 课程概述 | 这节课讲什么、为什么重要、学习目标 | 带着问题去听 |
| 核心概念 | 把每个概念用大白话 + 例子讲透 | 理解原理 |
| Python 实战 | 可直接运行的代码 + 逐段解释 | 动手验证 |
| 真实市场案例 | A股/港股/美股/加密的真实数字 | 建立直觉 |
| 避坑指南 | 常见错误 + 实战规则（SOP） | 少交学费 |
| 自测与延伸 | 自测题（附思路提示）+ 延伸阅读 | 检验吸收 |

> [!TIP]
> 页面上方有搜索框，忘记某个知识点时直接搜关键词（比如「复权」「√252」「协整」「IRR」）就能跳到对应位置。
