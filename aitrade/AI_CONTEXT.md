# 量化分析工具 - 执行文档

> **项目定位**: 个人投资工具（含实盘衔接） &nbsp;|&nbsp; **数据覆盖**: A 股 / 港股 / 美股全市场日线 + 关注池（百只量级）分钟级 &nbsp;|&nbsp; **本文档**: 所有模块设计与执行规划的单一信源 (single source of truth)

---

## 📑 文档结构索引


| 章节 | 内容 |
|---|---|
| [项目范围与原则](#项目范围与原则) | 定位、边界、设计原则 |
| [系统总体架构](#系统总体架构) | 跨模块架构图、依赖关系、数据流 |
| [跨模块路线图](#跨模块路线图含-walking-skeleton) | 全局执行顺序、Walking Skeleton 优先 |
| [开发约定与横切关注点](#开发约定与横切关注点) | 工程通用规范 |
| [术语表](#术语表) | 专业术语对照 |
| [数据词典](#数据词典) | 核心字段定义 |
| [性能预估与数据规模](#性能预估与数据规模) | 数据量、性能预算 |
| **[模块 0](#模块-0共享基础设施)** | 共享基础设施 |
| [模块 A](#模块-a可视化模块) | 行情可视化 |
| [模块 B](#模块-b量化分析模块) | 量化分析引擎 |
| [模块 C](#模块-c量化结果可视化模块策略筛选与排名实验台) | 策略筛选实验台 |
| **[模块 D](#模块-d数据接入层)** | 数据接入 |
| **[模块 E](#模块-eai-agent-层)** | AI Agent |
| **[模块 F](#模块-f任务调度与监控层)** | 任务调度与监控 |
| **[模块 G](#模块-g实盘衔接层)** | 实盘衔接 |


---

## 项目范围与原则

### 定位
- **个人投资工具**：单用户使用，无需认证 / 多用户 / 权限系统
- **含实盘衔接**：研究 → 回测 → Paper trading → 小资金实盘的完整链路
- **多市场覆盖**：A 股（主板 / 科创 / 创业 / 北交所）、港股、美股

### 数据范围
- **日线**：A / HK / US 全市场（含退市股、ST 标记、历史成分股）
- **分钟级**：用户维护的关注池（百只量级）
- **基本面**：A 股全市场季报 + 港美关键标的

### 性能约束（个人版预算）
- 单标的日线读取 < 50ms
- 单标的 5 年分钟数据读取 < 500ms（懒加载）
- 全市场单因子 1 年 IC 测试 < 30 秒
- 全市场单策略 5 年向量化回测 < 60 秒
- 实盘信号生成 < 1 秒

### 核心设计原则
1. **PIT 严格性**：所有历史回溯严格按当时可得信息（公告日、当时行业归属、当时 universe）
2. **可复现性**：每个实验结果绑定 `数据快照 ID + 代码 hash + 参数 JSON`
3. **正交分层**：每层只依赖下层（模块 0 是唯一被所有层共享的基础层）
4. **数据契约优先**：跨模块通信通过 schema 化的数据结构，不允许"约定俗成"
5. **研究 / 实盘隔离**：模块 G 是实盘相关代码的唯一入口

---

## 系统总体架构

```
┌──────────────────────────────────────────────────────────────────┐
│  应用层 (Application)                                              │
│  ┌────────────┐  ┌──────────────┐  ┌──────────────┐              │
│  │  模块 A     │  │  模块 C       │  │  模块 E       │              │
│  │  行情可视化  │  │  策略筛选实验台 │  │  AI Agent     │              │
│  └─────┬──────┘  └──────┬───────┘  └───────┬──────┘              │
│        │                │                  │                      │
├────────┼────────────────┼──────────────────┼──────────────────────┤
│        ▼                ▼                  ▼                      │
│  核心层 (Core Engine)                                              │
│  ┌────────────────────────────────────────────────────┐           │
│  │  模块 B：量化分析（L0 数据工程 → L6 实验管理）        │           │
│  │  因子工厂 / 回测引擎 / 组合优化 / 风险模型 / 绩效归因 │           │
│  └─────────────────┬──────────────────────────────────┘           │
│                    │                                              │
├────────────────────┼──────────────────────────────────────────────┤
│                    ▼                                              │
│  执行层 (Execution)                                                │
│  ┌──────────────┐  ┌──────────────────┐                           │
│  │  模块 F       │  │  模块 G           │                           │
│  │  调度与监控    │  │  实盘衔接          │                           │
│  └──────┬───────┘  └────────┬─────────┘                           │
│         │                   │                                     │
├─────────┼───────────────────┼─────────────────────────────────────┤
│         ▼                   ▼                                     │
│  接入层 (Ingestion)                                                │
│  ┌────────────────────────────────────────────────────┐           │
│  │  模块 D：数据接入                                    │           │
│  │  Tushare / AKShare / Polygon / Futu / 实盘行情      │           │
│  └─────────────────┬──────────────────────────────────┘           │
│                    │                                              │
├════════════════════┼══════════════════════════════════════════════┤
│                    ▼                                              │
│  基础层 (Shared) — 被所有模块依赖                                    │
│  ┌────────────────────────────────────────────────────┐           │
│  │  模块 0：共享基础设施                                 │           │
│  │  DataLoader / Calendar / Universe / Watchlist /     │           │
│  │  Storage / Data Contracts / PriceService / ID 规范  │           │
│  └────────────────────────────────────────────────────┘           │
└──────────────────────────────────────────────────────────────────┘

外部依赖：Anthropic API（模块 E） / 券商 API（模块 G） / 数据源 API（模块 D）
```

**依赖规则：**
- 应用层 → 核心层 → 接入层 → 基础层
- 模块 E（AI Agent）通过 Tool Use 可调用任何模块的公开接口
- 模块 0 不依赖任何其他模块
- 模块 G 是研究模块的下游消费者，不可被研究模块反向依赖

---

## 跨模块路线图（含 Walking Skeleton）

### 总体策略
**先 Walking Skeleton，再各模块加肉**。避免出现"模块 A 做完了但取数还没接通"。

| 里程碑 | 周期 | 目标 | 验收 |
|---|---|---|---|
| **M0 Walking Skeleton** | 2 周 | 端到端最小可运行：取数 → 存储 → 显示 | 浏览器输入 `SH:600519` 看到茅台日线 K |
| **M1 单标的研究能力** | 3 周 | 多标的对比 + 指标 + 因子工厂基础版 + 单因子 IC 测试 | 完成 10 个经典因子 + 跑出 IC 报告 |
| **M2 回测与策略筛选** | 4 周 | 完整回测引擎 + 模块 C 排名 | 在矩阵里看到多策略 × 多标的排名 |
| **M3 实盘衔接** | 3 周 | Paper trading + 实时行情 + 风控 | 单策略在 Paper 模式跑一周无异常 |
| **M4 AI 增强** | 4 周 | AI Agent 自然语言 + 异动归因 + 报告 | Agent 能跑一次完整异动归因 |
| **M5 持续打磨** | 持续 | 剩余 P1 / P2 项、性能优化 | - |

### M0 Walking Skeleton 详细任务
- 模块 0：DataLoader 接口 + Parquet 存储约定 + 标的 ID 规范 + Series schema
- 模块 D：Tushare 适配器 + 关注池增量更新框架
- 模块 A：Dash + TVLC 显示单标的日线 K + 时间范围切换
- 模块 F：APScheduler 跑每日数据更新一个任务

### 模块间并行可能性
- M1 期可并行：模块 A 多标的功能 ‖ 模块 B-L1 因子工厂 ‖ 模块 D 数据源扩展
- M2 期可并行：模块 B-L3 回测引擎 ‖ 模块 C 实验矩阵
- M3 / M4 期建议串行，复杂度高

---

## 开发约定与横切关注点

### 项目结构
```
sda/
├── pyproject.toml              # uv + ruff + mypy 配置
├── .pre-commit-config.yaml
├── config/
│   ├── default.yaml
│   ├── trading_rules.yaml      # 涨跌停 / T+1 / 撮合规则
│   └── .env.example
├── data/                       # gitignored
│   ├── raw/
│   ├── processed/
│   ├── factors/
│   ├── snapshots/
│   └── live/
├── core/                       # 模块 0
├── ingestion/                  # 模块 D
├── quant/                      # 模块 B
│   ├── factor_factory/
│   ├── factor_lab/
│   ├── backtest/
│   ├── portfolio/
│   ├── risk/
│   └── experiments/
├── visualization/              # 模块 A & C
├── agent/                      # 模块 E
├── scheduler/                  # 模块 F
├── live/                       # 模块 G
├── tests/
└── notebooks/
```

### 工具链
- **Python**：3.11+
- **依赖管理**：uv（锁定版本，可复现环境）
- **代码质量**：ruff（lint + format）+ mypy（strict 模式）+ pre-commit hooks
- **测试**：pytest + pytest-cov（目标覆盖率 70%+）
- **版本控制**：git + git-lfs（数据快照超 100MB）+ Conventional Commits

### 横切关注点

**日志**
- 结构化日志：`structlog` + JSON 输出
- 按模块切分，日志级别可配
- 实盘相关日志独立通道，永久保留

**配置管理**
- 分层：`config/default.yaml`（基础）→ `config/local.yaml`（个人覆盖）→ `.env`（密钥）
- 工具：`pydantic-settings`
- 关键约束：所有 magic number（交易成本、universe 定义、回测起止）必须在 config，不在代码

**错误处理（三层）**
- **数据源临时错误**：自动重试 + 指数退避（模块 D 处理）
- **数据完整性错误**：告警 + 跳过 + 标记（模块 D / B-L0）
- **逻辑错误**：fail fast，不静默吞错

**测试策略**
- 单元测试：每个 service / 工具函数
- **回测快照测试**（关键）：固定 fixture 数据 → 期望指标在 tolerance 内 → 任何回测引擎改动需通过
- 数据契约测试：Pydantic schema validation
- 集成测试：M0-M5 每个里程碑的端到端流程

**数据备份**
- 本地数据 weekly 增量备份到外部 SSD
- 实盘相关数据 daily 备份 + 云端冷存储

---

## 术语表

| 术语 | 全称 | 含义 |
|---|---|---|
| ADV | Average Daily Volume | 平均日成交量 |
| Brinson Attribution | - | 行业配置 + 个股选择的超额收益分解 |
| BHAR | Buy-and-Hold Abnormal Return | 持有期超额收益 |
| CAR | Cumulative Abnormal Return | 累计超额收益 |
| CPCV | Combinatorial Purged Cross-Validation | de Prado 提出，防止时序泄漏的 CV 方法 |
| DSR | Deflated Sharpe Ratio | 校正多次实验后的真实 Sharpe |
| EVT | Extreme Value Theory | 极值理论（尾部风险） |
| FDR | False Discovery Rate | 错误发现率（Benjamini-Hochberg 校正） |
| HMM | Hidden Markov Model | 隐马尔可夫（regime detection） |
| HRP | Hierarchical Risk Parity | 层次化风险平价（de Prado） |
| IC | Information Coefficient | 因子值与未来收益的截面相关性 |
| ICIR | IC Information Ratio | IC 均值 / IC 标准差 |
| MAD | Median Absolute Deviation | 中位数绝对偏差（去极值） |
| MinTRL | Minimum Track Record Length | 证明 Sharpe 显著所需最小样本长度 |
| OOS / IS | Out-of-Sample / In-Sample | 样本外 / 样本内 |
| PBO | Probability of Backtest Overfitting | 回测过拟合概率（CSCV 方法） |
| PIT | Point-in-Time | 时点真实可得状态 |
| PSR | Probabilistic Sharpe Ratio | Sharpe 高于阈值的概率 |
| RS Line | Relative Strength Line | 相对强弱线 |
| SHAP | SHapley Additive exPlanations | ML 模型特征重要性解释 |
| TR | Total Return | 含分红再投资的总回报 |
| TVLC | TradingView Lightweight Charts | 前端 K 线引擎 |
| Universe | - | 股票池 |
| VWAP / TWAP | Volume / Time Weighted Average Price | 量 / 时加权均价 |

---

## 数据词典

核心字段定义（所有模块共用）：

| 字段 | 类型 | 含义 | 单位 / 格式 |
|---|---|---|---|
| `symbol` | str | 标的 ID | `MARKET:CODE`（如 `SH:600519`） |
| `market` | str | 市场代码 | `SH` / `SZ` / `BJ` / `HK` / `US` |
| `ts` | datetime | 时间戳 | UTC |
| `open` / `high` / `low` / `close` | float | 原始价格（不复权） | 原币 |
| `volume` | int64 | 成交量 | 股 |
| `amount` | float | 成交额 | 原币 |
| `adj_factor` | float | 累计复权因子（前复权基准 = 最新日 1.0） | - |
| `tr_factor` | float | 总收益因子（含现金分红再投资） | - |
| `announce_date` | date | 公告日（财务 PIT 字段，**绝不可用 report_date 代替**） | YYYY-MM-DD |
| `effective_date` | date | 生效日（行业归属变更日、universe 调整日） | YYYY-MM-DD |
| `universe_id` | str | 当时股票池快照 ID | 如 `CSI300_2025-Q1` |
| `currency` | str | 原始币种 | ISO 4217 |
| `is_st` | bool | 当时是否 ST | - |
| `is_suspended` | bool | 当时是否停牌 | - |
| `limit_up` / `limit_down` | float | 当日涨 / 跌停价 | 原币 |
| `is_limit_locked` | bool | 当日是否封板（成交量 < 阈值） | - |

---

## 性能预估与数据规模

### 数据量评估

| 数据 | 量级 | 存储（Parquet + zstd） |
|---|---|---|
| A 股日线（5500 标的 × 30 年） | ~35M 行 | ~800 MB |
| 港股日线（2500 标的 × 30 年） | ~15M 行 | ~400 MB |
| 美股日线（6000 标的 × 30 年） | ~40M 行 | ~1 GB |
| 关注池分钟（200 标的 × 5 年） | ~60M 行 | ~2 GB |
| 财务数据（全市场） | - | ~500 MB |
| 因子缓存（50 因子 × 多 universe） | - | ~1 GB |
| 实盘 / Paper 数据 | - | ~500 MB / 年 |
| **总计** | - | **< 10 GB** |

### 性能瓶颈与方案

| 瓶颈 | 方案 |
|---|---|
| 分钟数据 query 慢 | DuckDB 按 ts 排序 + 月份分文件 + 谓词下推 |
| 因子缓存膨胀 | 按 `(factor_id, version, universe_id)` 哈希分目录，定期 GC |
| 实时取数延迟 | Redis 缓存最近 30 天关注池数据 |
| 大批量回测吞吐 | Joblib 多进程；单回测 ≤ 60s 时不上 Ray |
| 多次相同回测重复算 | `(strategy_hash, params_hash, data_snapshot_id)` 命中复用 |

---

# 模块 0：共享基础设施

> **定位**：被所有模块依赖的基础层，**不依赖任何其他模块**。提供数据访问、ID 规范、时间处理、复权 / TR 计算、Universe 与关注池管理、数据契约（schema）。

## 0.1 数据契约（Schema 定义）

所有跨模块通信的数据结构必须有 schema。建议用 Pydantic v2 + Arrow Table 组合（schema 给 metadata，Arrow 装数据）。

### Series（行情序列）
```python
class Series(BaseModel):
    symbol: str               # SH:600519
    market: Market            # SH / SZ / BJ / HK / US
    frequency: Frequency      # 1m / 5m / 15m / 30m / 60m / 1d / 1w / 1M
    adjust: AdjustType        # NONE / FORWARD / BACKWARD / TOTAL_RETURN
    currency: Currency
    calendar: str
    bars: pa.Table            # ts, open, high, low, close, volume, amount
    metadata: SeriesMeta      # source, version, fetched_at, snapshot_id
```

### FactorValue（因子值）
```python
class FactorValue(BaseModel):
    factor_id: str            # momentum_20d
    factor_version: str
    universe_id: str          # ⚠ 强制标注，避免漂移
    universe_size: int
    industry_classification: str  # e.g. SW2021_v2
    values: pa.Table          # symbol, date, raw_value, rank_pct, neutralized
    metadata: FactorMetadata
```

### BacktestResult（回测结果）
```python
class BacktestResult(BaseModel):
    experiment_id: UUID
    input_spec: InputSpec
    strategy_spec: StrategySpec
    params: dict
    period: DateRange
    
    # 复现三要素
    data_snapshot_id: str
    code_hash: str
    config_hash: str
    
    # 核心指标 (见模块 B-L5)
    metrics: BacktestMetrics
    metrics_is: BacktestMetrics    # In-sample
    metrics_oos: BacktestMetrics   # Out-of-sample
    
    # 明细
    equity_curve: pa.Table         # ts, total_value, cash, position_value, drawdown, returns
    trades: pa.Table               # 完整交易记录
    positions: pa.Table            # 持仓快照（每个 rebalance）
    
    # 过拟合诊断
    overfitting_flags: OverfittingFlags
    
    # 元数据
    cost_model: CostModelConfig
    universe_id: str
    base_currency: Currency
```

### InputSpec / StrategySpec / ScoreConfig（模块 C）
```python
class InputSpec(BaseModel):
    type: InputType  # SINGLE / WITH_BENCHMARK / GROUP / 
                     # CROSS_MARKET_PAIR / CROSS_MARKET_INDICATOR /
                     # BASKET / LONG_SHORT / FACTOR_PORTFOLIO
    symbols: list[SymbolWithRole]
    weight_scheme: WeightScheme  # EQUAL / FIXED / MARKET_CAP / INVERSE_VOL / RISK_PARITY
    alignment: AlignmentMode      # TRADING_DAY / NATURAL_DAY
    
class StrategySpec(BaseModel):
    strategy_id: str
    strategy_version: str
    params: dict
    
class ScoreConfig(BaseModel):
    mode: ScoreMode  # WEIGHTED / PARETO
    weights: dict[str, float]  # metric -> weight
    filters: list[Filter]
    normalization: NormType  # MIN_MAX / RANK_PCT
```

### DataSourceAdapter（模块 D）
```python
class DataSourceAdapter(Protocol):
    name: str
    def supports(market: Market, frequency: Frequency) -> bool
    def fetch_daily(symbols, start, end) -> pa.Table
    def fetch_minute(symbols, start, end, freq) -> pa.Table
    def fetch_financials(symbols, periods) -> pa.Table
    def fetch_dividends(symbols, start, end) -> pa.Table
    def fetch_corporate_actions(symbols, start, end) -> pa.Table
```

### LiveOrder / LiveFill（模块 G）
```python
class LiveOrder(BaseModel):
    order_id: UUID
    strategy_id: str
    symbol: str
    side: Side                # BUY / SELL / SHORT / COVER
    quantity: int
    order_type: OrderType     # MARKET / LIMIT / STOP / STOP_LIMIT
    price: Optional[float]
    tif: TimeInForce          # DAY / GTC / IOC / FOK
    submitted_at: datetime
    status: OrderStatus       # PENDING / SUBMITTED / PARTIAL / FILLED / CANCELED / REJECTED
    venue: str                # PAPER / QMT / FUTU / ALPACA / IB
    risk_check_pass: bool

class LiveFill(BaseModel):
    fill_id: UUID
    order_id: UUID
    symbol: str
    quantity: int
    price: float
    timestamp: datetime
    commission: float
    venue: str
```

## 0.2 DataLoader 抽象

```python
class DataLoader(Protocol):
    def get_series(symbol, start, end, frequency='1d',
                   adjust='FORWARD', universe=None) -> Series
    def get_series_batch(symbols, ...) -> dict[str, Series]
    def get_universe(name, date) -> list[str]
    def get_calendar(market) -> ExchangeCalendar
    def get_factor(factor_id, version, ...) -> FactorValue
    def get_financials(symbol, period_type, start, end) -> pa.Table
    def get_corporate_actions(symbol, start, end) -> pa.Table
    def get_symbol_mapping(input, as_of) -> str   # 别名 / 历史代码解析
```

实现：
- `LocalDataLoader`：从 `data/processed/` 读 Parquet，DuckDB 查询
- `CachedDataLoader`：装饰器，加 LRU 内存 + Redis 短期缓存
- `SnapshotDataLoader`：绑定 data_snapshot_id，用于 PIT 复现
- `InMemoryDataLoader`：测试用

## 0.3 存储约定

```
data/
├── raw/                          # 数据源直接落盘（保留审计追溯能力）
│   ├── tushare/
│   ├── akshare/
│   ├── polygon/
│   └── futu/
├── processed/                    # 标准化后供 DataLoader 使用
│   ├── daily/{market}/{symbol}/{year}.parquet
│   ├── minute/{market}/{symbol}/{year}-{month}.parquet
│   ├── financials/{symbol}/{period_type}.parquet
│   ├── adjust_factors/{symbol}.parquet
│   ├── tr_factors/{symbol}.parquet
│   ├── universe/{name}/{snapshot_date}.parquet
│   ├── industry/{classification}/{version}/{date}.parquet
│   ├── symbol_mapping/all.parquet
│   ├── fx/{pair}.parquet
│   └── corporate_actions/{symbol}.parquet
├── factors/{factor_id}/{version}/{universe_id}/{year}.parquet
├── snapshots/{snapshot_id}/      # PIT 视图元数据
├── experiments/{experiment_id}/  # 回测结果
└── live/
    ├── paper/{strategy_id}/{date}/{orders, fills, positions}.parquet
    ├── real/{strategy_id}/{date}/...
    └── reconciliation/
```

编码约定：
- Parquet + zstd 压缩（level 3，平衡读速 / 压缩率）
- 按 ts 列排序（谓词下推友好）
- 字符串列字典编码

查询层：DuckDB 直接 `SELECT FROM read_parquet(...)`，不导入

## 0.4 交易日历服务

基于 `exchange_calendars`，三市分开：`XSHG`（上交所）/ `XSHE`（深交所）/ `XHKG`（港交所）/ `XNYS`（纽交所）/ `XNAS`（纳斯达克）

```python
class CalendarService:
    def is_trading_day(market, date) -> bool
    def next_trading_day(market, date, n=1) -> date
    def prev_trading_day(market, date, n=1) -> date
    def n_days_before(market, date, n) -> date
    def trading_days_between(market, start, end) -> list[date]
    def session_open(market, date) -> datetime         # UTC
    def session_close(market, date) -> datetime
    def is_half_day(market, date) -> bool              # 港股半日市等
    def overlap_calendar(markets: list[Market]) -> Calendar  # 跨市场交集
```

特殊处理：
- A 股：沪深节假日（清明 / 春节 / 国庆等）、补班日
- 港股：半日市（圣诞前夜、平安夜、除夕）
- 美股：早闭市（Black Friday、Christmas Eve）

## 0.5 Universe 服务（含 PIT）

```python
class UniverseService:
    def get(name, date) -> list[str]
    def history(name, start, end) -> dict[date, list[str]]
    def diff(name, date_a, date_b) -> AddRemoveDiff
    def custom(symbols, name) -> UserUniverse
```

内置 universe：
- `CSI300` / `CSI500` / `CSI1000` / `CSI_ALL` / `STAR50` / `CHINEXT50`
- `SP500` / `NASDAQ100` / `RUSSELL2000`
- `HSI` / `HSCEI` / `HSTECH` / `STOCK_CONNECT_HK_NORTH` / `STOCK_CONNECT_HK_SOUTH`

关键：所有内置 universe 维护**月度历史快照**，保证 PIT。

## 0.6 关注池管理（个人版核心）

```python
class Watchlist(BaseModel):
    id: str
    name: str
    description: str
    symbols: list[str]
    tags: list[str]       # ["AI", "新能源", "实盘候选"]
    auto_update: bool     # 是否自动从某因子 top N 更新
    update_rule: Optional[UpdateRule]
    created_at: datetime
    updated_at: datetime
```

操作：CRUD / 标签管理 / 合并 / 从因子排名 top N 导入 / 导出

后端存储：SQLite（操作数据，不走 Parquet）

数据维护：关注池内的标的自动纳入模块 D 的分钟数据增量更新

UI 入口：模块 A 顶栏快速切换关注池

## 0.7 标的 ID / 时间 / 币种规范

**标的 ID**
- 格式：`MARKET:CODE`
- A 股：`SH:600519` `SZ:000858` `BJ:430047`
- 港股：`HK:00700` `HK:09988`
- 美股：`US:AAPL` `US:BABA`（OTC 用 `US:OTC:xxx`）
- 指数：`SH:000300` `US:SPX` `HK:HSI`
- ETF：`SH:510300` `US:SPY`
- 别名解析：用户输入 `茅台` / `贵州茅台` → `SH:600519`（通过 symbol_mapping）

**时间**
- 存储：一律 UTC
- 展示：按用户时区
- 财务字段必须有 `announce_date`（不是 `report_date`）

**币种**
- ISO 4217：`CNY` / `HKD` / `USD`
- 汇率：模块 D 拉，存到 `processed/fx/`
- 跨币种计算时显式指定 `base_currency`

## 0.8 PriceService：复权与 Total Return（量化 P0-1 修复）

**关键澄清**：四种价格序列必须明确区分，不可混用。

| 序列 | 用途 | 实现 |
|---|---|---|
| `raw_close` | 展示真实价 + 除权点标记 | 原始价 |
| `forward_adj_close` | 图表默认（前复权，看图友好） | `raw_close × adj_factor` |
| `backward_adj_close` | 回测的价格基础（连续性） | `raw_close × adj_factor / adj_factor_today` |
| `total_return_close` | **策略收益计算的唯一基础** | 含现金分红再投资 |

**策略收益必须用 Total Return**。A 股年均分红收益率约 2%，港美股更高，不用 TR 会让 Sharpe 系统性偏低，年化收益少算几个百分点。

```python
class PriceService:
    def get_raw(symbol, start, end, frequency) -> Series
    def get_adjusted(symbol, ..., adjust: AdjustType) -> Series
    def get_total_return(symbol, ...) -> Series   # 策略回测必须用
    def get_returns(symbol, ..., return_type='log'|'simple',
                    basis='price'|'total_return') -> pa.Table
    def get_dividend_yield(symbol, date) -> float
```

实现：单独存 `adjust_factors` 和 `tr_factors`，按需在 query 时计算。

## 0.9 股票代码生命周期管理（量化 P0-5 修复）

A 股大量改名 / 退市 / 重组 / ST 加摘 case，必须维护历史 mapping。

```python
class SymbolMapping(BaseModel):
    symbol: str                          # 当前 canonical ID
    historical_ids: list[HistoricalId]   # 历史 ID 与生效区间
    historical_names: list[HistoricalName]
    listing_date: date
    delisting_date: Optional[date]
    restructuring_events: list[Event]    # 合并 / 分拆 / 借壳 / 退市重组
    is_active: bool

class SymbolMappingService:
    def resolve_as_of(input: str, date: date) -> str
        """把 t 时点的代码/名称解析为当前 canonical ID"""
    def history(symbol: str) -> SymbolHistory
    def is_active(symbol, date) -> bool
    def successor(symbol, date) -> Optional[str]    # 重组后的承继股
```

典型 case：
- ST → ST* → 摘帽 → 改名 → 退市 → 重新上市
- 海外回归（中概股私有化后 A 股借壳）
- A+H 分时上市

实现：从 Tushare / AKShare 的 `stock_basic` + `name_change` + `delisting` 表合成，定期更新。

## 0.10 数据快照管理（PIT 复现）

```python
class DataSnapshot:
    snapshot_id: str          # YYYYMMDD_<hash>
    created_at: datetime
    description: str
    tables: dict[str, str]    # table_name -> as_of_date
    
class SnapshotService:
    def create(description) -> str
    def get_loader(snapshot_id) -> DataLoader
    def list_snapshots() -> list[DataSnapshot]
    def cleanup(retention_days=180)
```

工作机制：快照不复制数据，只记录"截止某日各表的有效记录"。通过 metadata 表实现：查询时过滤 `record_date <= snapshot_date`。

每个 `BacktestResult` 必须绑定一个 `data_snapshot_id`，否则不能保证复现。

---

# 模块 A：可视化模块

> 本节用于指导可视化模块的开发执行。**所有数据访问通过模块 0 的 DataLoader / PriceService 完成，不在本模块内自建。**

## A.1 目标与范围

构建支持 A 股 / 港股 / 美股的专业级行情可视化工具，覆盖日线与分钟级别，支持多标的对比、跨市场分析、指数相对强弱分析，并为后续因子研究、回测、AI Agent 辅助分析提供可视化基础。

**核心使用场景：**
- 研究员快速查看任意标的任意时段的 K 线
- 多标的横向对比（相关性、联动性）
- 个股相对大盘的超额表现分析
- 分钟级别盘口与异动观察（仅关注池）
- 因子值、回测信号、事件标记的叠加展示

## A.2 技术栈

| 层 | 选型 | 说明 |
|---|---|---|
| K 线引擎 | **TradingView Lightweight Charts (TVLC)** | 性能极佳，原生支持多 pane、跨 pane 联动十字光标 |
| 辅助图表 | **Plotly** | 热力图、相关性矩阵、分布图等 |
| 表格 | **AG Grid Community** | 排名 / 实验矩阵需高级表格能力 |
| 前端框架 | **Dash**（M0–M2）→ **FastAPI + Vue/React**（成熟后） | 先快速出原型 |
| 数据查询 | DuckDB（通过模块 0 DataLoader） | 不直接访问 Parquet |

**淘汰：** mplfinance（静态）/ 纯 Streamlit（多面板联动差）/ ECharts（分钟懒加载麻烦）/ PyQt（开发成本高）

## A.3 整体架构

```
┌──────────────────────────────────────────────────┐
│  顶部控制栏                                        │
│  [Watchlist▼] [Symbol Picker] [Range] [Freq]     │
│  [复权] [Mode: Overlay/Stacked]                  │
├──────────────────────────────────────────────────┤
│                                                  │
│   主图区 (TVLC, 支持 overlay / 多 pane)          │
│                                                  │
├──────────────────────────────────────────────────┤
│   副图区 (成交量 / 指标 / 资金流)                 │
├──────────────────────────────────────────────────┤
│   分析面板 (相关性矩阵 / 统计 / 事件列表)          │
└──────────────────────────────────────────────────┘
```

**核心抽象：** 所有渲染逻辑只依赖模块 0 的 `Series` 接口，不直接管数据源。

## A.4 功能实现细节

### A.4.1 任意股票任意时段 K 线
- **Symbol Picker**：单输入框，跨三市自动补全，前缀区分（`SH:600000` / `HK:00700` / `US:AAPL`），通过 `SymbolMappingService.resolve_as_of` 支持别名（"茅台"等）
- **时间选择**：快捷按钮（1M / 3M / 6M / 1Y / 3Y / All）+ 自定义日期范围
- **频率切换**：日 / 周 / 月（前端聚合）；分钟级走独立数据通道（仅关注池）
- **复权 toggle**：`raw` / `forward` / `backward` / `total_return`（默认 `forward` 看图，**回测进入模块 C 时强制切 `total_return`**）
- 切换复权时必须重算所有副图指标（MA 等会变）

### A.4.2 多标的对比 - 双模式

**Overlay 模式（叠加同图）**
- 必须**归一化**（起始日 = 100 或 = 0%）
- 支持三种 Y 轴模式：`价格` / `涨跌幅` / `对数收益`
- 不同颜色 + 图例 + hover 高亮当前线
- TVLC 多个 `addLineSeries` 共存，自动同步十字光标

**Stacked 模式（多 pane 上下排列）**
- TVLC 原生 multi-pane，跨 pane 共享 `timeScale` + crosshair
- 每个 pane 独立 Y 轴
- pane 高度可拖拽
- 一键切换 Overlay ↔ Stacked

### A.4.3 跨市场（A 股 / 港股 / 美股）

| 问题 | 方案 |
|---|---|
| 交易时段不同 | 两种 X 轴：①自然日对齐（缺失日空缺）②交易日序号对齐（消除非交易日） |
| 节假日不同 | 通过 `CalendarService.overlap_calendar(...)` 取交集 / 并集由用户选 |
| 时区 | 模块 0 统一 UTC 存储，前端按用户时区展示 |
| 币种 | `原币 / 统一折美元 / 涨跌幅` 三种模式（汇率从 PriceService 取） |
| 颜色规范 | A 股红涨绿跌，美股绿涨红跌，提供配色方案切换 |
| **隔夜跳空** | 跨时区 stacked 模式必须明确展示跳空缺口，不可平滑掉 |

### A.4.4 分钟级数据（仅关注池）

- **窗口化懒加载**：仅请求当前可视窗口 + 左右 1 倍 buffer，通过 TVLC 的 `subscribeVisibleTimeRangeChange` 触发
- **多分辨率金字塔**：模块 D 预聚合 `1m / 5m / 15m / 30m / 60m`，缩放自动切分辨率
- **分时图独立组件**：A 股分时图（价格 vs 均价 + 成交量）是独立形态
- **盘前盘后**：美股区分 pre / regular / post，用底色区分
- **关注池外的标的**：尝试取分钟数据时提示"请先加入关注池"

### A.4.5 股票 vs 指数（相对强弱）

四种模式：
1. **归一化叠加**：股票与指数都 rebased to 100
2. **相对强弱线 (RS Line)**：`股票 / 指数` 比值
3. **超额收益累计图**：`cumprod(1 + r_stock - r_index)`（**收益用 Total Return**）
4. **滚动 Alpha / Beta**：60 日滚动回归

基准池：
- A 股：CSI300 / CSI500 / CSI1000 / 申万一级行业
- 美股：SPX / NDX / RUT
- 港股：HSI / HSCEI / HSTECH

## A.5 专业增强功能（按优先级）

### P0 — 专业工具门槛
- 成交量副图（按涨跌染色）
- 技术指标叠加：MA(5/10/20/60/250) / BOLL / MACD / RSI / KDJ
- 事件标记：财报日 / 分红除权 / 停复牌 / ST 加摘 / 指数纳入剔除
- 涨跌停标记（A 股专属，含科创 / 创业 / 北交所不同规则色彩区分）

### P1 — 专业研究刚需
- 多周期联动：日 / 60 分 / 5 分三窗口同步
- 相关性矩阵热力图：N 标的 Pearson + Spearman，可切换时间窗口
- 滚动相关性曲线：60 / 120 日滚动相关（pair trading 基础）
- 回放模式 (Replay)：历史某日逐根 K 线播放

### P2 — 量化研究深度
- 回测信号叠加（数据来自模块 B-L3）
- 持仓 / PnL 视图：权重堆叠图 + 累计收益 + 回撤
- 因子值时序图（数据来自模块 B-L1）
- 分组收益图（数据来自模块 B-L2）
- 波动率锥：当前 IV / 历史 RV 各分位数
- 截面对比：小提琴图 / 箱线图

### P3 — 资金 / 情绪维度
- 北向资金净流入叠加（A 股专属）
- 龙虎榜热度标记
- 板块强弱热力图：申万一级行业 treemap
- 市场宽度指标：涨跌家数比、新高新低数

### P4 — 交互体验
- 多 workspace：保存工作区
- 画图工具：趋势线 / 斐波那契 / 矩形标注
- 截图导出 + 自动加水印

## A.6 落地阶段

| 阶段 | 周期 | 交付 |
|---|---|---|
| **A.1 MVP** | M0 内完成 | 单标的日线 K + 成交量 + 时间范围 |
| **A.2 多标的** | M1 内 | Overlay / Stacked + 跨市场 |
| **A.3 分钟级** | M1 内 | 关注池分钟懒加载 + 分时图 |
| **A.4 指数对比** | M2 内 | 四种相对强弱模式 + 滚动 Alpha/Beta |
| **A.5 指标 & 事件** | M2 内 | TA-Lib 指标 + 事件 marker |
| **A.6 专业增强** | M4 内 | 相关性矩阵 + Replay + 多周期联动 |
| **A.7 量化集成** | 持续 | 回测信号 / 因子 / AI Agent 接入 |

## A.7 关键风险

1. 分钟数据全量塞前端会卡死 → 懒加载 + 分辨率金字塔
2. 跨市场对齐没有"完美"方案 → 明确标注对齐策略
3. 复权切换要重算所有副图指标
4. TVLC 中文文档少 → 看官方 TS 文档 + 源码
5. Dash callback 性能瓶颈 → 合并状态到单一 Store

---

# 模块 B：量化分析模块

> 研究、回测、组合构建、风险管理的核心引擎。本模块**严格依赖模块 0**（数据访问、价格服务、universe、calendar）和模块 D（数据接入），为模块 A / C / E 提供能力。

## B.1 模块全景与分层

```
┌─────────────────────────────────────────────────┐
│  L6  实验管理 / 报告生成                          │
├─────────────────────────────────────────────────┤
│  L5  风险模型 + 绩效归因                          │
├─────────────────────────────────────────────────┤
│  L4  组合优化 (Portfolio Construction)           │
├─────────────────────────────────────────────────┤
│  L3  回测引擎 (Vectorized + Event-Driven)        │
├─────────────────────────────────────────────────┤
│  L2  因子研究台 (Factor Lab)                     │
├─────────────────────────────────────────────────┤
│  L1  因子工厂 (Factor Factory)                   │
├─────────────────────────────────────────────────┤
│  L0  数据工程辅助层 (PIT 严格性 / 数据质量监控)    │
└─────────────────────────────────────────────────┘
            ↓ 依赖
┌─────────────────────────────────────────────────┐
│  模块 0：共享基础设施                              │
└─────────────────────────────────────────────────┘
```

**架构原则：每层只依赖下层，不允许跨层调用。**

## B.2 L0 数据工程辅助层

> ⚠️ 注意：基础数据访问已下沉到模块 0。本层仅负责量化研究特有的数据质量保障。

- **PIT 严格性校验**：任何使用 t 日数据时只能用 t-1 日及之前可得信息；提供 `@pit_safe` 装饰器自动检查
- **数据质量监控（深度版）**：覆盖模块 D 之外的因子级问题——异常 IC 突变、universe 漂移、行业归属变更前后断层
- **数据快照管理 API**：基于模块 0 SnapshotService 的研究侧封装
- **退市股 / ST 标的处理工具**：模块 0 提供原始数据，本层封装"按因子条件去除 ST"等便捷函数

## B.3 L1 因子工厂

### 核心能力
- **因子定义 DSL**：类似 Qlib 表达式或 WorldQuant Alpha101
  ```python
  factor = "Ref($close, -20) / $close - 1"   # 20 日动量
  ```
- **惰性计算 + 缓存**：因子值落盘 Parquet（见模块 0 存储约定），按 `(factor_name, version, universe_id, date_range)` 做缓存键
- **依赖图**：复合因子依赖基础因子，DAG 调度复用中间结果

### 因子预处理流水线（关键）
1. **去极值**：MAD / 3σ / 分位数 winsorize
2. **标准化**：z-score / rank / 行业内 z-score
3. **中性化**：行业中性化（dummy 回归）+ 市值中性化（线性回归取残差）
4. **缺失处理**：截面中位数填充 / 删除

### 因子库管理
每个因子带元数据（描述、作者、版本、IC 历史、依赖、universe、行业分类版本），分类（价值 / 动量 / 质量 / 情绪 / 技术 / 另类 / 跨市场）。

### 财务因子与价格因子频率对齐（量化 P1-10 修复）
- **forward-fill 严格按 announce_date**（不能用披露后才能知道的数据）
- 季报披露集中期的"信息悬崖"：提供 `is_in_announcement_window` 标记
- 财务因子的"陈旧期"控制：超过 6 个月未更新视为失效

### 行业分类版本化（量化 P1-11 修复）
- 因子值绑定使用的行业分类版本（如 `SW2021_v2`）
- 申万行业历史调整通过 `effective_date` 标注
- 跨版本对比时强制告警

### Universe 标注（量化 P1-12 修复）
- `FactorValue.universe_id` 必填，不允许"未知"
- 跨 universe 复用因子值时强制重算或显式确认

## B.4 L2 因子研究台

### 单因子测试（必备）
- **IC / Rank IC**：每期截面相关系数
- **IC IR**：IC 均值 / IC 标准差
- **t 统计 + p 值**：IC 序列显著性
- **IC 衰减曲线**：T+1 到 T+20

### 分组回测
- 按因子值分 N 组（5 或 10）
- 各组累计收益曲线 + 多空组合收益（**收益用 Total Return**）
- 换手率分析
- 行业分布稳定性

### 因子组合工具
- **因子相关性矩阵**
- **因子正交化**：Schmidt 正交、对称正交
- **多因子合成**：等权 / IC 加权 / IC IR 加权 / 最大化 ICIR 加权 / ML 合成

### 多重检验校正与显著性深度（量化 P0-3 修复）

测 N 个因子时，**任何单因子的 p 值都必须校正**：

- **Bonferroni**：保守，`p_adj = N × p`
- **Benjamini-Hochberg FDR**（推荐）：控制错误发现率
- **Deflated Sharpe Ratio (DSR)**：校正多次实验后的真实 Sharpe（de Prado）
  ```
  DSR = ((SR - SR_threshold) × √(T-1)) / √(1 - γ³SR + (γ⁴-1)/4 × SR²)
  ```
- **Probabilistic Sharpe Ratio (PSR)**：Sharpe 高于阈值的概率
- **Minimum Track Record Length (MinTRL)**：证明 Sharpe 显著所需样本长度
- **PBO (Probability of Backtest Overfitting)**：用 CSCV 检测过拟合概率

模块 C 的"过拟合告警标识"必须基于以上指标实现，不能只是定性判断。

## B.5 L3 回测引擎

### 两套回测引擎
| 引擎 | 实现 | 适用 | 速度 |
|---|---|---|---|
| **向量化** | pandas / numpy / numba | 因子筛选、批量参数扫描 | 每秒数百到数千次 |
| **事件驱动** | 自建（500-800 行） | 真实策略验证、多资产、复杂订单 | 单次回测 30s–5min |

### 价格口径（量化 P0-1 修复）
- **策略收益必须从模块 0 PriceService.get_total_return() 取价**
- 净值曲线按 TR 计算
- 任何收益相关的指标（Sharpe / 年化 / 累计收益）均基于 TR

### 净值与资金口径（量化 P0-4 修复）
- 净值 = `(现金 + 持仓市值) / 初始资金`
- 持仓市值用收盘价计算
- 现金按可配置利率计息（默认：A 股按 7 天逆回购 ≈ 2%，美股按 SOFR）
- **Burn-in 期**：前 N 期不计入回测指标（N 由策略指标预热期决定，如 MA(60) 策略 N=60）
- 多账户隔离：每个子策略独立账户，组合层独立账户
- 跨币种：明确 `base_currency`，每日按当日汇率折算

### 撮合机制完整规则（量化 P0-2 修复）

不同市场板块涨跌停 / 集合竞价规则：

| 市场 / 板块 | 涨跌停 | 集合竞价 | T+? | 特殊规则 |
|---|---|---|---|---|
| 上证 / 深证主板 | ±10% | 9:15-9:25 | T+1 | - |
| 创业板 | ±20% | 9:15-9:25 | T+1 | 注册制后 |
| 科创板 | ±20% | 9:30 后 5 分钟 | T+1 | - |
| 北交所 | ±30% | 9:15-9:25 | T+1 | - |
| ST / *ST | ±5% | 同主板 | T+1 | ST 标记 |
| 新股首日 | 无 | 同板块 | T+1 | N 标记 |
| 港股 | 无 | 9:00-9:30 | T+0 | 仙股流动性差 |
| 美股 NMS | 无 | 4:00-9:30 pre | T+0 | - |
| 美股 OTC | 无 | - | T+0 | 流动性差，单独识别 |

涨跌停撮合细节：
- **涨停封单按比例成交**：`fill_qty = order_qty × (当日成交量 / (买盘 + 自身订单))`
- 跌停封单同理
- **不可成交场景**：涨停日不能新增买单、跌停日不能新增卖单
- 模块 D 提供 `is_limit_locked` 字段（基于真实成交量分布判断）

集合竞价规则：
- **开盘价是集合竞价产物**，不是可任意成交价
- 大单可能不能在开盘成交（量过大）
- 模块 0 提供 `auction_open_price` 字段

撮合价模式：
- `open` / `close` / `vwap` / `twap` / `next_open`（最常用，避免未来函数）

其他细节：
- **停牌**：持仓继续持有，不更新价格，复牌后按真实价处理
- **分红除权**：现金分红自动入账（TR 计算），送转股调整持仓数量
- **保证金**：融资融券、期货保证金、强平规则
- **未来函数检测**：自动校验任何 t 日决策只用 t-1 数据

### 流动性约束（量化 P1-7 修复）

回测内部必须的限制：
- **单日单股交易量 ≤ ADV(20) × X%**（默认 X=10）
- **大单拆分模拟**：超过限制的订单自动拆到多日
- 触及不可成交时：`skip` / `defer_to_next_day` / `use_vwap`（可配置）
- 提供 `liquidity_warning` 标记到 `trades` 表

### 冲击成本模型（量化 P1-8 修复）

三档可选：
- **L1 固定 bp**：简单，适合粗筛
- **L2 Square-Root impact**：`cost = α × σ × √(Q/ADV)`，α 可由实盘数据校准
- **L3 Almgren-Chriss 最优执行**：含临时 + 永久冲击，适合大单场景

实盘上线后用模块 G 的"滑点反向校准"持续更新 α 参数。

### 高阶功能
- **Walk-forward 测试**：滚动训练 + 样本外测试
- **CPCV (Combinatorial Purged Cross-Validation)**（量化 P2-13 修复）：比 Purged K-Fold 更彻底防时序泄漏
- **多策略组合**：子账户独立 + 资金分配 + 风险预算
- **参数敏感性分析**：参数热力图 + 稳健性评分
- **极端事件子集回测**：2015 / 2018 / 2020 / 2022 单独跑

## B.6 L4 组合优化

### 基础权重方案
- 等权 / 市值加权 / 反波动率 / 风险平价
- **HRP (Hierarchical Risk Parity)**（量化 P1-9 升级）：层次聚类 + 递归二分，对小样本协方差更稳健
- **Kelly Criterion**（量化 P2-15 修复）：完整 Kelly、分数 Kelly、约束 Kelly
- **风险预算 (Risk Budgeting)**：按风险贡献分配权重

### 优化范式
- **均值-方差** (Markowitz)
- **Black-Litterman**：引入主观观点
- **Robust optimization**：考虑参数不确定性

### 典型约束（必须全部支持）
- 个股权重上下限
- 行业偏离上限
- 风格因子暴露上限（市值 / 估值 / 动量）
- 换手率上限
- 跟踪误差上限
- 做空限制（A 股大部分场景禁止）
- **与已上线策略相关性上限**（实盘候选场景）

### 求解器
- `cvxpy` 主力
- `riskfolio-lib` 高层封装

## B.7 L5 风险模型 + 绩效归因

### 风险模型
- **多因子风险模型（Barra 风格）**
  - 国家因子 + 行业因子 + 风格因子（市值 / 估值 / 动量 / 波动 / 流动性 / 质量 / 成长 / 杠杆 / Beta）
  - 个股特质风险
- **协方差矩阵估计**
  - 样本协方差（基准）
  - **Ledoit-Wolf 收缩**（推荐）
  - **EWMA 协方差**（量化 P1-9 升级）：跟踪 regime
  - **Graphical Lasso (GLasso)**：稀疏化估计
  - 因子模型协方差
- **VaR / CVaR** + **压力测试**（历史极端情景）
- **EVT (Extreme Value Theory)**（量化 P1-9 升级）：尾部风险（POT / Block Maxima 方法）
- **Regime Detection**（量化 P1-9 升级）：HMM / 变点检测 (PELT) 识别市场状态切换
- **流动性风险**：Amihud illiquidity ratio + 持仓 vs ADV 占比

### 绩效指标（必备清单）
- **收益类**：年化收益、累计收益、月胜率、年胜率
- **风险类**：年化波动、最大回撤、回撤持续期、下行波动
- **风险调整**：Sharpe / Sortino / Calmar / Omega / Information Ratio
- **与基准对比**：超额收益、跟踪误差、Alpha / Beta
- **稳健性**：DSR / PSR / MinTRL / PBO
- **所有指标都要有滚动版本**（60 日滚动 Sharpe 等）

### 归因分析
- **Brinson 归因**：超额收益 = 行业配置 + 个股选择
- **因子归因**：超额收益分解到各风格因子暴露
- **交易归因**：择时收益 vs 选股收益
- **SHAP / Permutation Importance**（量化 P2-14 修复）：ML 模型可解释性

## B.8 L6 实验管理 + 报告生成

### 实验追踪
- 每次回测自动记录：参数、代码 hash、数据快照 ID、结果
- 工具：MLflow 或自建（建议自建，与 BacktestResult schema 一致）

### 回测对比
- 多个回测横向对比（指标 + 净值曲线叠加）
- 数据流：模块 C 的实验矩阵直接读这里

### 因子版本管理
- 因子代码 + 配置 git 化
- 重大修改时 bump version，旧版本仍可加载

### 自动报告（量化 P2-18 修复）

报告模板：
- **因子卡片**：描述、IC 历史、分组收益、回撤、行业分布、相关因子
- **策略月报 / 季报**：净值、归因、风险敞口、参数变化
- **实盘日报**：当日 PnL、持仓、订单、风险指标、异常告警
- **实盘月报**：live vs backtest 对比、策略衰减检测、滑点校准报告

接 AI Agent (模块 E) 自动撰写解读。

## B.9 专业坑（必读）

1. **过拟合是头号杀手** — 参数越多越要 walk-forward + 样本外 + 多市场验证
2. **幸存者偏差** — universe 必须含退市股
3. **未来函数 (Look-ahead Bias)** — 财务用 announce_date、价格用 t-1、避免任何 `iloc[i+1]`
4. **交易成本** — 高换手策略不算成本都能赚钱，加上后几乎全死
5. **流动性陷阱** — 小盘股回测亮眼但实盘进不去，加 ADV 流动性过滤
6. **数据窥探 (Data Snooping)** — 在同一数据集上测 1000 个因子，总有几个"显著"——必须 DSR / PBO 校正
7. **回归到均值** — 择时类策略尤其要警惕
8. **样本量** — 年化指标至少要 5 年数据
9. **市场状态切换** — 牛熊环境下因子有效性可能反转
10. **极端事件** — 别让 2015 / 2018 / 2020 / 2022 污染回测均值
11. **Total Return vs Price Return 混用** — 策略收益必须 TR，否则系统性偏低
12. **撮合机制太理想** — 默认能在任意价位成交是错误的，必须按各市场撮合规则
13. **多重检验未校正** — 测 N 个因子时单因子 p 值无意义
14. **行业归属用现在版本** — 必须用当时归属，否则 Brinson / 中性化全错
15. **Universe 漂移** — 因子在 CSI300 上测试有效，套到 CSI1000 可能完全失效

## B.10 技术栈

| 用途 | 选型 |
|---|---|
| 数值计算 | NumPy + Pandas + Polars（探索）+ Numba（热点提速） |
| 因子计算 | 自建 DSL 或 Qlib 表达式引擎 |
| 优化 | cvxpy + riskfolio-lib |
| 回测 | 向量化自建 + 事件驱动自建 |
| 风险模型 | 自建 + statsmodels + arch（GARCH/EVT） |
| 机器学习 | LightGBM（主力）+ PyTorch（深度学习） |
| 模型解释 | SHAP |
| 时序 CV | sklearn TimeSeriesSplit + 自建 Purged K-Fold + CPCV |
| 实验追踪 | 自建 + Parquet 存储 |
| 并行 | Joblib（默认）/ Ray（大规模时） |

## B.11 落地优先级

| 优先级 | 模块 | 里程碑 |
|---|---|---|
| **M1** | L1 因子工厂基础版（10 个经典因子） | - |
| **M1** | L2 单因子测试（IC / 分组回测） | - |
| **M2** | L3 向量化回测（完整撮合规则） | - |
| **M2** | L5 绩效指标 + Brinson 归因 | - |
| **M2** | L3 事件驱动回测 | - |
| **M3** | L4 组合优化（等权 → MVO → HRP） | - |
| **M4** | L5 Barra 风险模型 + 多重检验校正 | - |
| **M5** | L6 实验管理 + 自动报告 | - |

## B.12 Universe 标注规则

所有因子值、回测结果**必须强制标注**：
- `universe_id` (e.g. `CSI300_2025Q1`)
- `universe_version`
- `universe_size` (snapshot date 的标的数)
- `industry_classification` (e.g. `SW2021_v2`)
- 跨 universe 使用时强制告警（模块 C 实验矩阵在矩阵 cell 上必须可见 universe 标签）

## B.13 与其他模块的对接

| 模块 | 对接点 |
|---|---|
| 模块 0 | 取数 / 价格服务 / 日历 / Universe / 关注池 |
| 模块 D | 数据接入 / 实盘行情 / 财务数据 |
| 模块 A | L1 因子值、L3 回测信号、L5 净值与回撤曲线输出到组件 |
| 模块 C | L3 BacktestResult 作为输入；L2 单因子结果展示 |
| 模块 E | L2 单因子测试、L3 回测、L6 报告作为 tool 输出 |
| 模块 F | 调度因子计算、批量回测 |
| 模块 G | L3 撮合规则与 Paper 引擎共享；L5 绩效指标用于实盘监控 |

---

# 模块 C：量化结果可视化模块（策略筛选与排名实验台）

> 本模块是研究团队最高频使用的工具，目标是把"输入 × 策略"的所有实验结果统一收集、排名、对比、深挖，快速定位优质组合，并把候选策略推送到模块 G 进入 Paper trading。

## C.1 模块定位与核心抽象

```
Input Space × Strategy Space → Result Matrix → Ranking → Drill-down → Promote to Paper
```

每个"实验"是 `(Input, Strategy, Params, Period, Universe)` 五元组，统一格式的 `BacktestResult` 存入结果库供排名、对比、筛选。

**关键设计原则：**
- **输入与策略正交分开**：一个策略可套到 N 个输入，一个输入可跑 N 个策略
- **Universe 强制标注**：实验矩阵 cell 上必须可见 universe 标签（量化 P1-12 修复）
- **IS / OOS 强制分离**：UI 强制显示 OOS 指标，IS 用浅色弱化（量化 P0 强化）
- **可推送实盘**：每个 cell 有 "Promote to Paper Trading" 按钮 → 模块 G

## C.2 输入空间（Input Space）

| 类型 | 例子 | 用途 |
|---|---|---|
| **单标的** | `SH:600519` | 单股策略测试 |
| **标的 + 基准** | `SH:600519` vs `SH:000300` | 超额收益策略 |
| **同行业组（趋势套利）** | `[腾讯, 阿里, 美团, 京东]` | 行业内轮动 / 配对 |
| **跨市场同标的对** | `HK:09988` ↔ `US:BABA` | ADR 套利、领先指标 |
| **跨市场指数对** | `HK:HSI` → `US:NDX` | 隔夜情绪传导 |
| **股票篮子（自定义权重）** | `{600519: 0.3, 000858: 0.4, 600809: 0.3}` | 自定义组合回测 |
| **多空对组** | Long `A` / Short `B` | 价差套利 |
| **因子组合** | 选股因子 + 持仓数 + 调仓频率 | 因子策略 |
| **Universe** | `CSI300` / `MY_WATCHLIST_TECH` | 全 universe 选股策略 |

**权重模式：** 等权 / 自定义固定权重 / 市值加权 / 反波动率加权 / 风险平价 / HRP

**配置示例（YAML）：**
```yaml
input:
  type: cross_market_pair
  legs:
    - {symbol: HK:09988, role: leader, offset_hours: 0}
    - {symbol: US:BABA, role: follower, offset_hours: 12}
  alignment: trading_session
  base_currency: USD
  fx_handling: hedged    # hedged / unhedged / native
universe_id: CUSTOM_HK_US_TECH
```

## C.3 策略空间（Strategy Space）

**趋势 / 动量类**：双均线、突破、通道、跨标的动量轮动
**均值回归 / 配对类**：价差配对（协整 + Z-score）、行业内相对强弱回归、ADR 套利
**领先-滞后类（Lead-Lag）**：
- 港股开盘 → 美股当晚方向
- 美股隔夜 → A 股次日开盘
- 期货 → 现货
- 同一标的跨市场领先
**多因子选股**：单 / 多因子打分排序选股，调仓周期可设
**自定义策略**：用户传入 Python 函数 / DSL

## C.4 回测结果指标（每个 cell 必备）

按类别分组（UI 中可折叠）：

### 收益类
- 累计收益、年化收益、月度收益热力图
- 超额收益（相对基准）、Information Ratio

### 风险类
- 最大回撤 + 回撤持续天数
- 年化波动、下行波动
- VaR / CVaR / EVT 尾部估计

### 风险调整收益
- Sharpe / Sortino / Calmar / Omega
- **DSR / PSR / MinTRL**（必显示）

### 交易质量
- 胜率（按笔 / 按日 / 按月）
- 盈亏比 (Profit Factor)
- 平均持仓时长
- 总交易次数、换手率
- 平均单笔收益
- 流动性占用（持仓 vs ADV）

### 稳健性
- **样本内 vs 样本外指标对比（强制显示）**
- 参数敏感性评分（参数 ±10% 后指标变化）
- 不同市场状态下表现（牛 / 熊 / 震荡）
- **PBO 过拟合概率**（必显示）

### 成本敏感性
- 在不同滑点 / 手续费假设下的指标变化
- 不同冲击成本模型（L1/L2/L3）下的指标

### Universe 标签（强制）
- universe_id / universe_size / industry_version

## C.5 综合打分与排名系统（核心）

### C.5.1 加权综合得分（默认模式）

```
Score = Σ wi × normalize(metric_i)
```

预设档位（用户可改权重）：

| 模式 | Sharpe | 年化 | 最大回撤 | 胜率 | 稳健性 (PBO/DSR) |
|---|---|---|---|---|---|
| **保守型** | 0.30 | 0.15 | 0.30 | 0.10 | 0.15 |
| **平衡型** | 0.25 | 0.25 | 0.20 | 0.10 | 0.20 |
| **激进型** | 0.20 | 0.40 | 0.15 | 0.05 | 0.20 |
| **实盘候选** | 0.20 | 0.15 | 0.25 | 0.10 | 0.30 |
| **自定义** | slider 实时调整 |

归一化：Min-Max 或 Rank Percentile（前者受极值影响，后者更稳，推荐 Rank Percentile）。

### C.5.2 Pareto 前沿模式（专业模式）
只展示非劣解，常用维度对：`年化收益 vs 最大回撤` / `Sharpe vs 换手率` / `Sharpe vs PBO`

### C.5.3 过滤器（前置约束）
- 交易次数 < N
- 最大回撤 > X%
- **样本外 Sharpe 跌幅 > 50%**（疑似过拟合）
- **PBO > 0.5**（高过拟合风险）
- 单笔最大亏损 > X%
- **Universe 不匹配**（实验时的 universe 与当前候选 universe 不一致）

### C.5.4 排名场景

| 当变量是 | 排名展示 |
|---|---|
| 输入固定，策略变化 | "对这只股票最有效的策略 Top N" |
| 策略固定，输入变化 | "这个策略最有效的股票 Top N" |
| 都变化 | 二维矩阵 + 整体排名 |
| 输入是篮子 | 篮子表现 + 按贡献度排序成分股 |

## C.6 可视化界面布局

```
┌────────────────────────────────────────────────────────────┐
│  实验配置区（折叠）                                          │
│  Input | Strategy | Param Sweep | Period | Universe        │
├────────────────────────────────────────────────────────────┤
│  排名面板（30%）             │  详情面板（70%）              │
│  ┌──────────────────────┐  │  ┌────────────────────────┐ │
│  │  打分权重 sliders     │  │  │  净值曲线 + 基准对比    │ │
│  │  过滤器               │  │  ├────────────────────────┤ │
│  │  ──────────────       │  │  │  回撤曲线               │ │
│  │  Top N 排名表          │  │  ├────────────────────────┤ │
│  │  [策略 A] ★★★★☆ 95   │  │  │  月度收益热力图         │ │
│  │  ⚠ PBO=0.42          │  │  ├────────────────────────┤ │
│  │  [策略 B] ★★★★  87   │  │  │  交易记录 + K 线 marker │ │
│  │  Universe: CSI300    │  │  └────────────────────────┘ │
│  │  ...                  │  │  [→ Promote to Paper]       │
│  │  [Pareto 切换]        │  │  [对比模式：勾选多行叠加]    │
│  └──────────────────────┘  │                              │
└────────────────────────────────────────────────────────────┘
```

## C.7 关键专业视图

### 1. 实验矩阵热力图（Strategy × Input）
- X 轴：策略，Y 轴：输入
- 颜色：综合得分 / Sharpe / 年化 / PBO（可切换）
- 点击 cell 进详情
- **每个 cell 显示 universe 标签**

### 2. 跨市场领先性分析视图
- 时移相关性曲线：`corr(A_t, B_{t+k})` for k in [-10, 10]
- 显示最佳领先期 + 显著性 p 值（多重检验校正后）
- Granger 因果检验
- **滚动协整 + 半衰期估计 + 结构性断点检测**（Chow / Bai-Perron）（量化 P1-6 修复）
- 配对回测：以领先标的信号操作滞后标的

### 3. 鲁棒性诊断卡（避免过拟合的关键）
- 样本内 vs 样本外指标对比表
- 参数热力图（两个核心参数扫描）
- Bootstrap 置信区间
- 同策略在不同市场（A / HK / US）的表现对比
- **DSR / PSR / PBO 联合展示**

### 4. 篮子归因视图
- 篮子整体净值
- 每只成分股的收益贡献分解（Brinson 风格）
- 成分股权重漂移
- "如果去掉这只股票"的反事实分析

### 5. 交易细节视图
- 交易明细表
- 单笔收益分布（直方图 + 箱线图）
- 持仓时长分布
- 最大盈利 / 亏损前 10 笔标记在 K 线上

### 6. 风险分析卡
- 不同市场状态下表现（VIX / A 股波动率 / 趋势状态划分）
- 极端事件中表现（2015 / 2018 / 2020 / 2022）
- 月度 / 年度盈亏热力图

### 7. 跨市场分析增强视图（量化 P1-6 修复）
- **汇率风险**：港美股回 RMB 计价时的汇率波动叠加
- **隔夜跳空效应**：跨时区策略的开盘缺口分布
- **ADR 折溢价时序图**：港美同股价差，含融资约束、做空难度等结构性因素
- **stress periods 失效检测**：极端事件中跨市场领先性的失效情况
- **有效交易窗口**：港股 BABA 在哪些小时有足够流动性

### 8. 流动性诊断视图
- 持仓 vs ADV 占比时序
- 单股流动性轨迹（Amihud illiquidity ratio）
- 假设增 10x 资金后的可行性评估

### 9. 实盘候选筛选视图（新增）
- "Promote to Paper Trading" 工作流
- 与已上线策略的相关性检查（必须 < 阈值）
- Paper trading 期望表现 vs 实际表现追踪

## C.8 专业角度的关键增强

### P0 必做
- **结果可复现**：每个结果绑定 `策略代码 hash + 数据快照 ID + 参数 JSON`
- **样本内 / 样本外强制分离**：UI 强制显示 OOS 指标
- **过拟合告警**（基于 DSR / PBO / IS-OOS 差距）：在排名旁打红色 ⚠
- **Universe 标注与跨 Universe 告警**

### P1 强烈建议
- **批量参数扫描**：一键扫一组参数
- **回测耗时显示**
- **导出研究笔记**：选中策略 → 生成 markdown 报告（接模块 E）
- **收藏夹 / 标签**：`["待跟踪", "已淘汰", "实盘候选", "已上线"]`

### P2 进阶
- **策略相关性矩阵**：表现最好的 N 个策略的相关性，避免集合都是同一类
- **组合优化器衔接**：选中多个策略 → 一键求最优配置权重（调模块 B-L4）
- **"如果当时入场"**：选历史时间点，模拟按当时 Top 3 配置到今天的结果
- **A/B 测试框架**：策略改进版 vs 原版的统计显著性检验

## C.9 工程实现要点

| 关注点 | 方案 |
|---|---|
| 结果存储 | Parquet（BacktestResult 表）+ 单独存净值序列；DuckDB 查询 |
| 计算调度 | 通过模块 F 跑实验矩阵，支持并行 |
| 缓存 | `(input + strategy + params + universe + data_snapshot)` 哈希命中复用 |
| 实时更新 | WebSocket 推送，长任务用进度条 |
| 前端 | Dash + TVLC + Plotly + AG Grid（高级表格） |
| 排名计算 | 后端归一化打分，前端 slider 改权重时仅重排序不重算 |

## C.10 落地阶段

| 阶段 | 周期 | 交付 |
|---|---|---|
| **C.1 MVP** | M2 内 | 单 Input × 单 Strategy 跑回测 + 基础指标 |
| **C.2 实验矩阵** | M2 内 | 多 Input × 多 Strategy 网格 + 排名表 + 加权打分 |
| **C.3 详情面板** | M2 内 | 净值 / 回撤 / 月度热力图 / 交易记录 |
| **C.4 输入扩展** | M2 内 | 篮子 + 跨市场对 + 多空对 + Universe |
| **C.5 领先滞后** | M2 内 | 时移相关性 + Granger + 配对回测 |
| **C.6 鲁棒性** | M4 内 | IS/OOS + DSR/PBO + 参数扫描 + 过拟合告警 |
| **C.7 跨市场增强** | M4 内 | 汇率 / 隔夜跳空 / ADR / 滚动协整 |
| **C.8 Promote to Paper** | M3 内 | 实盘候选工作流，接模块 G |
| **C.9 AI 集成** | M4 内 | Agent 生成解读 + 自动研究笔记 |

## C.11 与其他模块依赖

| 依赖 | 用途 |
|---|---|
| 模块 0 | DataLoader / Watchlist / Universe |
| 模块 A | 净值 / K 线 / 热力图组件复用 |
| 模块 B-L3 | 跑回测（BacktestResult schema） |
| 模块 B-L5 | 绩效指标 / 归因 |
| 模块 B-L6 | 实验追踪 / 复现 |
| 模块 E | Agent 生成解读、自动选优 |
| 模块 F | 批量回测任务调度 |
| 模块 G | "Promote to Paper" 推送候选策略 |

## C.12 重点场景示例

### 场景 1：港股阿里 → 美股阿里
```yaml
input: cross_market_pair([HK:09988, US:BABA])
strategy: lead_lag_signal(leader=HK, follower=US, lookback=20)
```
系统自动算出最优领先小时数、显著性、近期失效检测、ADR 折溢价时序。

### 场景 2：港股早盘涨跌作为美股参考
```yaml
input: cross_market_indicator(leader=HK:HSI, target=US:SPX)
strategy: open_to_close_signal(threshold=±1%)
```
看不同港股开盘振幅下美股当晚走势的条件概率分布 + stress periods 失效检测。

### 场景 3：消费股篮子轮动
```yaml
input: basket([茅台, 五粮液, 泸州老窖], weight=equal)
strategy: relative_momentum_rotation(lookback=20, top_n=1)
```
评估：篮子整体净值 vs 等权持有 vs 单股最优。

---

# 模块 D：数据接入层

> **定位**：负责所有外部数据接入（拉取 / 清洗 / 标准化 / 增量更新 / 质量监控 / 版本化）。输出到模块 0 的存储约定。**唯一与外部数据源 / 券商行情通信的模块。**

## D.1 多数据源适配器

| 数据源 | 覆盖 | 用途 |
|---|---|---|
| **Tushare Pro** | A 股全市场、财务、基本面、龙虎榜、北向资金 | A 股主力 |
| **AKShare** | A 股 + 港股补充、宏观数据 | 备份 / 补充 |
| **Polygon.io** | 美股全市场 + 分钟级 + Reference data | 美股主力 |
| **Futu OpenAPI** | 港股实时 + 部分美股 | 港股 + 实盘行情 |
| **baostock** | A 股免费历史 | 备份 |
| **Alpha Vantage** | 美股 + 外汇 + 加密 | 补充 |

抽象接口：`DataSourceAdapter`（见模块 0）

**源切换策略**：主源失败 → 备份源 → 缓存数据 + 告警

## D.2 增量更新调度（聚焦关注池）

任务（由模块 F 调度）：
- **每日盘后**：全市场日线 + 财务公告
- **关注池**：当日分钟数据（每个市场盘后立即拉）
- **周末**：全市场分钟回填（如需）
- **月初**：universe 历史成分股快照
- **月初**：行业分类版本检查
- **季度**：财报集中披露期密集校验

## D.3 数据质量监控

每次更新后跑校验：
- 缺失检测（区分停牌、节假日 vs 真缺失）
- 异常价格（涨跌幅 > X%、价格为 0 / 负）
- 成交量为零异常
- 复权连续性（adj_factor 跳变）
- 财务数据完整性

告警三档：`INFO`（控制台）/ `WARN`（邮件）/ `ERROR`（邮件 + Telegram，阻断后续依赖任务）

## D.4 数据版本与快照（PIT 复现）

- 每次"实验"绑定 `data_snapshot_id`
- 快照不复制数据，只记录"截止某日的所有数据视图"
- 通过 metadata 表实现（查询时过滤 `record_date <= snapshot_date`）
- 长期实验定期物化快照（避免后续数据更新影响）

## D.5 实盘行情接入

- 关注池 Tick / 1m 实时（Futu / Polygon WebSocket）
- 写入 Redis 短期缓存 + 异步落 Parquet
- 模块 G 的实盘风控订阅 Redis 流
- **关键约束**：实盘行情通道与历史行情通道严格隔离，不可串扰

## D.6 落地阶段

| 阶段 | 周期 | 交付 |
|---|---|---|
| **D.1** | 3 天 | Tushare 适配器 + A 股日线 + Parquet 落盘 |
| **D.2** | 3 天 | 增量更新 + 数据质量基础校验 |
| **D.3** | 3 天 | AKShare 备份 + Polygon 美股 + Futu 港股 |
| **D.4** | 1 周 | 关注池分钟数据 + 多分辨率金字塔 |
| **D.5** | 4 天 | 数据快照机制 + 全量监控告警 |
| **D.6** | 1 周 | 实盘 WebSocket 行情接入（M3 配套） |

---

# 模块 E：AI Agent 层

> **定位**：用户用自然语言与整个工具交互；作为研究副驾驶（**不接管决策**）；自动化批量分析（异动归因、报告生成）。技术栈：Claude API + Tool Use + Prompt Caching。

## E.1 Agent 角色矩阵

| 角色 | 定位 | 模型 |
|---|---|---|
| **Copilot** | 副驾驶，自然语言查询 + 解读 | Claude Sonnet 4.6 |
| **Workflow Orchestrator** | 编排回测 / 因子测试 / 报告 | Claude Opus 4.7 |
| **Strategy Generator** | 生成候选因子 / 策略 | Claude Opus 4.7 |
| **Batch Analyzer** | 批量异动归因 / 全市场扫描 | Claude Haiku 4.5 |

**定位铁律：**
- 不让 LLM 直接给买卖信号
- **任何数值必须通过工具取得，禁止凭记忆回答**
- 关键决策保留人工审核

## E.2 工具集（Tool Use）

各模块向 Agent 暴露的 tools：

```
模块 0:
- get_series(symbol, start, end, frequency, adjust)
- get_universe(name, date)
- resolve_symbol(text)
- get_watchlist(name)

模块 B:
- run_factor_ic_test(factor_expr, universe, period)
- run_backtest(strategy_config, period)
- get_factor_value(factor_id, date_range)
- compute_performance_metrics(equity_curve)
- run_overfitting_diagnostic(experiment_id)

模块 C:
- query_strategy_rankings(filter, sort)
- get_backtest_detail(experiment_id)
- compare_strategies(experiment_ids)

模块 D:
- check_data_quality(symbol, date)
- get_news(symbol, date_range)
- get_announcements(symbol, date_range)
- get_corporate_actions(symbol, date_range)

模块 G:
- get_live_positions()
- get_paper_pnl(strategy_id)
- submit_paper_order(order_spec)        # 仅 Paper，实盘需人工确认
- get_risk_metrics()
```

## E.3 关键场景

1. **自然语言查询**："北向资金本周净流入前 10 的医药股"
2. **异动归因**：盘后扫描异动股 → 拉新闻 / 公告 / 同行业 / 资金流向 → 结构化归因
3. **因子生成**：用户描述 → 生成候选表达式 → 自动 IC 测试 → 解读
4. **报告生成**：月报 / 因子卡片 / 策略复盘 / 实盘日报
5. **风险预警**：实盘持仓的潜在风险（集中度、相关性、流动性）

## E.4 工程要点

**强制取数（防幻觉）**
- System prompt 明确："任何数值必须通过工具获取，禁止凭记忆回答"
- 输出 JSON Schema 约束
- 工具失败时不能编造数据

**Prompt Caching**
- 工具定义、指标说明、市场上下文做缓存前缀
- 节省 ≈ 90% input token

**可观测性**
- 每次调用记录 `input / output / tool_calls / latency / cost`
- 用 Langfuse 或自建 SQLite
- 异常调用告警

**沙箱执行**（代码生成场景）
- 用 RestrictedPython 限制
- 或单独 Docker 容器
- 不允许 Agent 直接执行未审核代码

**实盘安全**
- Agent 不可直接下实盘单
- 实盘相关 tool 只暴露查询，下单必须人工确认

## E.5 模型选择 + 成本预算

| 用途 | 模型 | 单次成本 |
|---|---|---|
| 日常 Copilot | Sonnet 4.6 | $0.05–0.2 |
| 复杂规划 | Opus 4.7 | $0.2–1 |
| 批量扫描 | Haiku 4.5 | $0.001–0.01 |

**预算**：个人用户月成本 < $50（Prompt Caching 优化后）

## E.6 落地阶段

| 阶段 | 周期 | 交付 |
|---|---|---|
| **E.1** | 1 周 | Anthropic SDK + 5 个核心 tool + 自然语言查询 demo |
| **E.2** | 1 周 | Prompt Caching + 调用日志 + 结构化输出 |
| **E.3** | 1 周 | 异动归因 batch agent + 报告生成 |
| **E.4** | 1 周 | 因子生成器 + 沙箱执行 |
| **E.5** | 持续 | 实盘风险预警 + 月报自动撰写 |

---

# 模块 F：任务调度与监控层

> **定位**：所有定时 / 触发式任务的执行与监控。个人版用 APScheduler，未来可升级 Airflow。

## F.1 任务清单

### 每日
- **06:30** 美股盘后日线 + 财务公告
- **09:00** A 股盘前数据准备（验证关注池可用）
- **09:15** A 股 / 港股开盘前实盘风控预备
- **15:00** A 股盘后日线
- **16:00** 港股盘后日线
- **17:00** 因子值滚动计算（关注池）
- **21:00** 美股盘前数据 + Paper trading 状态检查

### 每周
- **周末**：universe 历史成分股 monthly snapshot
- **周末**：全量数据质量审计
- **周末**：策略衰减检测报告

### 每月
- **月初**：因子库性能复盘
- **月初**：实盘月报

### 实盘场景（与模块 G 联动）
- **盘前 30 分钟**：数据准备 + 策略信号生成
- **盘中**：风控监控（订阅 Redis）+ 异常告警
- **盘后 1 小时**：对账 + 滑点反向校准 + 日报

## F.2 失败重试与告警

**重试策略**：指数退避，最多 3 次

**告警渠道**：
- `INFO`：控制台 + 日志
- `WARN`：邮件
- `ERROR`：邮件 + Telegram bot
- `CRITICAL`：邮件 + Telegram + 短信（实盘相关）

## F.3 调度器配置

```yaml
scheduler:
  type: apscheduler
  jobstore: sqlite:///scheduler.db
  jobs:
    - name: a_stock_daily_update
      cron: "0 15 * * MON-FRI"
      task: ingestion.update_daily
      args: {markets: [SH, SZ, BJ]}
      timeout: 1800
      retries: 3
    - name: watchlist_minute_update
      cron: "5 15,16,21 * * MON-FRI"
      task: ingestion.update_minute
      args: {watchlist: all}
```

## F.4 落地阶段

| 阶段 | 周期 | 交付 |
|---|---|---|
| **F.1** | 3 天 | APScheduler 框架 + 配置驱动 + 日志 |
| **F.2** | 4 天 | 全部基础任务接入 |
| **F.3** | 3 天 | 失败重试 + 告警渠道 |
| **F.4** | 1 周 | 实盘场景任务（盘前 / 盘中 / 盘后） |

---

# 模块 G：实盘衔接层

> **定位**：研究 → 回测 → Paper trading → 小资金实盘的完整链路。**所有实盘相关功能都在此模块，与研究模块严格隔离。**

## G.1 Paper Trading 模拟盘

**目的**：用实时行情但虚拟资金运行策略，验证回测假设。

**核心组件**：
- 实时行情订阅（模块 D 推送）
- 模拟撮合（与模块 B-L3 回测引擎共享核心，但用 live data）
- 虚拟账户 + PnL 追踪
- 信号执行延迟模拟（默认 100ms，可配置）

**数据存储**：`live/paper/{strategy_id}/{date}/{orders, fills, positions}.parquet`

## G.2 订单管理

```
研究侧策略产生 OrderIntent
  → 风控审核 (G.6)
  → Order Manager 转 LiveOrder
  → 路由到 Paper 或实盘券商
  → 接收 Fill 回报
  → 更新 Position
```

**券商接入**：
- A 股：QMT / Ptrade
- 港股：Futu OpenAPI
- 美股：Alpaca / IB

## G.3 实盘对账

**每日盘后**：
- 拉券商账单（持仓 + 成交）
- 对比内部 Position
- 偏差告警：自己记录的 vs 券商的不一致

**历史保留**：全部对账日志 ≥ 3 年

## G.4 滑点反向校准

用实盘成交价反过来校准回测的滑点假设：
- 收集 N 条实盘 trade
- 与同时点的 mid price / VWAP 对比
- 拟合滑点模型参数（更新模块 B-L3 的 Square-Root α）
- 每月更新回测默认滑点

## G.5 盘中风控

订阅 Redis 实时行情，触发式检查：

### 风控规则
- 单股最大仓位上限
- 组合最大回撤触发降仓
- 单股止损（绝对 / 跟踪）
- 组合最大日内回撤
- 杠杆上限（如有融资）
- 行业 / 风格集中度上限
- **与已上线策略的相关性上限**
- 流动性占用上限（持仓 vs ADV）

### 触发后行动
告警 / 拒绝新订单 / 强制平仓（按规则配置）

## G.6 策略衰减监控

**每日 / 周**：
- live Sharpe vs backtest Sharpe（60 日窗口）
- 差值显著（如 < backtest × 50%）触发告警
- 因子 IC 滚动监控（同步告警）
- 策略实际换手率 vs 回测换手率（差距大说明撮合假设有问题）

## G.7 落地阶段

| 阶段 | 周期 | 交付 |
|---|---|---|
| **G.1** | 1 周 | Paper trading 引擎 + 单策略 + 虚拟账户 |
| **G.2** | 1 周 | Order Manager + 风控基础规则 |
| **G.3** | 1 周 | 券商接入（先 Futu / Alpaca）+ 真实下单 |
| **G.4** | 4 天 | 对账 + 滑点反向校准 |
| **G.5** | 1 周 | 完整盘中风控 + 多策略 |
| **G.6** | 持续 | 策略衰减监控 + 实盘日报集成 |

## G.8 实盘安全准则

1. **任何实盘订单必先经 Paper 至少 1 周**
2. **实盘资金分批进入**：10% → 30% → 100%
3. **实盘必须有止损规则**
4. **每日实盘日报必看**
5. **策略 live Sharpe 显著低于 backtest 时降仓 / 停用**
6. **实盘失败回滚预案**：每个策略上线前写明出问题如何快速平仓
7. **API 密钥独立保管**：实盘 API key 单独 vault，不进 git
8. **小时级风控复核**：除了实时风控外，每小时跑一次完整风控状态检查

## G.9 与其他模块的对接

| 模块 | 对接点 |
|---|---|
| 模块 0 | 取价、关注池 |
| 模块 B-L3 | Paper 引擎共享撮合规则与成本模型 |
| 模块 B-L5 | 绩效指标用于实盘监控 |
| 模块 C | 接收 "Promote to Paper" 推送 |
| 模块 D | 实盘行情订阅、券商行情通道 |
| 模块 E | 实盘相关 tool 仅查询，下单需人工确认 |
| 模块 F | 盘前 / 盘中 / 盘后任务调度 |
