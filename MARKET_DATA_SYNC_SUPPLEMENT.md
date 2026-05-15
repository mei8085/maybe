# 市场数据同步机制补充分析报告

## 一、持仓数量从外部源进入账本的完整链路

### 1.1 整体架构概览

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                           外部数据源 (Plaid)                                     │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐             │
│  │  Transactions   │    │   Positions     │    │   Holdings      │             │
│  │  (交易记录)      │    │   (持仓快照)     │    │   (持仓明细)     │             │
│  └────────┬────────┘    └────────┬────────┘    └────────┬────────┘             │
└───────────┼──────────────────────┼──────────────────────┼──────────────────────┘
            ↓                      ↓                      ↓
┌──────────────────────────────────────────────────────────────────────────────────┐
│                          Plaid 数据导入层                                         │
│  ┌──────────────────────────────────────────────────────────────────────────┐    │
│  │  PlaidItem::Importer                                                    │    │
│  │    → fetch_and_import_accounts_data()                                   │    │
│  │      → PlaidAccount::Importer.import()                                  │    │
│  │        → import_investments() → upsert_plaid_investments_snapshot!()    │    │
│  └──────────────────────────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────────────────────┘
                                      ↓
┌──────────────────────────────────────────────────────────────────────────────────┐
│                          账户同步层 (Account::Syncer)                            │
│  ┌──────────────────────────────────────────────────────────────────────────┐    │
│  │  perform_sync(sync)                                                      │    │
│  │    1. import_market_data()  → 同步汇率 + 行情价格                         │    │
│  │    2. materialize_balances() → 计算持仓和余额                            │    │
│  └──────────────────────────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────────────────────┘
                                      ↓
┌──────────────────────────────────────────────────────────────────────────────────┐
│                        持仓计算层 (Holding::Materializer)                        │
│  ┌──────────────────────────────────────────────────────────────────────────┐    │
│  │  materialize_holdings()                                                  │    │
│  │    → calculate_holdings()                                                │    │
│  │      → Holding::ForwardCalculator / ReverseCalculator                    │    │
│  │        → transform_portfolio() - 根据交易更新持仓数量                     │    │
│  │        → build_holdings() - 结合价格计算持仓金额                          │    │
│  │    → persist_holdings() → upsert_all()                                   │    │
│  └──────────────────────────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────────────────────┘
                                      ↓
┌──────────────────────────────────────────────────────────────────────────────────┐
│                           数据库持久化                                           │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐                      │
│  │ trades  │    │ holdings│    │ balances│    │ prices  │                      │
│  │ (交易)   │    │ (持仓)   │    │ (余额)   │    │ (行情)   │                      │
│  └─────────┘    └─────────┘    └─────────┘    └─────────┘                      │
└──────────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 触发时序详解

#### 阶段一：Plaid 数据拉取

**触发条件**：定时同步任务或手动触发

```ruby
# app/models/plaid_item/syncer.rb
def perform_sync(sync)
plaid_item.import_latest_plaid_data  # 从Plaid获取最新数据
plaid_item.process_accounts           # 处理账户数据
plaid_item.schedule_account_syncs(    # 触发账户同步
    parent_sync: sync,
    window_start_date: sync.window_start_date,
    window_end_date: sync.window_end_date
)
end
```

**数据流转**：
1. `PlaidItem::Importer.import()` → `fetch_and_import_accounts_data()`
2. 通过 `PlaidItem::AccountsSnapshot` 获取账户快照数据
3. `PlaidAccount::Importer.import()` → `import_investments()`
4. 调用 `upsert_plaid_investments_snapshot!()` 保存持仓快照

#### 阶段二：账户同步触发

```ruby
# app/models/account/syncer.rb
def perform_sync(sync)
import_market_data      # 先同步市场数据（汇率 + 行情）
materialize_balances    # 再计算余额（依赖市场数据）
end
```

**关键设计**：市场数据同步在持仓计算**之前**执行，确保价格数据就绪。

#### 阶段三：持仓数量计算

**Forward 模式**（新建账户或首次同步）：

```ruby
# app/models/holding/forward_calculator.rb
def calculate
current_portfolio = generate_starting_portfolio  # 空投资组合 {security_id => 0}

account.start_date.upto(Date.current).each do |date|
    trades = portfolio_cache.get_trades(date: date)  # 获取当日交易
    next_portfolio = transform_portfolio(current_portfolio, trades)  # 更新持仓数量
    holdings += build_holdings(next_portfolio, date)  # 结合价格计算金额
    current_portfolio = next_portfolio
end

Holding.gapfill(holdings)  # 填充日期间隙
end
```

**Reverse 模式**（已有持仓的账户同步）：

```ruby
# app/models/holding/reverse_calculator.rb
def calculate_holdings
current_portfolio = portfolio_snapshot.to_h  # 从最新持仓快照开始

Date.current.downto(account.start_date).each do |date|
    today_trades = portfolio_cache.get_trades(date: date)
    previous_portfolio = transform_portfolio(current_portfolio, today_trades, direction: :reverse)
    holdings += build_holdings(current_portfolio, date)
    current_portfolio = previous_portfolio
end

holdings
end
```

### 1.3 与行情、汇率的衔接机制

#### 价格优先级策略

```ruby
# app/models/holding/portfolio_cache.rb
def load_prices
securities.each do |security|
    # 优先级1: 数据库价格（从Provider同步）
    db_prices = security.prices.where(date: account.start_date..Date.current).map do |price|
    PriceWithPriority.new(price: price, priority: 1, source: "db")
    end

    # 优先级2: 交易价格
    trade_prices = trades.select { |t| t.entryable.security_id == security.id }.map do |trade|
    PriceWithPriority.new(price: Security::Price.new(...), priority: 2, source: "trade")
    end

    # 优先级3: 持仓价格（仅Reverse模式）
    holding_prices = use_holdings ? holdings.select {...}.map {...} : []

    @security_cache[security.id] = {
    security: security,
    prices: db_prices + trade_prices + holding_prices
    }
end
end
```

**价格获取逻辑**：

```ruby
def get_price(security_id, date, source: nil)
security = @security_cache[security_id]

if source.present?
    price = security[:prices].select { |p| p.price.date == date && p.source == source }.min_by(&:priority)&.price
else
    price = security[:prices].select { |p| p.price.date == date }.min_by(&:priority)&.price
end

return nil unless price

# 汇率转换
price_money = Money.new(price.price, price.currency)
converted_amount = price_money.exchange_to(account.currency, fallback_rate: 1).amount

Security::Price.new(security_id: security_id, date: date, price: converted_amount, currency: account.currency)
end
```

### 1.4 完整数据流程图

```
外部数据源 (Plaid)
        │
        ▼
┌──────────────────────────────────────┐
│ 1. PlaidItem::Syncer.perform_sync() │
│    → import_latest_plaid_data()     │
│    → schedule_account_syncs()       │
└──────────────────────────────────────┘
        │
        ▼
┌──────────────────────────────────────┐
│ 2. Account::Syncer.perform_sync()   │
│    ├─→ import_market_data()         │
│    │     ├─ ExchangeRate.import()   │
│    │     └─ Security.import_prices()│
│    │                                │
│    └─→ materialize_balances()       │
└──────────────────────────────────────┘
        │
        ▼
┌──────────────────────────────────────┐
│ 3. Balance::Materializer            │
│    → materialize_holdings()         │
└──────────────────────────────────────┘
        │
        ▼
┌──────────────────────────────────────┐
│ 4. Holding::Calculator              │
│    ├─→ transform_portfolio()        │
│    │     交易 → 持仓数量变化          │
│    │                                │
│    ├─→ portfolio_cache.get_price()  │
│    │     价格查找 + 汇率转换          │
│    │                                │
│    └─→ build_holdings()             │
│          数量 × 价格 = 金额           │
└──────────────────────────────────────┘
        │
        ▼
┌──────────────────────────────────────┐
│ 5. Holding.gapfill()                │
│    LOCF填充日期间隙                  │
└──────────────────────────────────────┘
        │
        ▼
┌──────────────────────────────────────┐
│ 6. holdings.upsert_all()            │
│    持久化到数据库                    │
└──────────────────────────────────────┘
```

---

## 二、定时任务与时区分析

### 2.1 当前定时任务配置

```yaml
# config/schedule.yml
import_market_data:
cron: "0 22 * * 1-5"  # 美国东部时间下午5点
class: "ImportMarketDataJob"
queue: "scheduled"
description: "Imports market data daily at 5:00 PM EST (1 hour after market close)"
```

### 2.2 夏令时与冬令时影响分析

#### 时区转换机制

```ruby
# app/models/market_data_importer.rb
def end_date
Date.current.in_time_zone("America/New_York").to_date
end
```

**关键设计**：使用 `America/New_York` 时区而非固定 UTC 偏移，自动处理 DST。

#### 执行时点对照表

| 时段 | 美国东部时间 | UTC时间 | 与美股收盘关系 |
|------|-------------|--------|---------------|
| **冬令时 (EST)** | 下午5:00 | 22:00 | 收盘后1小时 |
| **夏令时 (EDT)** | 下午5:00 | 21:00 | 收盘后1小时 |

**美股收盘时间**：美国东部时间下午4:00（常规交易时段）

#### Rails 时区配置检查

```ruby
# config/application.rb (通常)
config.time_zone = 'America/New_York'
config.active_record.default_timezone = :utc
```

**关键要点**：
1. Rails 使用 `config.time_zone` 作为应用时区
2. 数据库存储使用 UTC
3. `Date.current` 返回应用时区的当前日期
4. `in_time_zone("America/New_York")` 确保正确转换

### 2.3 执行时序保障

#### 避免未来日期查询

```ruby
# app/models/security/price/importer.rb
def normalize_end_date(requested_end_date)
today_est = Date.current.in_time_zone("America/New_York").to_date
[ requested_end_date, today_est ].min
end
```

**设计意图**：防止因时区差异导致查询未来日期的数据。

#### 日期范围计算

```ruby
# app/models/market_data_importer.rb
def default_start_date
SNAPSHOT_DAYS.days.ago.to_date  # 31.days.ago
end

def end_date
Date.current.in_time_zone("America/New_York").to_date
end
```

### 2.4 定时任务框架分析

**Sidekiq Cron** 默认使用服务器本地时区执行任务。若服务器位于非 EST/EDT 时区，需要额外配置：

```ruby
# config/initializers/sidekiq.rb
Sidekiq::Cron::Job.load_from_hash YAML.load_file(File.expand_path("schedule.yml", __dir__))

# 确保使用正确时区
Sidekiq::Cron::Job.timezone = "America/New_York"
```

**当前配置检查**：需要确认 `schedule.yml` 中 cron 表达式是否在正确时区执行。

---

## 三、汇率缺失回退机制分析

### 3.1 回退机制实现

#### 汇率转换回退

```ruby
# app/models/holding/portfolio_cache.rb
def get_price(security_id, date, source: nil)
# ... 获取价格 ...

price_money = Money.new(price.price, price.currency)
converted_amount = price_money.exchange_to(
    account.currency,
    fallback_rate: 1  # 汇率缺失时使用1:1
).amount

Security::Price.new(
    security_id: security_id,
    date: date,
    price: converted_amount,
    currency: account.currency
)
end
```

#### 余额计算回退

```ruby
# app/models/balance/sync_cache.rb
def converted_entries
@converted_entries ||= account.entries.order(:date).to_a.map do |e|
    converted_entry = e.dup
    converted_entry.amount = converted_entry.amount_money.exchange_to(
    account.currency,
    date: e.date,
    fallback_rate: 1
    ).amount
    converted_entry.currency = account.currency
    converted_entry
end
end
```

### 3.2 对账本金额准确性的影响

#### 影响场景分析

| 场景 | 条件 | 影响 |
|------|------|------|
| **无影响** | 交易货币 = 账户货币 | 无需汇率转换 |
| **无影响** | 汇率存在于数据库 | 使用真实汇率 |
| **有影响** | 汇率缺失 + 货币不同 | 使用1:1汇率计算 |

#### 误差计算示例

**示例1：美元账户持有欧元资产**
```
真实汇率: 1 EUR = 1.08 USD
持仓数量: 100 shares
价格: 50 EUR/share
真实金额: 100 × 50 × 1.08 = 5,400 USD
回退金额: 100 × 50 × 1 = 5,000 USD
误差: -400 USD (-7.4%)
```

**示例2：欧元账户持有美元资产**
```
真实汇率: 1 USD = 0.9259 EUR
持仓数量: 100 shares
价格: 50 USD/share
真实金额: 100 × 50 × 0.9259 = 4,629.5 EUR
回退金额: 100 × 50 × 1 = 5,000 EUR
误差: +370.5 EUR (+8.0%)
```

### 3.3 现有告警机制分析

#### 当前告警实现

```ruby
# app/models/provider/synth.rb
def fetch_exchange_rates(from:, to:, start_date:, end_date:)
data = paginate("#{base_url}/rates/historical-range", ...)

data.paginated.map do |rate|
    date = rate.dig("date")
    rate = rate.dig("rates", to)

    if date.nil? || rate.nil?
    Rails.logger.warn("#{self.class.name} returned invalid rate data for pair from: #{from} to: #{to} on: #{date}")
    Sentry.capture_exception(InvalidExchangeRateError.new(...), level: :warning) do |scope|
        scope.set_context("rate", { from: from, to: to, date: date })
    end
    next
    end

    Rate.new(date: date.to_date, from:, to:, rate:)
end.compact
end
```

#### 告警覆盖范围

| 告警类型 | 是否覆盖 | 告警级别 |
|---------|---------|---------|
| Provider 返回无效数据 | ✅ | warning |
| Provider API 调用失败 | ✅ | warning/error |
| 汇率缺失导致使用 fallback | ❌ | - |
| 余额计算异常 | ❌ | - |

### 3.4 兜底机制评估

#### 现有兜底策略

```ruby
# exchange_to 方法中
fallback_rate: 1  # 硬编码回退到1:1
```

**优点**：
- 保证计算能继续，不会因缺失汇率中断
- 简单直接，易于理解

**缺点**：
- 缺乏监控，无法得知何时使用了回退
- 1:1 回退在汇率波动大时误差显著
- 无渐进式降级策略

### 3.5 风险评估矩阵

| 风险等级 | 场景 | 影响程度 | 现有控制 |
|---------|------|---------|---------|
| **高** | 主要货币对汇率缺失 | 账户余额计算错误 | 1:1回退 |
| **中** | 次要货币对汇率缺失 | 部分持仓金额错误 | 1:1回退 |
| **低** | 历史汇率缺失 | 历史余额显示错误 | LOCF填充 |

### 3.6 改进建议

#### 建议1：增加汇率缺失告警

```ruby
# 在 exchange_to 调用处增加监控
def get_price(security_id, date, source: nil)
# ...
converted_amount = price_money.exchange_to(account.currency, fallback_rate: 1).amount

# 检查是否使用了 fallback
if price.currency != account.currency && !ExchangeRate.exists?(from_currency: price.currency, to_currency: account.currency, date: date)
    Rails.logger.warn("Fallback rate used for security #{security_id} on #{date}: #{price.currency} -> #{account.currency}")
    Sentry.capture_message("Exchange rate fallback used", level: :warning) do |scope|
    scope.set_context("exchange_rate_fallback", {
        security_id: security_id,
        date: date,
        from_currency: price.currency,
        to_currency: account.currency
    })
    end
end
# ...
end
```

#### 建议2：使用最近可用汇率作为 fallback

```ruby
def get_price(security_id, date, source: nil)
# ...
# 先尝试获取精确日期的汇率
rate = ExchangeRate.find_by(from_currency: price.currency, to_currency: account.currency, date: date)

if rate
    fallback_rate = rate.rate
else
    # 获取最近的汇率作为 fallback
    nearest_rate = ExchangeRate.where(
    from_currency: price.currency,
    to_currency: account.currency
    ).order("ABS(date - '#{date}')").first

    fallback_rate = nearest_rate&.rate || 1

    if fallback_rate == 1
    # 记录严重告警
    Sentry.capture_exception(MissingExchangeRateError.new(...), level: :error)
    end
end

converted_amount = price_money.exchange_to(account.currency, fallback_rate: fallback_rate).amount
# ...
end
```

#### 建议3：增加汇率监控仪表盘

监控指标建议：
- 汇率缺失次数（按货币对）
- fallback 使用率
- 汇率更新延迟
- 错误率趋势

---

## 四、总结

### 4.1 持仓链路总结

1. **触发入口**：`PlaidItem::Syncer.perform_sync()` → `schedule_account_syncs()`
2. **市场数据准备**：`Account::Syncer.import_market_data()` 同步汇率和行情
3. **持仓计算**：`Holding::Calculator` 根据交易记录计算持仓数量，结合价格计算金额
4. **价格优先级**：数据库价格 > 交易价格 > 持仓价格
5. **汇率转换**：所有价格统一转换为账户货币

### 4.2 时区处理总结

1. **cron 配置**：`"0 22 * * 1-5"` 表示美国东部时间下午10点（需确认Sidekiq时区配置）
2. **设计保障**：使用 `in_time_zone("America/New_York")` 自动处理夏令时/冬令时
3. **执行时间**：无论DST如何变化，始终在美股收盘后1小时执行

### 4.3 汇率回退评估

**现状**：
- 回退策略：缺失时使用1:1汇率
- 告警覆盖：仅告警Provider错误，未告警fallback使用
- 误差影响：取决于货币对和汇率波动，可能产生显著误差

**建议改进**：
1. 增加汇率缺失时的告警通知
2. 使用最近可用汇率替代1:1回退
3. 增加汇率相关监控指标
