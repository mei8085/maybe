# Security 健康检查系统报告

## 1. 概述

本报告详细阐述了 Maybe 金融应用中的证券（Security）健康检查系统，包括：
- 后台定时任务调度机制
- 健康检查覆盖的核心维度
- 检查失败时的数据流转和前端反馈链路

---

## 2. 定时任务调度

### 2.1 调度配置

**位置**: `config/schedule.yml:16-20`

**调度规则**:
- **Cron 表达式**: `0 2 * * 1-5`
- **执行时间**: 每周一至周五 凌晨 2:00 AM EST / 3:00 AM EDT
- **任务类**: `SecurityHealthCheckJob`
- **队列**: `scheduled`

### 2.2 任务入口

**位置**: `app/jobs/security_health_check_job.rb:1-9`

```ruby
class SecurityHealthCheckJob < ApplicationJob
  queue_as :scheduled

  def perform
    return if Rails.env.development?  # 开发环境跳过
    Security::HealthChecker.check_all
  end
end
```

**注意**: 开发环境不执行此任务，以保护数据完整性。

---

## 3. 健康检查核心逻辑

### 3.1 批量处理策略

**位置**: `app/models/security/health_checker.rb:18-42`

系统采用**分批处理**策略，以避免一次性检查所有证券造成性能瓶颈：

| 维度 | 配置值 | 说明 |
|------|--------|------|
| `DAILY_BATCH_SIZE` | 1000 | 每天最多检查 1000 个已检查过的证券 |
| `HEALTH_CHECK_INTERVAL` | 7 天 | 已检查过的证券 7 天后才需要重新检查 |
| `MAX_CONSECUTIVE_FAILURES` | 5 次 | 连续失败 5 次标记为离线 |

### 3.2 优先级排序

检查顺序遵循以下优先级：

1. **从未检查过的证券**（`last_health_check_at: nil`）
   - 不受每日批次限制，全部检查
   - 优先级最高，确保新添加的证券尽快验证

2. **到期检查的证券**（超过 7 天未检查）
   - 按 `last_health_check_at` 升序排列（最久未检查的优先）
   - 每日最多检查 1000 个

**代码实现**:
```ruby
def check_all
  # 1. 优先检查从未检查过的证券（无数量限制）
  never_checked_scope.find_each do |security|
    new(security).run_check
  end

  # 2. 检查到期的证券（每日批次限制 1000）
  due_for_check_scope.limit(DAILY_BATCH_SIZE).each do |security|
    new(security).run_check
  end
end
```

---

## 4. 检查维度

### 4.1 核心检查维度

**位置**: `app/models/security/health_checker.rb:72-84`

健康检查仅验证**一个核心维度**：

> **能否从外部数据提供商（Provider）获取到该证券的当前价格**

检查调用链：
```
HealthChecker.run_check
  → latest_provider_price
    → provider.fetch_security_price(
        symbol: security.ticker,
        exchange_operating_mic: security.exchange_operating_mic,
        date: Date.current
      )
    → 检查 response.success?
    → 检查 response.data.price 是否存在
```

**检查参数**:
- `symbol`: 证券代码（ticker）
- `exchange_operating_mic`: 交易所操作 MIC 代码（精确标识市场）
- `date`: 当前日期

### 4.2 为什么只检查价格可获取性？

从业务角度看，这是最重要的验证：
- 价格数据是投资组合估值的基础
- 如果无法获取价格，该证券对用户来说就是"不可用"的
- 价格不可获取可能暗示：
  - 证券代码变更
  - 公司退市
  - 交易所 MIC 代码变更
  - 提供商数据覆盖缺失

---

## 5. 检查结果处理

### 5.1 成功处理

**位置**: `app/models/security/health_checker.rb:87-93`

检查成功时执行以下操作：

```ruby
def handle_success
  security.update!(
    offline: false,          # 标记为在线
    failed_fetch_count: 0,   # 重置失败计数器
    failed_fetch_at: nil     # 清空失败时间
  )
end
```

**影响**:
- 之前被标记为 offline 的证券会被恢复
- 该证券会重新纳入 `MarketDataImporter` 的每日价格导入

### 5.2 失败处理（渐进式）

**位置**: `app/models/security/health_checker.rb:95-119`

失败处理采用**渐进式降级策略**：

#### 阶段 1：累计失败计数（1-5 次）
```ruby
security.update!(
  failed_fetch_count: new_failure_count,  # 递增计数
  failed_fetch_at: Time.current           # 记录失败时间
)
```
- 证券仍保持 `offline: false` 状态
- 价格数据不会被删除

#### 阶段 2：标记离线（第 6 次失败）
```ruby
def convert_to_offline_security!
  Security.transaction do
    security.update!(
      offline: true,
      failed_fetch_count: MAX_CONSECUTIVE_FAILURES + 1,
      failed_fetch_at: Time.current
    )
    security.prices.delete_all  # 删除所有历史价格
  end
end
```

**离线状态的影响**:
- `MarketDataImporter` 会跳过该证券（不再导入价格）
- 所有历史价格数据被清空（视为不可信数据）
- 证券状态变为 `offline: true`

---

## 6. 数据模型与状态字段

### 6.1 数据库表结构

**位置**: `db/schema.rb:623-640`

```ruby
create_table "securities", force: :cascade do |t|
  t.string "ticker", null: false
  t.string "name"
  # ... 其他字段 ...
  t.boolean "offline", default: false, null: false       # 在线状态
  t.datetime "failed_fetch_at"                           # 最后失败时间
  t.integer "failed_fetch_count", default: 0, null: false # 连续失败次数
  t.datetime "last_health_check_at"                      # 最后健康检查时间
end
```

### 6.2 关键状态字段说明

| 字段 | 类型 | 默认值 | 用途 |
|------|------|--------|------|
| `offline` | boolean | false | 核心状态标识，决定是否纳入价格导入 |
| `failed_fetch_count` | integer | 0 | 连续失败计数，用于渐进式降级 |
| `failed_fetch_at` | datetime | null | 最后一次失败时间，用于追踪 |
| `last_health_check_at` | datetime | null | 最后检查时间，用于调度优先级 |

### 6.3 在线状态 Scope

**位置**: `app/models/security.rb:12`

```ruby
scope :online, -> { where(offline: false) }
```

这个 scope 被 `MarketDataImporter` 用于筛选需要导入价格的证券。

---

## 7. 前端反馈链路

健康检查的结果**不是通过主动推送**给前端的，而是通过**被动读取数据库状态**的方式体现。

### 7.1 链路概览

```
健康检查任务
    ↓
更新 securities 表状态字段
    ↓
前端页面渲染时读取这些状态
    ↓
通过价格缺失间接反馈给用户
```

### 7.2 关键影响点

#### 影响点 1：市场数据导入

**位置**: `app/models/market_data_importer.rb:25-34`

```ruby
def import_security_prices
  # 只导入在线证券的价格
  Security.online.find_each do |security|
    security.import_provider_prices(...)
    security.import_provider_details(...)
  end
end
```

**效果**:
- 离线证券不会被导入新价格
- 价格数据会逐渐过时（或已被清空）

#### 影响点 2：证券解析器

**位置**: `app/models/security/resolver.rb:30-44`

当用户输入一个无法在数据库或提供商找到的证券代码时：

```ruby
def offline_security
  security = Security.find_or_initialize_by(
    ticker: symbol,
    exchange_operating_mic: exchange_operating_mic,
  )

  security.assign_attributes(
    country_code: country_code,
    offline: true  # 直接标记为离线
  )

  security.save!
  security
end
```

#### 影响点 3：交易表单

**位置**: `app/models/trade/create_form.rb:22-29`

用户创建交易时，系统会尝试解析证券：

```ruby
def security
  ticker_symbol, exchange_operating_mic = ticker.present? ? ticker.split("|") : [ manual_ticker, nil ]

  Security::Resolver.new(
    ticker_symbol,
    exchange_operating_mic: exchange_operating_mic
  ).resolve
end
```

**可能的结果**:
- 如果证券已存在且被标记为 offline，用户可能无法获取最新价格
- 如果证券无法被解析，会创建一个 offline 证券

### 7.3 前端显示效果

前端页面通过**价格是否存在**来间接体现健康状态：

#### 持仓详情页

**位置**: `app/views/holdings/show.html.erb:22-25`

```erb
<div class="flex items-center justify-between text-sm">
  <dt class="text-secondary"><%= t(".current_market_price_label") %></dt>
  <dd class="text-primary">
    <%= @holding.security.current_price ? format_money(@holding.security.current_price) : t(".unknown") %>
  </dd>
</div>
```

**效果**:
- 如果证券 offline 且无价格 → 显示 `"Unknown"`
- 用户感知："当前市价未知"

#### 交易详情页

**位置**: `app/views/trades/_header.html.erb:55-60`

```erb
<% if trade.security.current_price.present? %>
  <div class="flex items-center justify-between text-sm">
    <dt class="text-secondary"><%= t(".current_market_price_label") %></dt>
    <dd class="text-primary"><%= format_money trade.security.current_price %></dd>
  </div>
<% end %>
```

**效果**:
- 如果无当前价格 → 整个价格区块不显示
- 用户感知：界面缺少关键信息

### 7.4 当前价格获取逻辑

**位置**: `app/models/security.rb:14-18`

```ruby
def current_price
  @current_price ||= find_or_fetch_price
  return nil if @current_price.nil?
  Money.new(@current_price.price, @current_price.currency)
end
```

如果 `find_or_fetch_price` 返回 nil（离线证券没有价格记录），前端就会看到"Unknown"或价格区块消失。

---

## 8. 健康检查流程图

```
┌─────────────────────────────────────────────────────────────────────┐
│                         定时任务调度                                  │
│  config/schedule.yml → SecurityHealthCheckJob (每天 2:00 AM)         │
└─────────────────────────┬───────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────────────┐
│                    Security::HealthChecker.check_all                  │
│                                                                      │
│  1. 从未检查的证券（无数量限制）                                       │
│     last_health_check_at: nil                                        │
│                          ↓                                           │
│  2. 到期检查的证券（每天 1000 个）                                     │
│     last_health_check_at <= 7 天前                                    │
└─────────────────────────┬───────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────────────┐
│                    run_check: 调用 Provider 验证价格可获取性           │
└─────────────────────────┬───────────────────────────────────────────┘
                          │
              ┌───────────┴───────────┐
              ↓                       ↓
        ┌───────────┐           ┌───────────┐
        │  成功     │           │  失败     │
        └─────┬─────┘           └─────┬─────┘
              ↓                       ↓
    ┌─────────────────┐     ┌────────────────────────┐
    │ 更新状态:       │     │ 检查失败次数:          │
    │ offline: false  │     │                       │
    │ failed_fetch: 0 │     │ 1-5 次 → 只计数       │
    │                 │     │                       │
    │ 恢复价格导入     │     │ >=6 次 → 标记离线     │
    └─────────────────┘     │ 并删除所有历史价格     │
                            └───────────┬────────────┘
                                        ↓
                            ┌───────────────────────┐
                            │  持久化到数据库        │
                            │  securities 表         │
                            │  - offline=true       │
                            │  - 价格记录被清空      │
                            └───────────┬───────────┘
                                        ↓
┌─────────────────────────────────────────────────────────────────────┐
│                         前端反馈链路                                  │
│                                                                      │
│  1. MarketDataImporter 跳过 offline 证券 → 无新价格                  │
│  2. current_price 方法返回 nil                                      │
│  3. 前端页面显示 "Unknown" 或价格区块消失                            │
│  4. 用户感知：价格信息缺失，投资组合估值可能不准确                    │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 9. 测试覆盖

**位置**: `test/models/security/health_checker_test.rb`

测试覆盖以下场景：

| 测试用例 | 验证内容 |
|----------|----------|
| `any security without a health check runs` | 未检查过的证券会被优先检查 |
| `offline security with no health check that fails stays offline` | 离线证券检查失败保持离线 |
| `after enough consecutive health check failures, security goes offline` | 连续 5 次失败后标记离线并删除价格 |
| `failure incrementor increases for each health check failure` | 失败计数器正确递增 |
| `failure incrementor resets to 0 when health check succeeds` | 成功后计数器重置 |

---

## 10. 设计特点总结

### 优点
1. **渐进式容错**: 连续 5 次失败才标记离线，避免偶发网络问题导致误判
2. **数据保护**: 标记离线时删除历史价格，防止错误数据影响估值
3. **优先级调度**: 未检查过的证券优先处理，确保新数据及时验证
4. **批量控制**: 每日 1000 个限制，避免性能瓶颈

### 潜在改进点
1. **前端反馈不直观**: 用户看到"Unknown"价格，但不知道原因是健康检查失败
2. **无通知机制**: 证券离线时没有主动通知用户或管理员
3. **离线恢复机制**: 离线证券恢复在线后，需要手动触发价格重新导入
4. **检查维度单一**: 仅验证价格可获取性，未检查数据质量、波动性等

---

## 11. 关键文件索引

| 文件路径 | 职责 |
|----------|------|
| `config/schedule.yml` | 定时任务配置 |
| `app/jobs/security_health_check_job.rb` | 任务入口 |
| `app/models/security/health_checker.rb` | 健康检查核心逻辑 |
| `app/models/security.rb` | Security 模型定义 |
| `app/models/security/resolver.rb` | 证券解析与离线标记 |
| `app/models/market_data_importer.rb` | 价格导入（受 offline 状态影响） |
| `db/schema.rb` | 数据库表结构 |
| `test/models/security/health_checker_test.rb` | 测试用例 |
