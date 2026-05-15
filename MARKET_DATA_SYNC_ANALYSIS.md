# 外部市场数据同步机制分析报告

## 一、概述

本报告梳理家庭账本系统中外市场数据的周期性同步机制，包括：
- 调度链设计
- 缓存命中策略
- 失败重试机制
- 数据源缺值时的账本一致性保障

---

## 二、调度链设计

### 2.1 整体架构

系统采用**双层同步架构**：

```
┌─────────────────────────────────────────────────────────────────┐
│                      定时任务层 (每日)                           │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  ImportMarketDataJob (config/schedule.yml)              │    │
│  │  → MarketDataImporter.import_all()                      │    │
│  │    → import_security_prices()                           │    │
│  │    → import_exchange_rates()                            │    │
│  └─────────────────────────────────────────────────────────┘    │
│                              ↓                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │             账户同步层 (按需触发)                          │    │
│  │  Account::Syncer.perform_sync()                         │    │
│  │  → Account::MarketDataImporter.import_all()             │    │
│  │    → import_exchange_rates()                            │    │
│  │    → import_security_prices()                           │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 定时任务调度

**配置位置**: `config/schedule.yml`

```yaml
import_market_data:
cron: "0 22 * * 1-5"  # 美国东部时间下午5点（收盘后1小时）
class: "ImportMarketDataJob"
queue: "scheduled"
args:
    mode: "full"
    clear_cache: false
```

**执行时间**: 工作日（周一至周五）美国东部时间下午5点，即美股收盘后1小时。

### 2.3 同步模式

`MarketDataImporter` 支持两种模式：

| 模式 | 说明 | 时间范围 |
|------|------|----------|
| `:full` | 完整同步 | 从第一条交易记录日期到当前 |
| `:snapshot` | 快照同步 | 最近31天（`SNAPSHOT_DAYS = 31`） |

### 2.4 执行流程

#### 2.4.1 定时任务入口

```ruby
# app/jobs/import_market_data_job.rb
def perform(opts)
return if Rails.env.development?  # 开发环境跳过
opts = opts.symbolize_keys
mode = opts.fetch(:mode, :full)
clear_cache = opts.fetch(:clear_cache, false)

MarketDataImporter.new(mode: mode, clear_cache: clear_cache).import_all
end
```

#### 2.4.2 安全价格导入流程

```ruby
# app/models/market_data_importer.rb
def import_security_prices
Security.online.find_each do |security|
    security.import_provider_prices(
    start_date: get_first_required_price_date(security),
    end_date: end_date,
    clear_cache: clear_cache
    )
    security.import_provider_details(clear_cache: clear_cache)
end
end
```

**关键逻辑**：
- 仅处理 `offline: false` 的证券
- `get_first_required_price_date` 根据模式决定起始日期：
- `:full` 模式：该证券最早交易日期
- `:snapshot` 模式：31天前

#### 2.4.3 汇率导入流程

```ruby
def import_exchange_rates
required_exchange_rate_pairs.each do |pair|
    start_date = snapshot? ? default_start_date : pair[:start_date]
    ExchangeRate.import_provider_rates(
    from: pair[:source],
    to: pair[:target],
    start_date: start_date,
    end_date: end_date,
    clear_cache: clear_cache
    )
end
end
```

**汇率需求分析**（`required_exchange_rate_pairs`）：
1. **基于交易记录**：所有 entry 货币与账户货币不同的组合
2. **基于账户配置**：账户货币与家庭货币不同的组合

#### 2.4.4 账户级补充同步

```ruby
# app/models/account/syncer.rb
def import_market_data
Account::MarketDataImporter.new(account).import_all
rescue => e
Rails.logger.error("Error syncing market data for account #{account.id}: #{e.message}")
Sentry.capture_exception(e)
end
```

**设计意图**：
- 定时任务已覆盖大部分市场数据需求
- 账户同步时作为**补充**，确保数据完整性
- 使用 `rescue` 保护，市场数据失败不影响账户同步

---

## 三、缓存命中策略

### 3.1 价格缓存机制

**核心实现**：`Security::Price::Importer`

```ruby
# app/models/security/price/importer.rb
def import_provider_prices
# 缓存命中检查
if !clear_cache && all_prices_exist?
    Rails.logger.info("No new prices to sync for #{security.ticker}")
    return 0
end

# 计算有效起始日期（跳过已存在的日期）
effective_start_date = (start_date..end_date).detect { |d| !db_prices.key?(d) } || end_date

# 从provider获取数据（提前5天获取用于LOCF）
provider_fetch_start_date = effective_start_date - 5.days
response = security_provider.fetch_security_prices(...)
end
```

**缓存命中策略**：

| 条件 | 行为 |
|------|------|
| `clear_cache: false` 且所有日期数据已存在 | 跳过，直接返回 |
| `clear_cache: false` 且部分日期缺失 | 仅补充缺失日期 |
| `clear_cache: true` | 强制重新获取并覆盖 |

### 3.2 单条价格查询缓存

```ruby
# app/models/security/provided.rb
def find_or_fetch_price(date: Date.current, cache: true)
price = prices.find_by(date: date)  # 先查数据库
return price if price.present?

return nil unless provider.present?
response = provider.fetch_security_price(...)
return nil unless response.success?

# 缓存到数据库（如果启用）
Security::Price.find_or_create_by!(...) if cache
price
end
```

### 3.3 安全详情缓存

```ruby
def import_provider_details(clear_cache: false)
# 如果已有名称和logo且不清除缓存，跳过
if self.name.present? && self.logo_url.present? && !clear_cache
    return
end
# ... 获取并更新详情
end
```

### 3.4 缓存有效性保障

- **数据唯一性约束**：`validates :date, uniqueness: { scope: %i[security_id currency] }`
- **幂等性设计**：使用 `upsert_all` 确保重复导入不会产生重复数据
- **批量操作优化**：批量大小为 200（`batch_size = 200`）

---

## 四、失败重试机制

### 4.1 Provider 层重试

**实现位置**：`Provider::Synth.client`

```ruby
# app/models/provider/synth.rb
def client
@client ||= Faraday.new(url: base_url) do |faraday|
    faraday.request(:retry, {
    max: 2,                    # 最多重试2次
    interval: 0.05,            # 初始间隔50ms
    interval_randomness: 0.5,  # 随机因子50%
    backoff_factor: 2          # 指数退避因子2
    })
    faraday.response :raise_error
    # ...
end
end
```

**重试策略参数**：

| 参数 | 值 | 说明 |
|------|-----|------|
| `max` | 2 | 最多重试2次，总共尝试3次 |
| `interval` | 0.05s | 首次重试等待50ms |
| `interval_randomness` | 0.5 | 随机抖动50% |
| `backoff_factor` | 2 | 指数退避，第n次等待 = 0.05 * 2^(n-1) |

**退避时间表**：
- 第1次失败：等待 50ms ± 25ms
- 第2次失败：等待 100ms ± 50ms

### 4.2 错误处理封装

**统一响应模式**：`Provider::Response`

```ruby
# app/models/provider.rb
Response = Data.define(:success?, :data, :error)

def with_provider_response(error_transformer: nil, &block)
data = yield
Response.new(success?: true, data: data, error: nil)
rescue => error
transformed_error = error_transformer ? error_transformer.call(error) : default_error_transformer(error)
Response.new(success?: false, data: nil, error: transformed_error)
end
```

**错误转换逻辑**：

```ruby
def default_error_transformer(error)
if error.is_a?(Faraday::Error)
    self.class::Error.new(
    error.message,
    details: error.response&.dig(:body),
    )
else
    self.class::Error.new(error.message)
end
end
```

### 4.3 日志与监控

**日志记录点**：

```ruby
# 价格获取失败
Rails.logger.warn("#{security_provider.class.name} could not fetch prices for #{security.ticker}")

# 汇率数据无效
Rails.logger.warn("#{self.class.name} returned invalid rate data for pair from: #{from} to: #{to}")

# 安全信息缺失
Sentry.capture_exception(SecurityInfoMissingError.new(...), level: :warning) do |scope|
scope.set_tags(security_id: self.id)
scope.set_context("security", {...})
end
```

### 4.4 熔断保护

**账户同步层**：

```ruby
# app/models/account/syncer.rb
def import_market_data
Account::MarketDataImporter.new(account).import_all
rescue => e
Rails.logger.error("Error syncing market data for account #{account.id}: #{e.message}")
Sentry.capture_exception(e)
# 异常被捕获，不影响后续的余额计算
end
```

**设计原则**：
- 市场数据获取失败**不应阻断**账户同步流程
- 使用合理的 fallback 机制处理缺失数据

---

## 五、数据源缺值时的一致性保障

### 5.1 价格缺失处理：LOCF 算法

**实现位置**：`Security::Price::Importer.import_provider_prices`

```ruby
gapfilled_prices = effective_start_date.upto(end_date).map do |date|
db_price_value       = db_prices[date]&.price
provider_price_value = provider_prices[date]&.price
provider_currency    = provider_prices[date]&.currency

# 选择优先级
chosen_price = if clear_cache
    provider_price_value || db_price_value   # 清除缓存模式：优先provider
else
    db_price_value || provider_price_value   # 正常模式：优先缓存
end

# LOCF：使用上一个观测值填充
chosen_price ||= prev_price_value
prev_price_value = chosen_price  # 更新上一个值

{
    security_id: security.id,
    date: date,
    price: chosen_price,
    currency: provider_currency || prev_price_currency || db_price_currency || "USD"
}
end
```

**LOCF（Last Observation Carried Forward）策略**：

| 场景 | 处理方式 |
|------|----------|
| 数据库有值 | 使用数据库值 |
| 数据库无值，provider有值 | 使用provider值 |
| 两者都无值 | 使用上一个日期的价格 |

**货币处理**：
- 优先使用provider返回的货币
- 降级到前一个价格的货币
- 最终兜底为 USD

### 5.2 起始价格缺失处理

```ruby
prev_price_value = start_price_value

unless prev_price_value.present?
Rails.logger.error("Could not find a start price for #{security.ticker} on or before #{start_date}")

Sentry.capture_exception(MissingStartPriceError.new(...)) do |scope|
    scope.set_tags(security_id: security.id)
end

return 0  # 无法继续，返回0条更新
end
```

**起始价格查找逻辑**：

```ruby
def start_price_value
# 优先从provider获取起始日期或之前的价格
provider_price_value = provider_prices.select { |date, _| date <= start_date }
                                        .max_by { |date, _| date }
                                        &.last&.price
# 降级到数据库
db_price_value = db_prices[start_date]&.price
provider_price_value || db_price_value
end
```

### 5.3 持仓缺失处理

**实现位置**：`Holding::Gapfillable.gapfill`

```ruby
holdings.group_by { |h| h.security_id }.each do |security_id, security_holdings|
sorted = security_holdings.sort_by(&:date)
previous_holding = sorted.first

sorted.first.date.upto(Date.current) do |date|
    holding = security_holdings.find { |h| h.date == date }

    if holding
    filled_holdings << holding
    previous_holding = holding
    else
    # 创建填充持仓，使用前一天的数据
    filled_holdings << Holding.new(
        account: previous_holding.account,
        security: previous_holding.security,
        date: date,
        qty: previous_holding.qty,
        price: previous_holding.price,
        currency: previous_holding.currency,
        amount: previous_holding.amount
    )
    end
end
end
```

### 5.4 汇率缺失处理

**实现位置**：`Balance::SyncCache`

```ruby
def converted_entries
@converted_entries ||= account.entries.order(:date).to_a.map do |e|
    converted_entry = e.dup
    converted_entry.amount = converted_entry.amount_money.exchange_to(
    account.currency,
    date: e.date,
    fallback_rate: 1  # 汇率缺失时使用1:1转换
    ).amount
    converted_entry.currency = account.currency
    converted_entry
end
end
```

**fallback 策略**：
- 汇率缺失时使用 `fallback_rate: 1`
- 即假设 1:1 汇率，保证计算能继续

### 5.5 证券解析失败处理

**实现位置**：`Security::Resolver.resolve`

```ruby
def resolve
return nil if symbol.blank?

exact_match_from_db ||
    exact_match_from_provider ||
    close_match_from_provider ||
    offline_security  # 创建离线证券
end

def offline_security
security = Security.find_or_initialize_by(
    ticker: symbol,
    exchange_operating_mic: exchange_operating_mic,
)

security.assign_attributes(
    country_code: country_code,
    offline: true  # 标记为离线，避免后续重试
)

security.save!
security
end
```

**设计意图**：
- 无法解析的证券标记为 `offline: true`
- `Security.online` 作用域排除离线证券
- 避免对无效证券的重复查询

---

## 六、数据一致性保障总结

### 6.1 一致性保障机制汇总

| 层级 | 机制 | 保障目标 |
|------|------|----------|
| Provider | Faraday 重试 + 指数退避 | 网络抖动容错 |
| 价格导入 | LOCF 算法 | 日期连续性 |
| 持仓计算 | Gapfill 填充 | 持仓连续性 |
| 汇率转换 | fallback_rate: 1 | 计算可继续 |
| 证券解析 | offline 标记 | 避免无效重试 |
| 账户同步 | rescue 保护 | 部分失败不影响整体 |

### 6.2 数据流转一致性

```
Provider API → Response 对象 → 数据库 upsert → 余额计算
     ↓              ↓                ↓              ↓
   失败重试      错误捕获        唯一约束        LOCF填充
     ↓              ↓                ↓              ↓
  Sentry告警    日志记录       幂等性保障      连续性保障
```

### 6.3 潜在改进点

1. **熔断机制增强**：当前缺乏对持续失败的熔断处理，建议引入断路器模式
2. **监控指标完善**：增加市场数据成功率、缓存命中率等关键指标
3. **数据过期策略**：考虑对过旧数据设置有效期，定期清理
4. **多数据源支持**：当前仅支持 Synth，可考虑多数据源冗余

---

## 七、关键文件清单

| 文件路径 | 职责 |
|----------|------|
| `app/jobs/import_market_data_job.rb` | 定时任务入口 |
| `app/models/market_data_importer.rb` | 全局市场数据导入器 |
| `app/models/account/market_data_importer.rb` | 账户级市场数据导入器 |
| `app/models/security/price/importer.rb` | 价格导入与LOCF填充 |
| `app/models/security/provided.rb` | 证券数据Provider接口 |
| `app/models/provider/synth.rb` | Synth Provider实现 |
| `app/models/provider.rb` | Provider基类与响应封装 |
| `app/models/holding/gapfillable.rb` | 持仓间隙填充 |
| `app/models/balance/sync_cache.rb` | 余额计算缓存 |
| `config/schedule.yml` | 定时任务配置 |
