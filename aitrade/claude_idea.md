一、策略分类(Strategy Taxonomy)

  完整学术命名:

  ▎ Conditional Short-Horizon Mean-Reversion Strategy with Quality Screening, Regime Conditioning, and Scale-In Position Sizing
  ▎ (条件性短周期均值回归策略 — 含质量因子筛选、市场状态过滤与金字塔加仓)

  简称: Quality-Filtered Buy-the-Dip with Regime Overlay

  二、因子分解(Factor Decomposition)

```txt
  ┌────────────────┬─────────────────────────────────────────┬──────────────────────────────────────────────────────┐
  │      模块      │                学术因子                 │                       经典文献                       │
  ├────────────────┼─────────────────────────────────────────┼──────────────────────────────────────────────────────┤
  │ 52周均值下方   │ Long-term Reversal / Value              │ DeBondt & Thaler (1985)                              │
  ├────────────────┼─────────────────────────────────────────┼──────────────────────────────────────────────────────┤
  │ 公司经营稳定   │ Quality / Profitability Factor          │ Asness, Frazzini & Pedersen (2019);Fama-French 5因子 │
  ├────────────────┼─────────────────────────────────────────┼──────────────────────────────────────────────────────┤
  │ 日内 -3% 触发  │ Short-term Reversal                     │ Jegadeesh (1990);Lehmann (1990)                      │
  ├────────────────┼─────────────────────────────────────────┼──────────────────────────────────────────────────────┤
  │ 大盘趋势过滤   │ Time-Series Momentum / Regime Switching │ Moskowitz, Ooi & Pedersen (2012);Hamilton (1989)     │
  ├────────────────┼─────────────────────────────────────────┼──────────────────────────────────────────────────────┤
  │ T+1/T+2 持有   │ Sub-weekly Reversal Window              │ Lo & MacKinlay (1990)                                │
  ├────────────────┼─────────────────────────────────────────┼──────────────────────────────────────────────────────┤
  │ 1/3 → 2/3 加仓 │ Scale-In / Pyramid Averaging            │ 类 Martingale 但有上限                               │
  ├────────────────┼─────────────────────────────────────────┼──────────────────────────────────────────────────────┤
  │ +2% 止盈       │ Fixed Take-Profit / Asymmetric Exit     │ —                                                    │
  └────────────────┴─────────────────────────────────────────┴──────────────────────────────────────────────────────┘
```

  三、形式化数学表达(Formal Specification)

  信号函数:
  $$
  S_{i,t} = \mathbb{1}{P_{i,t} < \text{MA}{252}(P_i)} \cdot \mathbb{1}{r{i,t} < -3%} \cdot Q_i \cdot R_t
  $$

  其中:
  - $P_{i,t}$:个股 $i$ 在 $t$ 日收盘价
  - $r_{i,t} = P_{i,t}/P_{i,t-1} - 1$:日收益率
  - $Q_i \in {0,1}$:质量因子二值化筛选(基本面稳定 = 1)
  - $R_t \in {0,1}$:大盘状态指示函数(上涨/平稳 = 1)

  仓位演化(Position Path):
  $$
  w_{i,t} = \begin{cases}
  1/3 & \text{若 } S_{i,t}=1 \text{ 且 } w_{i,t-1}=0 \
  2/3 & \text{若 } S_{i,t}=1 \text{ 且 } w_{i,t-1}=1/3 \
  w_{i,t-1} & \text{其他情况}
  \end{cases}
  $$

  出场规则:
  $$
  \text{Exit at } \tau \in {t+1, t+2} \quad \text{iff} \quad \frac{P_{i,\tau}}{\bar{P}_{\text{cost}}} - 1 \geq 2%
  $$

  其中 $\bar{P}_{\text{cost}}$ 为加权平均成本。

  目标函数(隐含):
  最大化 $E[\text{Sharpe}]$ 等价于求解
  $$
  \max_{\theta} \frac{E[r_\text{strategy}]}{\sigma[r_\text{strategy}]} \quad \text{s.t. holding period} \leq 2 \text{ days}
  $$

  四、行业黑话翻译(Practitioner Vocabulary)

  如果向对冲基金或私募量化PM描述这个策略,标准说法是:

  ▎ "A long-only, single-name, intraday-triggered mean-reversion strategy with a fundamental quality overlay and a top-down beta regime filter, employing stepwise capital deployment (1/3, 2/3) on adverse moves and a fixed 2% take-profit with a
  ▎ 1-2 day max holding window."

  关键标签(用于策略归类/产品备案):
  - Style: Mean-Reversion / Contrarian
  - Horizon: Sub-weekly (T+1~T+2)
  - Universe: Single-name equity, fundamentally screened
  - Sizing: Discretionary scale-in (1/N pyramid)
  - Risk Profile: Asymmetric P&L (capped upside, uncapped downside) ← 专业人士会立刻指出这一点
  - Beta Profile: Long-only, low-to-medium market beta with regime gating

  五、和已知策略的同源关系

  该策略 ≈ (Bridgewater "Pure Alpha" 风格的 Quality + Value 过滤) × (Renaissance 短周期 reversal 信号) × (散户常见的"补仓") 的简化版本。

  最接近的标准化产品:
  - Quality-at-a-Reasonable-Price (QARP) + 短线择时
  - Drawdown Buying (Buying-the-Dip) with quality tilt
  - 类似于 AQR 早期论文 "Quality Minus Junk" 的多空版本中的纯多头切片

  六、专业评估的标准化指标(应当报告)

  如果以这个策略写策略报告或募资材料,必须披露:
  
```txt
  ┌────────────────────┬──────────────────────────────┐
  │        指标        │             说明             │
  ├────────────────────┼──────────────────────────────┤
  │ Sharpe Ratio       │ 年化风险调整收益             │
  ├────────────────────┼──────────────────────────────┤
  │ Sortino Ratio      │ 下行风险调整收益             │
  ├────────────────────┼──────────────────────────────┤
  │ Calmar Ratio       │ 收益/最大回撤                │
  ├────────────────────┼──────────────────────────────┤
  │ Max Drawdown       │ 最大回撤(无止损时通常很难看) │
  ├────────────────────┼──────────────────────────────┤
  │ Hit Rate           │ 胜率                         │
  ├────────────────────┼──────────────────────────────┤
  │ Profit Factor      │ 总盈利/总亏损                │
  ├────────────────────┼──────────────────────────────┤
  │ Avg Win / Avg Loss │ 盈亏比(此策略的死穴)         │
  ├────────────────────┼──────────────────────────────┤
  │ Turnover           │ 换手率(2日持有 → 极高)       │
  ├────────────────────┼──────────────────────────────┤
  │ Capacity           │ 资金容量(受流动性和滑点限制) │
  ├────────────────────┼──────────────────────────────┤
  │ Beta to CSI300     │ 与大盘相关性                 │
  ├────────────────────┼──────────────────────────────┤
  │ Tail Ratio         │ 95% 分位收益 / 5% 分位损失   │
  └────────────────────┴──────────────────────────────┘
```

