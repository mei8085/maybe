# 家庭整体财务数据导出链路分析报告

## 1. 概述

本报告分析 Maybe 应用中家庭整体财务数据的导出链路，涵盖从控制器层到模型层的数据聚合过程，重点阐述账户模型与汇率模型的协作方式，以及导出控制器的职责边界。

## 2. 导出链路整体架构

```
用户请求 → FamilyExportsController → FamilyDataExportJob → Family::DataExporter → 生成ZIP文件
```

### 2.1 核心组件职责划分

| 层级 | 组件 | 主要职责 |
|------|------|----------|
| 控制器层 | FamilyExportsController | HTTP请求处理、权限验证、任务触发 |
| 任务层 | FamilyDataExportJob | 异步处理、状态管理、错误处理 |
| 数据聚合层 | Family::DataExporter | 数据查询、格式转换、文件生成 |
| 模型层 | Account、ExchangeRate等 | 数据提供、货币处理 |

### 2.2 控制器、后台任务与导出器的时序边界

```
时间轴
  ↓
T0 ─ 用户点击导出按钮
  │
T1 ─ FamilyExportsController#create 开始
  │   ├── 验证管理员权限
  │   ├── 创建 FamilyExport 记录 (status: pending)
  │   └── FamilyDataExportJob.perform_later()
  │
T2 ─ 控制器返回 HTTP 响应给用户
  │   └── 重定向到设置页面，提示"导出已开始"
  │
T3 ─ 【ActiveJob 异步边界】任务被加入队列
  │
T4 ─ 队列调度器 picked up 任务并执行
  │
T5 ─ FamilyDataExportJob#perform 开始
  │   ├── family_export.update!(status: :processing)
  │   └── exporter = Family::DataExporter.new(family)
  │
T6 ─ Family::DataExporter#generate_export 执行
  │   ├── 1. accounts.csv 生成
  │   ├── 2. transactions.csv 生成
  │   ├── 3. trades.csv 生成
  │   ├── 4. categories.csv 生成
  │   └── 5. all.ndjson 生成
  │
T7 ─ 导出文件生成完成，ZIP 包准备就绪
  │   ├── family_export.export_file.attach()
  │   └── family_export.update!(status: :completed)
  │
T8 ─ 任务完成，用户可下载导出文件
```

**关键时序边界说明：**

| 时间点 | 边界类型 | 说明 |
|--------|----------|------|
| T2-T3 | **同步/异步边界** | 控制器在此处结束，用户获得立即响应；任务被 ActiveJob 加入队列 |
| T3-T4 | **任务调度边界** | 任务在队列中等待，取决于队列适配器实现和当前负载 |
| T5-T7 | **数据处理边界** | 数据库查询密集型操作，时间取决于数据量 |
| T7-T8 | **文件持久化边界** | Active Storage 附件上传，可能涉及外部存储服务 |

### 2.3 ActiveJob 环境适配差异

| 环境 | queue_adapter | 异步行为表现 |
|------|---------------|------------|
| **生产环境** | `:sidekiq` | 真正异步执行，任务入 Sidekiq 队列，需独立 Sidekiq 进程消费；支持重试、超时等高级特性 |
| **测试环境** | `:test` | 默认不立即执行，任务存入 `enqueued_jobs` 数组；需调用 `perform_enqueued_jobs` 才实际执行；便于断言队列状态 |
| **开发环境** | 默认（通常 :async） | 异步执行但使用线程池；无需独立进程，适合本地开发调试 |

**设计意图：**
- **请求快速返回**：避免 HTTP 超时
- **资源隔离**：导出任务不会阻塞 web 服务器请求处理
- **可重试性**：任务失败可独立重试，不影响用户体验
- **环境一致性**：通过 ActiveJob 抽象层，保证不同环境的 API 一致

## 3. 导出控制器职责边界分析

### 3.1 FamilyExportsController 核心职责

**文件位置：`app/controllers/family_exports_controller.rb`**

#### 3.1.1 权限控制
```ruby
before_action :require_admin
```
- 仅允许管理员用户发起导出操作
- 通过 `Current.user.admin?` 验证权限

#### 3.1.2 导出触发（create 动作）
```ruby
def create
  @export = Current.family.family_exports.create!
  FamilyDataExportJob.perform_later(@export)
  # 重定向响应...
end
```
**职责：**
- 创建 `FamilyExport` 持久化记录
- 触发 ActiveJob 异步任务（不阻塞，立即返回）
- 不参与实际数据处理逻辑

#### 3.1.3 导出历史查询（index 动作）
```ruby
def index
  @exports = Current.family.family_exports.ordered.limit(10)
end
```

#### 3.1.4 文件下载（download 动作）
```ruby
def download
  if @export.downloadable?
    redirect_to @export.export_file, allow_other_host: true
  end
end
```

### 3.2 控制器设计原则

1. **单一职责原则**：控制器仅负责 HTTP 层面的处理，不包含业务逻辑
2. **异步处理**：通过 ActiveJob 后台任务处理耗时操作，避免请求超时
3. **状态管理**：通过 FamilyExport 模型跟踪导出状态

## 4. 数据聚合核心：Family::DataExporter

**文件位置：`app/models/family/data_exporter.rb`**

### 4.1 导出文件结构

DataExporter 生成包含以下文件的 ZIP 包：

| 文件名 | 内容 |
|--------|------|
| accounts.csv | 账户基本信息 |
| transactions.csv | 交易记录 |
| trades.csv | 投资交易 |
| categories.csv | 分类信息 |
| all.ndjson | 完整数据的 NDJSON 格式 |

### 4.2 数据聚合流程

#### 4.2.1 账户数据导出
```ruby
def generate_accounts_csv
  @family.accounts.includes(:accountable).find_each do |account|
    csv << [
      account.id,
      account.name,
      account.accountable_type,
      account.subtype,
      account.balance.to_s,  # 原始金额
      account.currency,      # 货币单位
      account.created_at.iso8601
    ]
  end
end
```

**关键点：**
- 通过 `@family.accounts` 关联获取家庭所有账户
- 使用 `includes(:accountable)` 避免 N+1 查询
- 导出原始金额和货币，不做汇率转换

#### 4.2.2 交易数据导出
```ruby
def generate_transactions_csv
  @family.transactions
    .includes(:category, :tags, entry: :account)
    .find_each do |transaction|
      # 处理交易数据...
  end
end
```

## 5. 币种字段逐层传递细节与落点分析

### 5.1 币种字段的数据模型层来源

#### 5.1.1 账户层级的币种字段
```ruby
# app/models/account.rb
class Account < ApplicationRecord
  # 数据库字段: currency (string)
  # 存储格式: ISO 4217 标准代码 (如 "USD", "EUR", "CNY")
  validates :currency, presence: true
end
```

#### 5.1.2 交易层级的币种字段
```ruby
# app/models/entry.rb
class Entry < ApplicationRecord
  # 数据库字段: currency (string)
  # 每个 Entry 独立存储货币，可与 Account 货币不同
  validates :currency, presence: true
  
  belongs_to :account  # Account 有自己的 currency
end
```

**币种字段来源链：**
```
Account.currency ← 账户级别基准货币
    ↑
Entry.currency   ← 每笔交易独立货币（可不同）
    ↑
Transaction.entry.currency
Trade.entry.currency
Valuation.entry.currency
```

### 5.2 CSV 文件中的币种落点（列号从 0 开始）

#### 5.2.1 accounts.csv 币种字段
**表头：`["id", "name", "type", "subtype", "balance", "currency", "created_at"]`**

| 列索引 | 字段名 | 说明 | 数据来源 |
|--------|--------|------|----------|
| 0 | id | 账户ID | Account.id |
| 1 | name | 账户名称 | Account.name |
| 2 | type | 账户类型 | Account.accountable_type |
| 3 | subtype | 子类型 | Account.subtype |
| 4 | balance | 余额金额 | Account.balance |
| **5** | **currency** | **货币代码** | **Account.currency** |
| 6 | created_at | 创建时间 | Account.created_at |

```ruby
# 共 7 列，币种在第 5 列（索引从 0 开始）
csv << [
  account.id,              # 列 0
  account.name,            # 列 1
  account.accountable_type,# 列 2
  account.subtype,         # 列 3
  account.balance.to_s,    # 列 4: 原始金额数值
  account.currency,        # 列 5: 货币代码 (ISO 4217)
  account.created_at.iso8601 # 列 6
]
```

#### 5.2.2 transactions.csv 币种字段
**表头：`["date", "account_name", "amount", "name", "category", "tags", "notes", "currency"]`**

| 列索引 | 字段名 | 说明 | 数据来源 |
|--------|--------|------|----------|
| 0 | date | 交易日期 | Entry.date |
| 1 | account_name | 账户名称 | Account.name |
| 2 | amount | 交易金额 | Entry.amount |
| 3 | name | 交易名称 | Entry.name |
| 4 | category | 分类名称 | Category.name |
| 5 | tags | 标签 | Tag.name |
| 6 | notes | 备注 | Entry.notes |
| **7** | **currency** | **货币代码** | **Entry.currency** |

```ruby
# 共 8 列，币种在第 7 列（索引从 0 开始）
csv << [
  transaction.entry.date.iso8601,  # 列 0
  transaction.entry.account.name,  # 列 1
  transaction.entry.amount.to_s,   # 列 2
  transaction.entry.name,          # 列 3
  transaction.category&.name,      # 列 4
  transaction.tags.pluck(:name).join(","), # 列 5
  transaction.entry.notes,         # 列 6
  transaction.entry.currency       # 列 7: 交易级别的货币代码
]
```

#### 5.2.3 trades.csv 币种字段
**表头：`["date", "account_name", "ticker", "quantity", "price", "amount", "currency"]`**

| 列索引 | 字段名 | 说明 | 数据来源 |
|--------|--------|------|----------|
| 0 | date | 交易日期 | Entry.date |
| 1 | account_name | 账户名称 | Account.name |
| 2 | ticker | 证券代码 | Security.ticker |
| 3 | quantity | 数量 | Trade.qty |
| 4 | price | 单价 | Trade.price |
| 5 | amount | 总金额 | Entry.amount |
| **6** | **currency** | **货币代码** | **Trade.currency** |

```ruby
# 共 7 列，币种在第 6 列（索引从 0 开始）
csv << [
  trade.entry.date.iso8601,   # 列 0
  trade.entry.account.name,   # 列 1
  trade.security.ticker,      # 列 2
  trade.qty.to_s,             # 列 3
  trade.price.to_s,           # 列 4
  trade.entry.amount.to_s,    # 列 5
  trade.currency              # 列 6: 交易级别的货币代码
]
```

### 5.3 NDJSON 文件中的币种落点

#### 5.3.1 Account 对象在 all.ndjson 中的结构
```ruby
# 数据来源: Account.as_json
{
  type: "Account",
  data: {
    id: 123,
    family_id: 456,
    name: "Checking Account",
    balance: "10000.00",      # 金额数值
    currency: "USD",          # 货币代码 - 顶层字段
    cash_balance: "10000.00",
    cash_currency: "USD",     # 现金货币（可能不同）
    status: "active",
    accountable_type: "Depository",
    accountable: { ... }
  }
}
```

#### 5.3.2 Transaction 对象在 all.ndjson 中的结构
```ruby
# 数据来源: Entry.currency 直接嵌入
{
  type: "Transaction",
  data: {
    id: 789,
    entry_id: 101112,
    account_id: 123,
    date: "2024-01-15",
    amount: 50.25,            # 金额数值
    currency: "EUR",          # 货币代码 - 与 Account 可能不同
    name: "Grocery Store",
    notes: "Weekly shopping",
    category_id: 45,
    merchant_id: 67,
    tag_ids: [8, 9],
    kind: "expense",
    created_at: "2024-01-15T10:30:00Z",
    updated_at: "2024-01-15T10:30:00Z"
  }
}
```

#### 5.3.3 Trade 对象在 all.ndjson 中的结构
```ruby
{
  type: "Trade",
  data: {
    id: 131415,
    entry_id: 161718,
    account_id: 123,
    security_id: 192021,
    ticker: "AAPL",
    date: "2024-01-20",
    qty: 10,
    price: 180.50,
    amount: 1805.00,
    currency: "USD",          # 货币代码
    created_at: "...",
    updated_at: "..."
  }
}
```

#### 5.3.4 Valuation 对象在 all.ndjson 中的结构
```ruby
{
  type: "Valuation",
  data: {
    id: 222324,
    entry_id: 252627,
    account_id: 123,
    date: "2024-01-31",
    amount: 25000.00,
    currency: "CNY",          # 估值货币
    name: "Monthly Valuation",
    created_at: "...",
    updated_at: "..."
  }
}
```

### 5.4 币种字段传递完整链路图

```
数据库层
  │
  ├─ Account(id, name, balance, currency)
  │     ↓ (includes(:accountable) 预加载)
  │     ↓
  ├─ DataExporter.generate_accounts_csv
  │     │
  │     ├─→ accounts.csv: balance(列4), currency(列5)
  │     │
  │     └─→ all.ndjson: Account.data.balance, Account.data.currency
  │
  └─ Entry(id, account_id, amount, currency, entryable_type)
        │
        ├─→ belongs_to :account (关联 Account.currency)
        │    ↓ (可以不同!)
        │
        ├─ entryable_type = "Transaction"
        │    │
        │    ├─→ DataExporter.generate_transactions_csv
        │    │     ↓
        │    │     transactions.csv: currency(列7)
        │    │
        │    └─→ all.ndjson: Transaction.data.amount, Transaction.data.currency
        │
        ├─ entryable_type = "Trade"
        │    │
        │    ├─→ DataExporter.generate_trades_csv
        │    │     ↓
        │    │     trades.csv: currency(列6)
        │    │
        │    └─→ all.ndjson: Trade.data.amount, Trade.data.currency
        │
        └─ entryable_type = "Valuation"
             │
             └─→ all.ndjson: Valuation.data.amount, Valuation.data.currency
```

## 6. 账户模型与汇率模型的协作

### 6.1 Account 模型核心结构

**文件位置：`app/models/account.rb`**

```ruby
class Account < ApplicationRecord
  include Monetizable  # 货币处理
  
  belongs_to :family
  has_many :entries
  has_many :transactions, through: :entries
  
  monetize :balance, :cash_balance  # 金额字段
end
```

### 6.2 ExchangeRate 模型

**文件位置：`app/models/exchange_rate.rb`**

```ruby
class ExchangeRate < ApplicationRecord
  include Provided
  
  validates :from_currency, :to_currency, :date, :rate, presence: true
end
```

### 6.3 货币转换机制：Money 类

**文件位置：`lib/money.rb`**

#### 6.3.1 汇率查询与账户模型协作流程：
```ruby
def exchange_to(other_currency, date: Date.current, fallback_rate: nil)
  if iso_code == other_iso_code
    self
  else
    exchange_rate = store.find_or_fetch_rate(
      from: iso_code, 
      to: other_iso_code, 
      date: date
    )
    Money.new(amount * exchange_rate, other_iso_code)
  end
end
```

### 6.4 汇率模型三种调用路径详细分析

#### 6.4.1 调用入口：Money#exchange_to

```ruby
# lib/money.rb
def exchange_to(other_currency, date: Date.current, fallback_rate: nil)
  iso_code = currency.iso_code
  other_iso_code = Money::Currency.new(other_currency).iso_code

  if iso_code == other_iso_code
    self  # 路径A: 相同货币，直接返回
  else
    # 进入 ExchangeRate.find_or_fetch_rate
    exchange_rate = store.find_or_fetch_rate(
      from: iso_code, 
      to: other_iso_code, 
      date: date
    )&.rate || fallback_rate

    raise ConversionError unless exchange_rate
    
    Money.new(amount * exchange_rate, other_iso_code)
  end
end
```

#### 6.4.2 路径一：缓存命中（数据库有记录）

```
调用链:
Money.exchange_to("USD", date: "2024-01-15")
  ↓
ExchangeRate.find_or_fetch_rate(from: "EUR", to: "USD", date: "2024-01-15")
  ↓
find_by(from_currency: "EUR", to_currency: "USD", date: "2024-01-15")
  ↓
找到记录 → 返回 ExchangeRate 模型对象
  ↓
返回 .rate 字段值 (如 1.085)
  ↓
Money.new(amount * 1.085, "USD")
```

**代码可验证的特征：**
- 纯数据库 `SELECT` 查询，无网络 IO，无外部 API 调用
- 执行路径：`find_by` 单条查询 → 直接返回模型对象
- 返回类型：`ExchangeRate` ActiveRecord 模型对象

#### 6.4.3 路径二：缓存未命中（数据库无记录，但有 Provider）

**文件位置：`app/models/exchange_rate/provided.rb`**

```ruby
def find_or_fetch_rate(from:, to:, date: Date.current, cache: true)
  rate = find_by(from_currency: from, to_currency: to, date: date)
  return rate if rate.present?  # 命中则返回

  return nil unless provider.present?

  response = provider.fetch_exchange_rate(from: from, to: to, date: date)
  return nil unless response.success?

  rate = response.data  # Provider 返回的业务对象（非 ActiveRecord）
  
  # 关键：只有 cache=true 时才写入数据库，且返回值被忽略
  ExchangeRate.find_or_create_by!(
    from_currency: rate.from,
    to_currency: rate.to,
    date: rate.date,
    rate: rate.rate
  ) if cache
  
  rate  # 无论 cache 是 true/false，始终返回 Provider 的业务对象
end
```

```
调用链:
Money.exchange_to("USD", date: "2024-01-15")
  ↓
ExchangeRate.find_or_fetch_rate(...)
  ↓
find_by(...) → nil
  ↓
provider.present? → true (如 Synth provider)
  ↓
provider.fetch_exchange_rate(from: "EUR", to: "USD", date: "2024-01-15")
  ↓
外部 HTTP 网络请求
  ↓
response.success? → true
  ↓
rate = response.data  # Provider 业务对象，含 from/to/date/rate
  ↓
cache = true?
  ├─ 是 → ExchangeRate.find_or_create_by!(...) 写入数据库（返回值被丢弃）
  └─ 否 → 不做任何数据库操作
  ↓
始终返回 rate (Provider 业务对象)
  ↓
Money.new(amount * rate.rate, "USD")
```

**关键事实说明：**
1. **返回值类型差异**：缓存命中时返回 ExchangeRate 模型对象，缓存未命中时返回 Provider 定义的业务对象，二者类型不一致但都可通过 `.rate` 读取汇率值

2. **find_or_create_by! 的定位与局限**：
   - 仅作为**可选落库**机制：只有 `cache: true` 时才执行，且返回值被完全丢弃，不影响最终返回值
   - **不能天然保证幂等性**：数据库唯一索引才是防重复的最终屏障，高并发场景仍可能触发唯一性约束异常

3. **cache 参数的精确影响**：
   - `cache: true`（默认）：调用外部 API 后尝试写入数据库，后续相同条件的查询将命中路径一
   - `cache: false`：仅调用外部 API，不写入数据库，每次相同条件调用都会重新触发网络请求

4. **同日期币种记录已存在但 rate 不同时的行为**：
   - 数据库层面：`[from_currency, to_currency, date]` 组合唯一索引 + rate 字段无约束
   - 落库行为：数据库已存在同组合记录时，`find_or_create_by!` 的 `find_by` 子句命中，**直接丢弃 Provider 返回的新 rate 值**，以数据库已有记录为准（不会触发 update 操作）
   - 实际效果：汇率值一旦写入数据库不会被自动修正，Provider 返回值变化对已缓存的日期无影响
   - 调用方结果：汇率已落库时，后续相同查询返回的是数据库已存储的 rate 值而非 Provider 最新返回值

#### 6.4.4 路径三：无 Provider 配置

```
调用链:
Money.exchange_to("USD", date: "2024-01-15")
  ↓
ExchangeRate.find_or_fetch_rate(...)
  ↓
find_by(...) → nil
  ↓
provider.present? → false (自托管环境常见)
  ↓
返回 nil
  ↓
检查 fallback_rate 参数
  │
  ├─ 有 fallback_rate → 使用备用汇率
  │
  └─ 无 fallback_rate → raise Money::ConversionError
```

**Provider 检查逻辑：**
```ruby
# app/models/exchange_rate/provided.rb
def provider
  registry = Provider::Registry.for_concept(:exchange_rates)
  registry.get_provider(:synth)  # 可能返回 nil
end
```

**代码可验证的特征：**
- 自部署环境通常不配置外部汇率 Provider
- 执行路径：find_by 查询 nil → provider 检查 nil → 返回 nil
- 完全离线操作，无网络 IO
- 最终结果取决于调用方是否提供 `fallback_rate`

### 6.5 为什么当前导出流程不会触发汇率转换

#### 6.5.1 根本原因：导出器只读取原始字段，不调用转换方法

**CSV 导出代码分析：**
```ruby
# app/models/family/data_exporter.rb

# 账户导出 - 直接读取数据库字段
account.balance.to_s    # 原始 BigDecimal
account.currency        # 原始字符串

# 交易导出 - 直接读取 Entry 的 currency 字段
transaction.entry.amount.to_s   # 原始 BigDecimal
transaction.entry.currency      # 原始字符串

# 投资交易导出 - 同上
trade.entry.amount.to_s   # 原始 BigDecimal
trade.currency            # 原始字符串
```

**NDJSON 导出代码分析：**
```ruby
# Account - 简单 as_json，没有转换
account.as_json(include: { accountable: {} })

# Transaction - 手动构建，只复制原始字段
data: {
  amount: transaction.entry.amount,    # BigDecimal
  currency: transaction.entry.currency,# 原封不动
  ...
}

# Trade - 同上，原始值直接嵌入
amount: trade.entry.amount,
currency: trade.currency,
```

#### 6.5.2 关键证据：导出器中无 exchange_to 调用

在 `Family::DataExporter` 的完整实现中，**完全没有**以下调用模式：
```ruby
# 不存在于代码中！
account.balance_money.exchange_to("USD", date: ...)
transaction.entry.amount_money.exchange_to(...)
```

#### 6.5.3 设计意图的三层解释

**第一层：数据保真**
- 货币转换是有损操作，浮点运算存在精度损失
- 例如：100 EUR × 1.085 = 108.5 USD，但反向转换可能得到 99.999... EUR
- 保留原始数据让下游系统决定转换策略

**第二层：性能考量（代码可验证）**
- 触发汇率转换意味着对每条记录执行 `exchange_to` 调用，其路径含：
  - 至少一次数据库 `SELECT` 查询
  - 缓存未命中时触发外部 HTTP 网络请求
- 批量导出时，汇率查询的 N 次独立 IO 开销远大于原始字段直接读取
- 当前设计将导出与汇率转换解耦，避免导出流程依赖外部服务可用性

**第三层：架构解耦**
- 汇率转换属于"数据消费"逻辑，不是"数据导出"逻辑
- 导出器的单一职责：提取并打包原始数据
- 转换逻辑由报表系统、BI 工具等下游处理

#### 6.5.4 反例：哪些场景会触发汇率转换

对比以下会触发汇率转换的场景：
```ruby
# app/models/balance_sheet/account_totals.rb
# 资产负债表汇总 - 统一货币显示
def total_balance
  accounts.sum do |account|
    account.balance_money.exchange_to(family.currency, date: Date.current)
  end
end

# app/models/income_statement/family_stats.rb
# 收入支出统计 - 需要统一货币
def total_income
  transactions.income.sum do |t|
    t.entry.amount_money.exchange_to(...)
  end
end
```

**关键区别：**
- 报表展示 → 需要统一货币 → 触发汇率转换
- 数据导出 → 需要原始保真 → 不触发转换

### 6.6 导出过程中的汇率使用策略

**重要发现：** 在当前的导出实现中，**导出数据保留原始货币，不进行汇率转换**。

#### 6.6.1 不做转换的原因：
1. **数据保真**：保留原始货币和金额，避免转换精度损失
2. **灵活性**：下游系统可以自行处理转换
3. **性能**：避免大量汇率查询拖慢导出速度
4. **复杂度**：无需处理汇率服务不可用、 fallback 策略等边界情况

#### 6.6.2 汇率模型在导出中的潜在应用场景：

虽然当前导出中没有直接使用汇率转换，但在以下场景中可能需要：

1. **合并报表生成**：
   - 当需要生成统一货币的汇总数据
   - 资产负债表
   - 收入支出汇总

2. **多货币账户的统一显示**：
   - Family 模型中的 `requires_data_provider?` 方法检测多货币账户
   - 多货币账户需要汇率数据

```ruby
# 多货币检测：
def requires_data_provider?
  # 有非家庭货币的账户需要汇率
  return true if accounts.where.not(currency: self.currency).any?
  
  # 条目有多种货币需要汇率
  uniq_currencies = entries.pluck(:currency).uniq
end
```

## 7. 数据聚合的关联关系

### 7.1 Family 模型作为聚合根

**文件位置：`app/models/family.rb`**

```ruby
class Family < ApplicationRecord
  has_many :accounts
  has_many :entries, through: :accounts
  has_many :transactions, through: :accounts
  has_many :trades, through: :accounts
  has_many :categories
  has_many :tags
end
```

### 7.2 数据聚合的关联链

```
Family
  ↳ Account
    ↳ Entry
      ↳ Transaction（交易）
      ↳ Trade（投资交易）
      ↳ Valuation（估值）
    ↳ Holding（持仓）
  ↳ Category（分类）
  ↳ Tag（标签）
```

## 8. 导出流程的状态管理

### 8.1 FamilyExport 状态机

**文件位置：`app/models/family_export.rb`**

```ruby
enum :status, {
  pending: "pending",
  processing: "processing",
  completed: "completed",
  failed: "failed"
}, default: :pending
```

### 8.2 状态流转

```
pending → processing → completed
            ↓
          failed
```

## 9. 关键设计决策分析

### 9.1 优点

1. **关注点分离**：控制器、任务、导出器各司其职
2. **异步处理**：通过 ActiveJob 抽象层，支持多种队列后端
3. **数据完整性**：NDJSON 格式包含完整数据关系
4. **批量查询**：使用 find_each 避免内存溢出
5. **Eager Loading**：includes 预加载避免 N+1 查询
6. **数据保真**：不进行汇率转换，保留原始货币信息

### 9.2 潜在优化点

1. **汇率数据导出**：当前未包含汇率数据，多货币场景下游使用不便
   - 建议：可增加 exchange_rates.csv 导出相关日期的汇率
2. **增量导出**：当前全量导出，大数据量场景效率待提升
3. **导出进度**：缺乏细粒度进度反馈
4. **币种字段说明**：导出文件缺少 schema 说明文档

## 10. 总结

### 10.1 控制器职责边界

- ✅ HTTP 请求处理
- ✅ 权限验证
- ✅ ActiveJob 异步任务触发
- ✅ 下载重定向
- ❌ 不处理业务逻辑
- ❌ 不直接操作数据聚合

### 10.2 币种字段落点速查表（0 基索引）

| 文件格式 | 数据类型 | 字段/列名 | 位置索引 | 数据来源 |
|---------|----------|-----------|----------|----------|
| CSV | Account | balance | 列 4 | Account.balance |
| CSV | Account | currency | 列 5 | Account.currency |
| CSV | Transaction | currency | 列 7 | Entry.currency |
| CSV | Trade | currency | 列 6 | Trade.currency |
| NDJSON | Account | data.balance | - | Account.balance |
| NDJSON | Account | data.currency | - | Account.currency |
| NDJSON | Transaction | data.amount | - | Entry.amount |
| NDJSON | Transaction | data.currency | - | Entry.currency |
| NDJSON | Trade | data.amount | - | Entry.amount |
| NDJSON | Trade | data.currency | - | Trade.currency |
| NDJSON | Valuation | data.amount | - | Entry.amount |
| NDJSON | Valuation | data.currency | - | Entry.currency |

### 10.3 汇率调用路径总结

| 场景 | 数据库查询操作 | 网络IO | 返回值类型 | 代码可验证的性能定性描述 | cache 参数影响 |
|------|---------------|---------|-----------|---------------------|----------------|
| 缓存命中 | ✅ 1次 `find_by` SELECT | ❌ 无 | `ExchangeRate` ActiveRecord 模型对象 | 本地数据库查询，仅单次 IO | 无影响 |
| 缓存未命中 + 有Provider | ✅ 1次 `find_by` SELECT + (cache=true时追加1次 `find_or_create_by!`) | ✅ 外部 HTTP API 请求 | Provider 业务对象 | 含跨网络远程调用，IO 开销大于纯数据库操作 | cache=true 尝试落库（返回值丢弃），false 不操作数据库 |
| 无 Provider | ✅ 1次 `find_by` SELECT | ❌ 无 | nil | 本地数据库查询，无外部依赖 | 无影响 |

**关键修正与证据补充：**
1. 缓存未命中时，`find_or_create_by!` 的返回值被完全丢弃，最终始终返回 Provider 业务对象
2. `cache: false` 时完全跳过数据库写入，每次相同条件调用都会重新触发网络请求
3. `find_or_create_by!` 仅为可选落库机制，并发场景仍可能触发唯一性约束异常
4. 数据库已存在同日期同币种记录时，新 rate 值被静默丢弃，不会触发更新

### 10.4 账户与汇率模型协作

| 组件 | 职责 | 在导出中的作用 |
|------|------|-----------------|
| Account | 账户数据管理 | 提供原始余额和货币信息 |
| ExchangeRate | 汇率存储与获取 | 未直接使用，供下游消费方转换 |
| Money | 货币转换逻辑 | 封装汇率查询接口 |
| Family::DataExporter | 数据聚合 | 协调各模型数据，不触发转换 |

### 10.5 导出链路的核心设计哲学

**"原始数据导出，转换由下游处理"**

这一设计决策确保了：
- 数据的原始性和完整性
- 导出性能不依赖外部服务
- 系统解耦，转换逻辑自由下放
- 下游系统灵活性最大化
