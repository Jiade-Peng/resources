从单股策略到多因子选股:完整升级路径

  这是收官实操篇。把前面九轮的所有理论落地为可执行的 12 个月路线图。

  ▎ 核心思维转变:
  ▎ - 旧思维: "我要找到一只好股票,反复操作它"
  ▎ - 新思维: "我要建立一个评分系统,让 4000 只股票每天给自己打分,我每周买入前 30 名,卖出跌出前 50 的"

  ---
  Phase 0:思维范式转换(第 0 月,1 周)

  1. 五个核心范式转换

```txt
  ┌────────────┬──────────────────────┬────────────────────────────┐
  │    维度    │       单股思维       │         多因子思维         │
  ├────────────┼──────────────────────┼────────────────────────────┤
  │ 关注对象   │ 1-10 只精选股票      │ 全市场 4000+ 只股票        │
  ├────────────┼──────────────────────┼────────────────────────────┤
  │ 决策方式   │ 主观判断"这只好不好" │ 客观打分"这只排第几"       │
  ├────────────┼──────────────────────┼────────────────────────────┤
  │ 持仓数量   │ 几只                 │ 几十只(30-100)             │
  ├────────────┼──────────────────────┼────────────────────────────┤
  │ 单股仓位   │ 1/3 - 67%            │ 1-3%                       │
  ├────────────┼──────────────────────┼────────────────────────────┤
  │ 再平衡     │ 事件驱动(跌了就买)   │ 时间驱动(每周一定时再平衡) │
  ├────────────┼──────────────────────┼────────────────────────────┤
  │ alpha 来源 │ "我看对了一只股"     │ "我的评分系统略胜随机 5%"  │
  └────────────┴──────────────────────┴────────────────────────────┘
```

  2. 心理准备

  你必须接受三个事实:

  1. 你将持有"看起来不喜欢"的股票 — 因为评分系统说它好,即使你直觉上不爱
  2. 你将卖出"看起来还能涨"的股票 — 因为它跌出了 top 名单
  3. 你将"放弃寻找完美时机" — 多因子选股不预测顶底,只赌"top 30 跑赢平均"

  ▎ 这是从"赌神"到"庄家"的转变。
  ▎ 庄家不知道下一手谁赢,但他知道长期他赢。

  3. 工具准备清单

```txt
  基础工具(必装):
  ├── Python 3.10+(开发语言)
  ├── Anaconda / Miniconda(环境管理)
  ├── Jupyter Notebook(交互式研究)
  ├── VSCode(代码编辑)

  数据源(选其一):
  ├── akshare(免费,推荐入门)
  ├── tushare Pro(¥200/年,数据更全)
  ├── 聚宽(策略平台,自带回测)

  核心 Python 包:
  ├── pandas, numpy(数据处理)
  ├── scipy, statsmodels(统计)
  ├── alphalens(因子分析,核心!)
  ├── vectorbt(回测,推荐)
  ├── matplotlib, seaborn(可视化)
```

  第一周任务: 装好环境,跑通一个 Hello World 级别的因子计算 — 比如计算所有 A 股的过去 20 日收益率,排序输出前 50 名。

  ---
  Phase 1:数据基础设施(第 1 月)

  1. 数据需求清单

  # 必备数据(以日频为基础)
```txt
  required_data = {
      # 行情数据
      'price_daily': ['open', 'high', 'low', 'close', 'volume', 'amount', 'turnover'],
      'price_adjusted': ['adj_close'],  # 复权价(分红送股)

      # 基本面数据
      'financial_quarterly': [
          'revenue', 'net_profit', 'total_asset', 'total_equity',
          'roe_ttm', 'roa_ttm', 'gross_margin', 'net_margin',
          'debt_to_equity', 'current_ratio', 'cash_ratio',
          'operating_cashflow', 'capex',
          'revenue_yoy', 'net_profit_yoy', 'eps_yoy'
      ],

      # 估值指标
      'valuation_daily': ['pe_ttm', 'pb', 'ps_ttm', 'pcf_ttm', 'ev_ebitda'],

      # 市场属性
      'classification': ['industry_sw', 'list_date', 'is_st', 'market_cap'],

      # 大盘数据
      'index_daily': ['000300.SH', '000905.SH', '000852.SH', '932000.CSI'],

      # 流通信息(用于风控)
      'shareholder': ['pledge_ratio', 'top_holder_change'],
  }
```

  2. 数据存储设计

  推荐 SQLite + Parquet 双栈:
  
```txt
  本地数据/
  ├── meta.db              # SQLite,存储元数据(行业、上市日)
  ├── price/
  │   ├── 2024.parquet    # 列存,按年分文件
  │   ├── 2025.parquet
  │   └── 2026.parquet
  ├── financial/
  │   └── ...
  └── factor/
      └── ...              # 因子计算结果缓存

```

  为什么这样设计?
  - Parquet:列存,按因子读取快 10 倍
  - 按年分文件:增量更新方便
  - SQLite:复杂查询(如行业筛选)用 SQL

  3. 数据更新流程

  # 每日运行(如:每日 17:00,收盘后)

```txt
  def daily_update():
      # 1. 增量拉取当日数据
      today_data = fetch_today_from_akshare()

      # 2. 数据质量检查
      check_data_quality(today_data)  # 缺失、异常、停牌

      # 3. 写入存储
      append_to_parquet(today_data)

      # 4. 计算衍生指标
      compute_returns()
      compute_market_cap()

      # 5. 触发因子重算
      schedule_factor_recompute()
```

  4. 数据陷阱预警

  A 股数据的典型坑:

```txt
  ┌─────────────┬──────────────────────────────────┬───────────────────────────┐
  │     坑      │               表现               │           解决            │
  ├─────────────┼──────────────────────────────────┼───────────────────────────┤
  │ 复权处理    │ 不复权数据中,送股日"假跌停"      │ 必须用后复权价计算收益    │
  ├─────────────┼──────────────────────────────────┼───────────────────────────┤
  │ 财报披露日  │ 用"报告期"而非"披露期"会未来函数 │ 必须用 公告日期(ann_date) │
  ├─────────────┼──────────────────────────────────┼───────────────────────────┤
  │ 停牌处理    │ 停牌期间价格不变,误以为"低波动"  │ 计算时剔除停牌日          │
  ├─────────────┼──────────────────────────────────┼───────────────────────────┤
  │ 退市股      │ 缺失会导致生存偏差               │ 必须用 包含退市的全样本   │
  ├─────────────┼──────────────────────────────────┼───────────────────────────┤
  │ ST/*ST 切换 │ 涨跌幅限制变化                   │ 单独标记,可能需要排除     │
  ├─────────────┼──────────────────────────────────┼───────────────────────────┤
  │ 分红再投资  │ 不处理会低估收益                 │ 用全收益指数(CSIH00300)   │
  └─────────────┴──────────────────────────────────┴───────────────────────────┘
```

  第 1 月任务:
  - ✅ 跑通数据下载脚本
  - ✅ 完成 2015-2026 全样本数据本地化
  - ✅ 写一个数据质量检查工具

  ---
  Phase 2:单因子研究(第 2-3 月)

  1. 因子设计的"7 大类目"

  基于前面讨论,A 股有效因子主要分为 7 类:

  # 因子分类与示例

```txt
  factor_taxonomy = {
      # 1. 价值类(EP、BP、SP、CFP)
      'value': {
          'ep': 'net_profit_ttm / market_cap',
          'bp': 'total_equity / market_cap',
          'sp': 'revenue_ttm / market_cap',
          'cfp': 'operating_cashflow_ttm / market_cap',
      },

      # 2. 质量类(ROE、毛利率、稳定性)
      'quality': {
          'roe_ttm': 'net_profit_ttm / avg_equity',
          'gross_margin_5y_avg': '5 年平均毛利率',
          'roe_stability': '5 年 ROE 标准差(取负)',
          'accrual': '应计利润占比(造假预警)',
      },

      # 3. 成长类(增速)
      'growth': {
          'revenue_yoy': '营收同比',
          'net_profit_yoy': '净利润同比',
          'roe_growth': 'ROE 改善幅度',
      },

      # 4. 反转/动量类
      'momentum_reversal': {
          'reversal_5d': '过去 5 日累计收益(取负 → 反转)',
          'reversal_20d': '过去 20 日累计收益(取负)',
          'momentum_60_5': '过去 60 日 - 过去 5 日(经典动量)',
      },

      # 5. 波动率类(低波因子)
      'volatility': {
          'vol_60d': '60 日收益标准差(取负)',
          'idiosyncratic_vol': '残差波动率',
          'beta_60d': 'Beta 系数(可能取负)',
      },

      # 6. 流动性类
      'liquidity': {
          'turnover_20d': '20 日平均换手率(可能取负)',
          'amihud': 'Amihud 非流动性',
          'volume_zscore_5d': '近 5 日成交量异动',
      },

      # 7. 情绪/资金流类
      'sentiment_flow': {
          'north_capital_flow': '北向资金净流入',
          'large_order_imbalance': '大单净额',
          'analyst_upgrade': '卖方评级上调数',
      }
  }
```

  2. 单因子研究的标准流程

  对每个候选因子,跑完整的"因子检验五步法":

  Step 1:因子值计算

```txt
  # 例:计算 EP 因子
  def compute_ep(date):
      df = get_panel_data(date)
      df['ep'] = df['net_profit_ttm'] / df['market_cap']

      # 必做:去极值 + 标准化
      df['ep_winsorized'] = winsorize(df['ep'], 0.01, 0.99)
      df['ep_zscore'] = (df['ep_winsorized'] - df['ep_winsorized'].mean()) / df['ep_winsorized'].std()

      # 必做:行业中性化(防止行业暴露)
      df['ep_neutral'] = neutralize_by_industry(df['ep_zscore'], df['industry'])

      return df['ep_neutral']
```

  Step 2:IC 时序检验

```txt
  import alphalens as al

  # alphalens 一键生成因子分析报告
  factor_data = al.utils.get_clean_factor_and_forward_returns(
      factor=ep_factor,
      prices=close_prices,
      quantiles=10,           # 分 10 组
      periods=(1, 5, 10, 20)  # 1日、5日、10日、20日预测
  )

  al.tears.create_full_tear_sheet(factor_data)
  # 自动输出:IC 时序、分组收益、衰减曲线、行业暴露
```

  Step 3:分组收益

```txt
  理想形态:
  组 1(因子值最低)收益 ━━━━━━━━━━━━━━ -3% 年化
  组 2 ━━━━━━━━━━━━━━━━━━━━━━━━━━━ -1%
  组 3 ━━━━━━━━━━━━━━━━━━━━━━━━━━━ 0%
  ...
  组 10(因子值最高)━━━━━━━━━━━━━━ +12% 年化

  多空收益:组 10 - 组 1 = +15% 年化
  单调性:从组 1 到组 10 大体单调上升
```

  Step 4:稳健性检验

  - 不同时间段(2015-2018、2018-2022、2022-2026)分别跑
  - 不同股票池(沪深 300、中证 500、中证 1000)分别跑
  - 不同市值组(大、中、小)分别跑
  - 不同行业(金融、消费、TMT、周期)分别跑

  → 如果只在某一段时间或某一个股票池有效,这个因子风险大

  Step 5:过拟合检验

  - 计算 PSR、Deflated Sharpe
  - Walk-Forward 验证
  - 与已知因子(EP、ROE、动量)的相关性

  3. 因子研究的"优先级排序"

  新手起步,建议按这个顺序研究:

```txt
  Tier 1 (必做,基础):
  ├── EP(盈利价格比)  ← Liu, Stambaugh, Yuan (2019) 核心
  ├── ROE_TTM
  ├── 反转 5D / 反转 20D
  ├── 60日波动率(取负)
  └── 60日换手率(取负)

  Tier 2 (推荐,补充):
  ├── 应计利润(造假预警)
  ├── 营收增速
  ├── 北向资金 5 日累计净流入
  ├── 残差波动率
  └── 大单净额 / 总成交额

  Tier 3 (高阶,选做):
  ├── 分析师一致预期变化
  ├── 财报披露效应
  ├── 行业 momentum
  └── 另类数据因子(搜索热度等)
```

  4. 阶段性产出

  第 2-3 月结束时,你应该有:
  - ✅ 10-15 个已检验单因子
  - ✅ 每个因子的 完整 alphalens 报告
  - ✅ 因子相关性矩阵
  - ✅ "因子库"文档(每个因子的逻辑、表现、风险)

  ---
  Phase 3:多因子合成(第 4-5 月)

  1. 为什么需要"合成",不是"叠加"?

  叠加(你之前的思路):

  ▎ "EP 高 AND ROE 高 AND 跌 -3% AND 大盘上涨"

  问题:
  - 同时满足所有条件的股票极少
  - 错过大量"差一点"的好股票
  - 任一条件略变就剧烈影响选股池

  合成(专业思路):

  ▎ 给每个股票算一个综合分,取分数最高的 30 只

  优势:
  - 永远有 30 只股票可选
  - 单一因子失效不致命
  - 可控制因子权重

  2. 合成方法对比

  方法 1:Z-Score 加权(最简单,推荐入门)

```txt
  def composite_score_zscore(df, weights):
      """
      简单 Z-Score 加权合成
      """
      score = pd.Series(0, index=df.index)
      for factor, weight in weights.items():
          score += weight * df[factor + '_zscore']
      return score
```

  # 使用

```txt
  weights = {
      'ep': 0.20,
      'roe_ttm': 0.20,
      'reversal_20d': 0.15,
      'vol_60d_neg': 0.15,  # 取负的波动率
      'turnover_neg': 0.10,
      'profit_growth': 0.10,
      'accrual_neg': 0.10,  # 取负的应计利润
  }
```

```txt
  scores = composite_score_zscore(df_today, weights)
  top30 = scores.nlargest(30)
```

  方法 2:IC 加权(进阶)

```txt
  def composite_score_ic_weighted(df, factor_ics):
      """
      用每个因子的历史 IC 作为权重
      """
      score = pd.Series(0, index=df.index)
      total_weight = 0
      for factor, ic in factor_ics.items():
          if abs(ic) > 0.02:  # 只用有效因子
              weight = ic / np.std(ic_series[factor])  # IC / IC 标准差
              score += weight * df[factor + '_zscore']
              total_weight += abs(weight)
      return score / total_weight
```

  方法 3:回归合成(机构主流)

```txt
  def composite_score_regression(df, lookback_period=12):
      """
      用过去 12 个月的截面回归求最优权重
      """
      # 历史数据回归:next_month_return = α + β1·factor1 + β2·factor2 + ...
      X = df[factor_columns].values  # 因子值矩阵
      y = df['next_month_return'].values

      # Ridge 回归(防止过拟合)
      model = Ridge(alpha=1.0)
      model.fit(X, y)

      # 用拟合的系数作为权重
      weights = model.coef_
      return df[factor_columns] @ weights
```

  方法 4:机器学习(顶级量化)

```txt
  def composite_score_ml(df):
      """
      用 LightGBM/XGBoost 拟合非线性合成
      """
      # 训练:用过去 5 年的因子值预测下月收益
      model = lgb.LGBMRegressor(
          num_leaves=31,
          learning_rate=0.05,
          n_estimators=100,
          min_child_samples=100,  # 防止过拟合
          reg_alpha=0.1,
          reg_lambda=0.1,
      )
      model.fit(X_train, y_train)

      # 预测当期收益
      return model.predict(df[factor_columns])
```

  选择建议:
  - 第一年:用方法 1(Z-Score 加权),简单可靠
  - 第二年:升级到方法 2 或 3
  - 方法 4 慎用 — 机器学习极容易过拟合,需要专业经验

  3. 因子权重的"理性配置"

  不要平均分配,根据因子特性配比:

```txt
  ┌──────────────────────┬──────────┬─────────────────────────┐
  │       因子类别       │ 建议权重 │          原因           │
  ├──────────────────────┼──────────┼─────────────────────────┤
  │ 价值因子(EP)         │ 20-25%   │ A 股最稳定的 alpha 来源 │
  ├──────────────────────┼──────────┼─────────────────────────┤
  │ 质量因子(ROE+稳定性) │ 20-25%   │ 防雷,长期有效           │
  ├──────────────────────┼──────────┼─────────────────────────┤
  │ 短反转               │ 10-15%   │ alpha 强但衰减快        │
  ├──────────────────────┼──────────┼─────────────────────────┤
  │ 低波动               │ 10-15%   │ 风险调整收益贡献        │
  ├──────────────────────┼──────────┼─────────────────────────┤
  │ 流动性               │ 5-10%    │ 辅助筛选                │
  ├──────────────────────┼──────────┼─────────────────────────┤
  │ 成长                 │ 10-15%   │ 周期性强,需谨慎         │
  ├──────────────────────┼──────────┼─────────────────────────┤
  │ 情绪/资金流          │ 5-10%    │ 高频信号补充            │
  └──────────────────────┴──────────┴─────────────────────────┘
```

  4. 关键的"因子中性化"

  这是新手最容易忽略的步骤,但极其重要:

```txt
  def neutralize_factor(factor, market_cap, industry):
      """
      对因子做市值 + 行业中性化
      """
      # 1. 行业 dummy
      industry_dummies = pd.get_dummies(industry)

      # 2. 加上市值(对数化)
      log_mcap = np.log(market_cap)

      # 3. 回归取残差
      X = np.column_stack([log_mcap.values, industry_dummies.values])
      y = factor.values

      model = LinearRegression()
      model.fit(X, y)

      # 残差 = 因子中性化后的纯 alpha
      return y - model.predict(X)
```

  为什么必须做?
  - 如果不做,你的"价值因子"实际上**=价值 + 小盘暴露**
  - 当小盘崩盘(如 2024.2),你的"价值"信号也崩
  - 中性化让信号变成纯 alpha,与风格无关

  5. 阶段性产出

  第 4-5 月结束时,你应该有:
  - ✅ 一个多因子合成评分系统
  - ✅ 每日可计算全市场 5000 只股票的综合分
  - ✅ Top 30 / Top 50 / Top 100 的历史表现回测
  - ✅ 因子权重的敏感性分析

  ---
  Phase 4:组合构建与回测(第 6 月)

  1. 组合构建的核心问题

  评分系统给出 Top 50 股票后,关键问题:
  1. 持有多少只?(集中度 vs 分散度)
  2. 每只多少仓位?(等权 vs 评分加权)
  3. 多久换一次?(再平衡频率)
  4. 行业/风格是否要约束?

  2. 经典组合构建方案

  方案 A:等权重,周度再平衡(推荐入门)

```txt
  def build_portfolio_equal_weight(scores, n_stocks=30, total_position=0.6):
      """
      最简单方案:等权重持有 Top 30
      """
      top_stocks = scores.nlargest(n_stocks).index
      weights = pd.Series(total_position / n_stocks, index=top_stocks)
      return weights
```

  # 每周一开盘前再计算
  # 更换 = 卖出已经不在 Top 30 的,买入新进入的

  特点:
  - ✅ 简单,无过拟合风险
  - ✅ 学术研究证明长期不输于复杂方案
  - ❌ 不能精确控制风险

  方案 B:风险预算 + 行业约束(进阶)

```txt
  import cvxpy as cp

  def build_portfolio_optimized(scores, cov_matrix, constraints):
      """
      用 Markowitz 优化求解
      """
      n = len(scores)
      w = cp.Variable(n)

      # 目标:最大化 评分 - λ·风险
      objective = cp.Maximize(scores.values @ w - 5 * cp.quad_form(w, cov_matrix))

      # 约束
      constraints_list = [
          cp.sum(w) == constraints['total_position'],  # 总仓位
          w >= 0,                                       # 不允许做空
          w <= constraints['max_single'],               # 单股上限
          # 行业约束
          # 风格约束
      ]

      problem = cp.Problem(objective, constraints_list)
      problem.solve()
      return w.value
```

  方案 C:基准约束(指数增强思路)

```txt
  def build_portfolio_index_enhanced(scores, benchmark_weights, tracking_error_target=0.04):
      """
      指数增强:跟踪沪深 300 / 中证 500,但权重略偏向高分股
      """
      # 在基准权重基础上,根据评分微调
      active_weights = scores.rank(pct=True) - 0.5  # 排名转换为 [-0.5, +0.5]
      final_weights = benchmark_weights + active_weights * 0.05  # 偏离 5%

      # 强制非负
      final_weights = final_weights.clip(lower=0)
      final_weights = final_weights / final_weights.sum()  # 归一化

      return final_weights
```

  3. 再平衡频率选择

```txt
  ┌──────┬───────────────────┬─────────────────┬────────────────┐
  │ 频率 │      换手率       │    摩擦成本     │      适合      │
  ├──────┼───────────────────┼─────────────────┼────────────────┤
  │ 每日 │ 极高(年化 5000%+) │ 极高(年化 -25%) │ 高频量化(机构) │
  ├──────┼───────────────────┼─────────────────┼────────────────┤
  │ 每周 │ 高(年化 1500%)    │ 中(年化 -8%)    │ 个人推荐       │
  ├──────┼───────────────────┼─────────────────┼────────────────┤
  │ 每月 │ 中(年化 500%)     │ 低(年化 -3%)    │ 多因子主流     │
  ├──────┼───────────────────┼─────────────────┼────────────────┤
  │ 每季 │ 低(年化 200%)     │ 极低(年化 -1%)  │ 价值投资       │
  └──────┴───────────────────┴─────────────────┴────────────────┘
```

  核心权衡:
  - 频率高 → 信号新鲜 → 但摩擦成本高
  - 频率低 → 信号过期 → 但摩擦成本低
  - 最优点通常在每周/每两周

  4. 完整回测代码框架

```txt
  import vectorbt as vbt

  def backtest_multi_factor_strategy(
      start_date='2018-01-01',
      end_date='2025-12-31',
      initial_capital=1_000_000,
      n_stocks=30,
      rebalance_freq='W-MON',  # 每周一
      transaction_cost=0.0015,  # 双边 0.15%
  ):
      # 1. 计算每日评分
      daily_scores = compute_composite_scores(start_date, end_date)

      # 2. 在再平衡日生成目标持仓
      rebalance_dates = pd.date_range(start_date, end_date, freq=rebalance_freq)
      target_holdings = {}
      for date in rebalance_dates:
          target_holdings[date] = daily_scores.loc[date].nlargest(n_stocks).index

      # 3. 模拟交易
      portfolio = vbt.Portfolio.from_signals(
          close=close_prices,
          entries=generate_entries(target_holdings),
          exits=generate_exits(target_holdings),
          fees=transaction_cost / 2,  # 单边
          slippage=0.001,
          freq='1D',
          init_cash=initial_capital,
      )

      # 4. 输出报告
      print(portfolio.stats())
      portfolio.plot().show()

      return portfolio
```

  5. 第一次完整回测的"健康检查"

  回测出来的策略,关注以下指标:

```txt
  expected_metrics = {
      # 收益类
      'annual_return': '> 12%',           # 年化收益
      'excess_return_vs_csi300': '> 5%',  # 超额收益

      # 风险类
      'max_drawdown': '< 25%',            # 最大回撤
      'volatility': '15% - 25%',          # 年化波动率

      # 风险调整
      'sharpe_ratio': '> 1.0',            # 夏普
      'calmar_ratio': '> 0.5',            # 收益/回撤
      'sortino_ratio': '> 1.2',           # 下行风险调整

      # 稳定性
      'win_rate_monthly': '> 55%',        # 月度胜率
      'consecutive_loss_max': '< 4',       # 最长连亏月数

      # 实务
      'turnover_annual': '< 1500%',       # 换手率(高了不可行)
      'avg_holding_days': '> 5',          # 平均持有期
      'capacity': '> 1000万',              # 资金容量
  }
```

  警报:
  - Sharpe > 3.0 → 几乎肯定过拟合
  - 最大回撤 < 5% → 可疑
  - 全部年份盈利 → 可疑
  - 单年贡献 > 50% 总收益 → 不稳定

  ---
  Phase 5:风险管理(第 7 月)

  1. 三层风险控制体系

  第一层:个股风险

```txt
  single_stock_rules = {
      'max_weight': 0.03,              # 单股 ≤ 3%
      'min_market_cap': 50e8,          # 市值 ≥ 50 亿
      'min_listing_days': 365,         # 上市满 1 年
      'exclude_st': True,              # 排除 ST
      'max_pledge_ratio': 0.7,         # 大股东质押率 < 70%
      'max_goodwill_ratio': 0.3,       # 商誉/净资产 < 30%
      'recent_suspension_check': True, # 最近 12 月停牌天数 < 5
  }
```

  第二层:组合风险

```txt
  portfolio_rules = {
      'total_position_max': 0.95,     # 总仓位 ≤ 95%(留 5% 现金)
      'sector_max': 0.25,              # 单行业 ≤ 25%
      'style_max_deviation': 1.0,      # 风格偏离 ≤ 1σ
      'beta_target': (0.8, 1.2),      # Beta 在 0.8-1.2 之间
      'tracking_error_max': 0.08,     # 跟踪误差 < 8%
      'concurrent_holdings': (20, 50), # 持仓数量 20-50
  }
```

  第三层:策略风险

```txt
  strategy_rules = {
      'monthly_drawdown_pause': -0.10, # 月度回撤 -10% 暂停 1 周
      'rolling_3m_vs_benchmark': -0.05, # 3 月跑输基准 5% 报警
      'rolling_6m_drawdown': -0.20,    # 6 月回撤 -20% 暂停 1 月
      'annual_loss_kill_switch': -0.30, # 年度亏损 -30% 全面停止
      'ic_decay_alert': 0.5,           # IC 衰减到历史 50% 报警
  }
```

  2. 风格暴露监控

```txt
  def compute_style_exposures(portfolio_weights, style_factors):
      """
      计算组合在 Barra 风格因子上的暴露
      """
      exposures = {}
      for style in ['size', 'value', 'momentum', 'quality', 'volatility', 'liquidity']:
          # 加权平均
          exposures[style] = (portfolio_weights * style_factors[style]).sum()

      # 与基准对比
      benchmark_exposures = compute_style_exposures(benchmark_weights, style_factors)
      active_exposures = exposures - benchmark_exposures

      return active_exposures
```

  # 警报:任一风格 |暴露| > 1σ 触发警告

  3. 极端情况预案

```txt
  crisis_response_plan = {
      '大盘单日 -5%': '暂停新增交易,等待 24 小时',
      '大盘单日 -7%': '减仓 30%',
      '持仓平均跌幅 -8%': '强制再平衡,排除跌幅最大的 20% 持仓',
      '量化机构集中平仓信号': '减仓至 30%',
      '商誉爆雷季(11-1月)': '加严基本面过滤(商誉/净资产 < 15%)',
      '年报披露季(3-4月)': '回避业绩预告变脸高风险股',
  }
```

  ---
  Phase 6:执行系统(第 8 月)

  1. 从"想做"到"真做"的鸿沟

  纸上策略 vs 实盘策略的差距:

```txt
  ┌────────────┬────────────┬───────────────────────────┐
  │    因素    │ 纸上(回测) │           实盘            │
  ├────────────┼────────────┼───────────────────────────┤
  │ 价格       │ 收盘价     │ 实际成交价(滑点 0.1-0.3%) │
  ├────────────┼────────────┼───────────────────────────┤
  │ 流动性     │ 假设无限   │ 大单可能拆几天才能买完    │
  ├────────────┼────────────┼───────────────────────────┤
  │ 订单类型   │ 自动成交   │ 限价/市价/竞价的选择      │
  ├────────────┼────────────┼───────────────────────────┤
  │ 执行时间   │ 瞬间完成   │ 实际可能需要 30 分钟      │
  ├────────────┼────────────┼───────────────────────────┤
  │ 系统稳定性 │ 100%       │ 网络/电脑/券商可能宕机    │
  └────────────┴────────────┴───────────────────────────┘
```

  2. 执行选项

  选项 A:券商 API 自动化(推荐)

  可用的券商:
  - 东方财富:有 Level-2 + Python API
  - 国泰君安(QMT、PTrade):专业量化平台
  - 华泰证券(MATIC):量化投资平台
  - 国信证券(TradeMaster):量化通

  优势:
  - 自动执行,消除情绪化
  - 可拆单减少冲击
  - 时间精准

  代价:
  - 需要对接 API(技术门槛)
  - 部分平台需要 50 万+ 资产门槛

  选项 B:半自动(过渡期推荐)

```txt
  # 周一早上 8:00 自动生成"今日交易清单"
  def generate_trade_list():
      target = compute_target_holdings()
      current = read_current_holdings_from_broker()

      trades = []
      for stock in target.index | current.index:
          diff = target.get(stock, 0) - current.get(stock, 0)
          if abs(diff) > 100:  # 至少 100 股
              trades.append({
                  'stock': stock,
                  'action': 'BUY' if diff > 0 else 'SELL',
                  'shares': abs(diff),
                  'order_type': 'LIMIT',
                  'price': last_close * 1.005 if diff > 0 else last_close * 0.995,
              })

      # 输出 Excel,你按清单手动下单
      pd.DataFrame(trades).to_excel('today_trades.xlsx')
```

  优势:
  - 不需要 API,降低门槛
  - 保留人工最终确认环节

  代价:
  - 仍有情绪化破坏纪律的风险
  - 时间成本(每天 30-60 分钟)

  选项 C:全手动(不推荐)

  完全靠自己每天手动操作。99% 的情况会因为情绪化破坏策略。

  3. 智能拆单(若资金 > 100 万)

```txt
  def split_large_order(stock, total_shares, day_volume):
      """
      把大单拆成小单,降低市场冲击
      """
      # 不超过日成交量的 5%
      max_per_day = day_volume * 0.05

      if total_shares <= max_per_day:
          # 单日内分 5 笔下单
          return [{'shares': total_shares / 5, 'time': t}
                  for t in ['9:35', '10:30', '11:00', '13:30', '14:30']]
      else:
          # 跨日执行
          days_needed = math.ceil(total_shares / max_per_day)
          return generate_multi_day_orders(stock, total_shares, days_needed)

```

  Phase 7:监控与迭代(第 9-12 月,持续)

  1. 监控仪表盘

  最小可行监控(每周 30 分钟):

```txt
  weekly_dashboard = {
      # PnL
      'this_week_return': '本周收益',
      'mtd_return': '月初至今',
      'ytd_return': '年初至今',
      'vs_csi300_ytd': '相对沪深 300',

      # 持仓
      'current_positions': '当前持仓数',
      'top_5_holdings': '前 5 大持仓',
      'sector_distribution': '行业分布',
      'avg_holding_days': '平均持有天数',

      # 风险
      'current_drawdown': '当前回撤',
      'max_drawdown_30d': '30 日最大回撤',
      'volatility_30d': '30 日波动率',

      # 因子
      'rolling_3m_ic_value': '滚动 3 月 IC(价值)',
      'rolling_3m_ic_quality': '滚动 3 月 IC(质量)',
      'rolling_3m_ic_reversal': '滚动 3 月 IC(反转)',

      # 健康度
      'red_lights_triggered': '红绿灯触发情况',
  }
```

  2. 月度复盘流程

  每月最后一周:
  1. 计算本月归因(alpha vs beta vs 风格 vs 行业)
  2. 与基准(沪深 300、中证 500)详细对比
  3. 检查每个因子的 IC 表现
  4. 评估当前红绿灯状态
  5. 决定是否调整因子权重
  6. 写月度复盘报告(强制!)

  3. 季度迭代

  每季度末:
  1. 完整 Walk-Forward 验证
  2. 因子半衰期评估
  3. 新因子研发(每季 1-2 个新因子加入候选)
  4. 风险预算调整
  5. 与年度目标对比,决定是否大改

  4. 年度战略

  每年末:
  1. 完整年度回顾
  2. 重新审视所有因子是否仍然有效
  3. 退役失效因子(3-5 个)
  4. 上线新因子(3-5 个)
  5. 重新校准参数
  6. 决定下一年是否继续运行

  ---
  完整 12 个月路线图

```txt
  月份  | 核心任务                       | 投入时间    | 关键产出
  ─────┼────────────────────────────┼──────────┼─────────────────────────
    0  | 思维转换 + 工具准备             | 1 周        | 环境就绪,Hello World 跑通
    1  | 数据基础设施                  | 40-60 小时  | 全样本数据本地化
    2  | 单因子研究(Tier 1)            | 60-80 小时  | 5-7 个核心因子检验完成
    3  | 单因子研究(Tier 2)            | 60-80 小时  | 10-15 个因子库建立
    4  | 多因子合成(Z-Score 加权)       | 40-60 小时  | 综合评分系统 v1
    5  | 因子中性化 + 权重优化           | 40-60 小时  | 综合评分系统 v2
    6  | 完整回测(2018-2025)           | 60-80 小时  | 通过严格回测的策略 v1
    7  | 风险管理体系                  | 30-40 小时  | 三层风控系统
    8  | 执行系统对接                  | 40-60 小时  | 半自动化交易系统
    9  | 小资金实盘(< 总资产 5%)       | 30-40 小时  | 第一个实盘月
   10  | 监控仪表盘 + 第一次月度复盘     | 20-30 小时  | 仪表盘 v1
   11  | 第一次季度迭代                 | 30-40 小时  | 因子权重调整 v1
   12  | 完整年度复盘 + 决策            | 20-30 小时  | 是否扩大资金规模决策
```

  总投入:约 500-700 小时(每周 10-15 小时)

  ---
  关键里程碑与决策点

```txt
  第 6 月末:第一个回测策略
    ├─ 通过 → 进入第 7 月
    └─ 不通过(Sharpe < 0.8 或 MDD > 30%)→ 回到第 2 月,重做因子研究

  第 9 月末:小资金实盘 1 个月
    ├─ 跑赢基准 → 继续小资金
    └─ 大幅跑输 → 暂停,诊断原因

  第 12 月末:完整一年实盘
    ├─ 跑赢基准 ≥ 5% 且 MDD < 30% → 扩大资金至总资产 30%
    ├─ 跑赢基准 0-5% → 维持小资金,继续迭代
    └─ 跑输基准 → 严肃考虑放弃自建,转向私募/公募
```

  ---
  升级路径中的"五大致命陷阱"

  ❌ 陷阱 ①:跳过单因子研究,直接合成

  很多人急于看到"组合表现",跳过单因子检验直接做合成。
  结果: 不知道哪个因子贡献了多少,失效时也不知道改什么。
  正解: 单因子完整检验是地基,不能省。

  ❌ 陷阱 ②:用太多因子

  新手常想"加 30 个因子,组合一定好"。
  结果: 严重过拟合,样本外失效。
  正解: 5-10 个高质量因子好过 30 个平庸因子。

  ❌ 陷阱 ③:不做因子中性化

  直接用原始因子值合成。
  结果: 组合实际暴露在某个风格(如小盘),你以为是 alpha,其实是 beta。
  正解: 因子必须至少做行业 + 市值中性化。

  ❌ 陷阱 ④:回测期太短

  只回测 2-3 年,因为"早期数据不重要"。
  结果: 没经过完整周期检验,2024 微盘股危机重演时崩溃。
  正解: 至少 7 年回测,覆盖牛熊周期。

  ❌ 陷阱 ⑤:实盘前不充分预演

  回测好就直接上 100 万实盘。
  结果: 滑点、流动性、执行问题完全没考虑,回测结果实盘减半。
  正解: 小资金实盘 6 个月以上才能扩大。

  ---
  简化版:如果不想搞这么复杂

  如果觉得12 个月太长、500 小时太多,有 3 个简化路径:

  简化路径 A:只做 Top-N 选股

  - 用 2-3 个最简单因子(EP + ROE + 反转 20D)
  - 等权 Top 20
  - 月度再平衡
  - 大约 80-100 小时投入
  - 预期年化超额 2-5%(不如完整版的 5-10%,但够简单)

  简化路径 B:配置专业产品

  - 50% 私募指增产品
  - 30% 公募量化基金
  - 20% 自己玩(<= 总资产 20%)
  - 核心收益靠专业产品,自建只是学习

  简化路径 C:停止主动管理

  - 50% 沪深 300 ETF + 50% 中证 500 ETF + 一些红利
  - 接受市场平均回报
  - 专注于赚钱(主业)而非省钱(投资)

  ---
  一句话总结整个升级路径

  ▎ 从单股策略升级到多因子选股,本质上不是"做一个更好的策略",
  ▎ 而是"成为一个不一样的投资者"。
  ▎
  ▎ 旧的你:靠看准一只股票赚钱 — 高情商、低胜率、不可复制
  ▎ 新的你:靠有效的评分系统赚钱 — 低情绪、高胜率、可复制可监控
  ▎
  ▎ 这个升级真正改变的,不是钱包,是你这个人。
  ▎
  ▎ 当你完成这个升级,你会发现:
  ▎ - 你不再焦虑"今天应该买什么"
  ▎ - 你不再恐惧"明天会跌多少"
  ▎ - 你不再纠结"该不该补仓"
  ▎ - 你只关心一件事:"我的系统这周还在不在工作"
  ▎
  ▎ 这种心理上的解脱,远比那 5-10% 的超额收益更值钱。
