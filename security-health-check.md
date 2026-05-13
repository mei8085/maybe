# Security 健康检查系统报告

## 1. 概述

本报告详细阐述了 Maybe 金融应用中的证券（Security）健康检查系统，包括：
- 后台定时任务调度机制
- 健康检查覆盖的核心维度
- Provider 调用抛异常时的处理分支
- 各状态字段（failed_fetch_count、offline、last_health_check_at）的变化逻辑
- 从状态变化 → 价格读取 → 页面展示的完整前端反馈链路

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

## 5. 检查结果处理（含异常分支）

### 5.1 run_check 完整执行流程

**位置**: `app/models/security/health_checker.rb:49-63`

```ruby
def run_check
  Rails.logger.info("Running health check for #{security.ticker}")

  if latest_provider_price
    handle_success
  else
    handle_failure
  end
rescue => e
  Sentry.capture_exception(e) do |scope|
    scope.set_tags(security_id: @security.id)
  end
ensure
  security.update!(last_health_check_at: Time.current)
end
```

**关键观察**:
- `ensure` 块**始终执行**，无论成功、失败还是异常
- `last_health_check_at` **始终被更新**为当前时间

### 5.2 四种执行路径与状态字段变化

#### 路径 1：检查成功（Provider 返回有效价格）

**触发条件**:
- `provider.fetch_security_price` 调用成功
- `response.success? == true`
- `response.data.price` 存在

**执行路径**:
```
latest_provider_price 返回非 nil
  → handle_success
  → ensure: last_health_check_at 更新
```

**状态字段变化**:

| 字段 | 变化 |
|------|------|
| `offline` | `true` → `false`（被重置） |
| `failed_fetch_count` | 任意值 → `0`（被重置） |
| `failed_fetch_at` | 任意值 → `nil`（被清空） |
| `last_health_check_at` | 任意值 → 当前时间（**始终更新**） |

**handle_success 代码** (`app/models/security/health_checker.rb:87-93`):
```ruby
def handle_success
  security.update!(
    offline: false,
    failed_fetch_count: 0,
    failed_fetch_at: nil
  )
end
```

---

#### 路径 2：检查失败（Provider 返回空价格或响应失败）

**触发条件**（二选一）:
- `response.success? == false`（Provider 返回错误响应）
- `response.data.price` 为 `nil`（响应成功但无价格数据）

**执行路径**:
```
latest_provider_price 返回 nil
  → handle_failure
  → ensure: last_health_check_at 更新
```

**状态字段变化（分阶段）**:

**阶段 A：1-5 次连续失败** (`failed_fetch_count` < 5)

| 字段 | 变化 |
|------|------|
| `offline` | 保持不变（仍为 `false`） |
| `failed_fetch_count` | `n` → `n + 1`（递增） |
| `failed_fetch_at` | 任意值 → 当前时间（更新） |
| `last_health_check_at` | 任意值 → 当前时间（**始终更新**） |

**阶段 B：第 6 次及以上连续失败** (`failed_fetch_count` >= 5)

| 字段 | 变化 |
|------|------|
| `offline` | `false` → `true`（标记离线） |
| `failed_fetch_count` | `5` → `6`（设置为 MAX+1） |
| `failed_fetch_at` | 任意值 → 当前时间（更新） |
| `last_health_check_at` | 任意值 → 当前时间（**始终更新**） |
| `security_prices` 表 | **所有记录被删除** |

**handle_failure 代码** (`app/models/security/health_checker.rb:95-119`):
```ruby
def handle_failure
  new_failure_count = security.failed_fetch_count.to_i + 1
  new_failure_at = Time.current

  if new_failure_count > MAX_CONSECUTIVE_FAILURES
    convert_to_offline_security!
  else
    security.update!(
      failed_fetch_count: new_failure_count,
      failed_fetch_at: new_failure_at
    )
  end
end

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

---

#### 路径 3：Provider 调用抛异常

**触发条件**:
- `provider.fetch_security_price` 内部抛出异常（网络超时、API 限流、连接错误等）
- 异常在 `rescue => e` 块被捕获

**执行路径**:
```
latest_provider_price 执行中抛出异常
  → rescue: Sentry 上报异常
  → ensure: last_health_check_at 更新
  → handle_success 和 handle_failure 都**不执行**
```

**状态字段变化（关键！）**:

| 字段 | 变化 | 说明 |
|------|------|------|
| `offline` | **保持不变** | `handle_failure` 未执行 |
| `failed_fetch_count` | **保持不变** | **异常不计入失败计数** |
| `failed_fetch_at` | **保持不变** | 不更新 |
| `last_health_check_at` | **更新为当前时间** | `ensure` 块始终执行 |

**代码说明**:
```ruby
def run_check
  # ... 业务逻辑 ...
rescue => e
  # 只上报异常，不更新状态字段
  Sentry.capture_exception(e) do |scope|
    scope.set_tags(security_id: @security.id)
  end
ensure
  # 无论如何，都更新 last_health_check_at
  security.update!(last_health_check_at: Time.current)
end
```

**异常路径的重要特性**:
1. **异常不会导致 `failed_fetch_count` 递增**
2. **异常不会导致证券标记为 `offline`**
3. **异常不会删除历史价格**
4. **异常会重置 7 天检查周期**（因为 `last_health_check_at` 被更新）

---

#### 路径 4：Provider 未配置

**触发条件**:
- `provider.present? == false`（没有配置证券数据提供商）

**执行路径**:
```
latest_provider_price 中 provider 为 nil
  → 直接返回 nil（不调用 Provider）
  → handle_failure
  → ensure: last_health_check_at 更新
```

**状态字段变化**:
与 **路径 2（检查失败）** 完全相同。

**代码** (`app/models/security/health_checker.rb:72-74`):
```ruby
def latest_provider_price
  return nil unless provider.present?  # 直接返回 nil，进入 handle_failure
  # ...
end
```

---

### 5.3 四条路径对比汇总

| 路径 | 触发条件 | `offline` | `failed_fetch_count` | `failed_fetch_at` | `last_health_check_at` | 价格数据 |
|------|----------|-----------|---------------------|-------------------|-----------------------|---------|
| **1. 成功** | Provider 返回有效价格 | 设为 false | 设为 0 | 设为 nil | 更新为当前时间 | 保留 |
| **2. 失败(1-5次)** | Provider 返回空/错误 | 不变 | +1 | 更新 | 更新为当前时间 | 保留 |
| **2. 失败(≥6次)** | Provider 返回空/错误 | 设为 true | 设为 6 | 更新 | 更新为当前时间 | **删除** |
| **3. 异常** | Provider 调用抛异常 | **不变** | **不变** | **不变** | **更新为当前时间** | **保留** |
| **4. 无Provider** | 未配置数据提供商 | 同路径2 | 同路径2 | 同路径2 | 更新为当前时间 | 同路径2 |

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

| 字段 | 类型 | 默认值 | 用途 | 更新时机 |
|------|------|--------|------|---------|
| `offline` | boolean | false | 核心状态标识，决定是否纳入价格导入 | 成功时设为 false，连续失败 6 次设为 true |
| `failed_fetch_count` | integer | 0 | 连续失败计数，用于渐进式降级 | 成功时设为 0，失败时 +1，**异常时不变** |
| `failed_fetch_at` | datetime | null | 最后一次失败时间，用于追踪 | 成功时设为 nil，失败时更新 |
| `last_health_check_at` | datetime | null | 最后检查时间，用于调度优先级 | **无论成功/失败/异常，始终更新** |

### 6.3 在线状态 Scope

**位置**: `app/models/security.rb:12`

```ruby
scope :online, -> { where(offline: false) }
```

这个 scope 被 `MarketDataImporter` 用于筛选需要导入价格的证券。

---

## 7. 前端反馈完整链路

健康检查的结果**不是通过主动推送**给前端的，而是通过**被动读取数据库状态**的方式体现。

### 7.1 链路概览

```
健康检查任务
    ↓
更新 securities 表状态字段
    ↓
影响 MarketDataImporter（价格导入）
    ↓
影响 security_prices 表数据
    ↓
影响 current_price 方法返回值
    ↓
影响 Holding/Trade 业务计算
    ↓
前端页面展示
```

### 7.2 从状态到页面的完整链路

#### 第 1 层：健康检查 → 状态字段

如上一章所述，四条路径产生不同的状态组合。

#### 第 2 层：状态字段 → 价格导入

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

**影响**:
| `offline` 状态 | MarketDataImporter 行为 |
|----------------|-------------------------|
| `false` | 被纳入导入列表，尝试导入新价格 |
| `true` | **被跳过**，不导入新价格 |

**价格导入逻辑** (`app/models/security/price/importer.rb:15-66`):
- 调用 `provider.fetch_security_prices` 获取价格区间数据
- 若 Provider 返回错误，记录警告到 Sentry，返回空哈希 `{}`
- 使用 LOCF（Last Observation Carried Forward）填补价格缺口
- 结果 upsert 到 `security_prices` 表

#### 第 3 层：价格导入 → security_prices 表

| 场景 | security_prices 表变化 |
|------|---------------------|
| 健康检查成功 + 价格导入成功 | 新增/更新价格记录 |
| 健康检查失败 (1-5次) + 价格导入 | 仍在线，可能有新价格 |
| 健康检查失败 (≥6次) | **所有历史价格被删除** |
| 健康检查异常 | 价格数据**保留**（异常不删除价格） |

#### 第 4 层：security_prices → current_price 方法

**位置**: `app/models/security.rb:14-18` 和 `app/models/security/provided.rb:39-62`

```ruby
# app/models/security.rb
def current_price
  @current_price ||= find_or_fetch_price
  return nil if @current_price.nil?
  Money.new(@current_price.price, @current_price.currency)
end

# app/models/security/provided.rb
def find_or_fetch_price(date: Date.current, cache: true)
  price = prices.find_by(date: date)  # 先查数据库

  return price if price.present?

  # 数据库没有，尝试从 Provider 获取
  return nil unless provider.present?
  response = provider.fetch_security_price(...)

  return nil unless response.success?

  price = response.data
  Security::Price.find_or_create_by!(...) if cache
  price
end
```

**current_price 返回逻辑**:

| 场景 | `prices.find_by(date: today)` | Provider 可获取 | `current_price` 返回 |
|------|------------------------------|-----------------|---------------------|
| 数据库有今日价格 | 非 nil | 不调用 | 返回数据库价格 |
| 数据库无今日价格 | nil | 成功 | 返回 Provider 价格并缓存 |
| 数据库无今日价格 | nil | 失败/异常 | 返回 `nil` |
| 证券被标记 offline 且价格已删除 | nil | 不调用（被导入器跳过） | 返回 `nil` |

#### 第 5 层：current_price → 业务计算

##### Holding（持仓）
**位置**: `app/views/holdings/show.html.erb` 和 `app/models/holding.rb`

| 计算项 | 依赖 `current_price` |
|--------|---------------------|
| 当前市价 | **依赖**（直接显示） |
| 投资组合权重 | 不依赖（基于 `amount`/`account.balance`） |
| 平均成本 | 不依赖（基于历史交易） |
| 总收益趋势 | 不依赖（基于 `amount` vs `start_amount`） |

##### Trade（交易）
**位置**: `app/models/trade.rb:18-27`

```ruby
def unrealized_gain_loss
  return nil if qty.negative?
  current_price = security.current_price
  return nil if current_price.nil?  # 无价格则无法计算

  current_value = current_price * qty.abs
  cost_basis = price_money * qty.abs

  Trend.new(current: current_value, previous: cost_basis)
end
```

| 计算项 | 依赖 `current_price` |
|--------|---------------------|
| 当前市价 | **依赖** |
| 未实现盈亏 | **依赖**（无价格则返回 nil） |

#### 第 6 层：业务计算 → 页面展示

##### 场景 A：证券在线，有价格数据

```
健康检查成功
  → offline=false
  → 价格导入正常
  → security_prices 有今日价格
  → current_price 返回 Money 对象
```

**持仓详情页** (`app/views/holdings/show.html.erb:22-25`):
```erb
<dd class="text-primary"><%= format_money(@holding.security.current_price) %></dd>
```
→ **显示**: `$100.00`（具体价格）

**交易详情页** (`app/views/trades/_header.html.erb:55-69`):
```erb
<% if trade.security.current_price.present? %>
  <div>当前市价: <%= format_money trade.security.current_price %></div>
  <% if trade.unrealized_gain_loss.present? %>
    <div>总收益: <%= render "shared/trend_change", trend: ... %></div>
  <% end %>
<% end %>
```
→ **显示**:
  - 当前市价: `$105.00`
  - 总收益: `+5.00 (+5.00%)`（绿色/红色趋势）

---

##### 场景 B：证券离线（连续失败 ≥6 次）

```
健康检查连续失败 6 次
  → offline=true
  → prices.delete_all（所有价格被删除）
  → 价格导入被跳过
  → security_prices 无记录
  → current_price 返回 nil
```

**持仓详情页**:
```erb
<dd class="text-primary"><%= t(".unknown") %></dd>
```
→ **显示**: `Unknown`（国际化文本）

**交易详情页**:
```erb
<% if trade.security.current_price.present? %>
  # 整个区块不渲染
<% end %>
```
→ **显示**: 当前市价和总收益区块**完全消失**

**用户感知**:
- 持仓详情：市价显示为 "Unknown"
- 交易详情：缺少当前市价和收益信息
- 投资组合估值：可能不准确（依赖其他计算方式）

---

##### 场景 C：Provider 调用异常（关键差异）

```
健康检查时 Provider 抛异常
  → rescue 块捕获，Sentry 上报
  → offline 保持不变（假设之前是 false）
  → failed_fetch_count 保持不变（不递增）
  → last_health_check_at 更新（7 天周期重置）
  → 价格数据**保留**
```

**价格导入行为**:
- 由于 `offline=false`，MarketDataImporter **仍然会尝试导入**
- 如果导入时 Provider 也异常，则无新价格
- 但**旧价格数据保留**

**current_price 行为**:
| 情况 | 返回值 |
|------|--------|
| 数据库有历史价格 | 返回**旧价格**（可能已过时） |
| 数据库无价格但 Provider 可获取 | 尝试获取，可能成功或失败 |
| 数据库无价格且 Provider 异常 | 返回 `nil` |

**页面展示**:
- **如果有历史价格缓存**: 显示**过时的价格**（用户可能不知道数据已过期）
- **如果无价格缓存**: 显示 `Unknown` 或区块消失

**异常路径的隐蔽问题**:
1. **失败计数不递增**: 连续多次异常不会触发离线保护
2. **检查周期被重置**: 7 天后才会再次检查，但问题可能持续存在
3. **可能展示过时价格**: 历史价格仍在显示，但 Provider 已不可用
4. **仅 Sentry 告警**: 开发人员知道问题，但用户无感知

---

### 7.3 前端展示影响汇总表

| 健康检查结果 | `offline` | 价格数据 | `current_price` | 持仓详情 | 交易详情 |
|-------------|-----------|---------|-----------------|---------|---------|
| **成功** | false | 有 | Money 对象 | 显示价格 | 显示价格+收益 |
| **失败(1-5次)** | false | 可能有 | Money 对象 或 nil | 显示价格 或 Unknown | 显示 或 消失 |
| **失败(≥6次)** | true | 被删除 | nil | 显示 Unknown | 区块消失 |
| **异常** | 不变 | 保留 | 旧价格 或 nil | 显示过时价格 或 Unknown | 显示 或 消失 |
| **无 Provider** | 同失败路径 | 同失败路径 | 同失败路径 | 同失败路径 | 同失败路径 |

---

## 8. 健康检查流程图（完整版）

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
│                      run_check: 调用 Provider                         │
│                                                                      │
│   if latest_provider_price                                           │
│     → handle_success                                                 │
│   else                                                               │
│     → handle_failure                                                 │
│                                                                      │
│   rescue => e                                                        │
│     → Sentry.capture_exception (仅上报，不更新状态)                   │
│                                                                      │
│   ensure                                                             │
│     → last_health_check_at = Time.current (始终执行)                 │
└─────────────────────────┬───────────────────────────────────────────┘
                          │
          ┌───────────────┼───────────────┐
          ↓               ↓               ↓
    ┌─────────┐     ┌─────────┐     ┌─────────┐
    │ 成功    │     │ 失败    │     │ 异常    │
    │ (有价格)│     │ (无价格)│     │ (抛异常)│
    └────┬────┘     └────┬────┘     └────┬────┘
         ↓               ↓               ↓
┌─────────────────┐ ┌───────────────┐ ┌──────────────────┐
│ handle_success  │ │ handle_failure│ │ rescue + ensure  │
│                 │ │               │ │                  │
│ offline=false   │ │ 1-5次:        │ │ offline 不变     │
│ failed_count=0  │ │   仅计数       │ │ failed_count 不变│
│ failed_at=nil   │ │               │ │ failed_at 不变   │
│                 │ │ ≥6次:         │ │                  │
│ 价格保留        │ │   offline=true │ │ 价格保留         │
│                 │ │   删除价格     │ │                  │
└────────┬────────┘ └───────┬───────┘ └────────┬─────────┘
         │                 │                   │
         └─────────────────┼───────────────────┘
                           ↓
                   ┌───────────────┐
                   │ 持久化到数据库  │
                   │ securities 表  │
                   └───────┬───────┘
                           ↓
┌─────────────────────────────────────────────────────────────────────┐
│                        MarketDataImporter                           │
│                                                                     │
│   Security.online.find_each  ← 只导入 offline=false 的证券           │
│     ↓                                                               │
│   import_provider_prices                                             │
│     ↓                                                               │
│   upsert 到 security_prices 表                                       │
└───────────────────────────┬─────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────────────┐
│                        current_price 方法                            │
│                                                                     │
│   prices.find_by(date: today)  → 有则返回                            │
│     ↓ 无                                                             │
│   provider.fetch_security_price  → 成功则缓存并返回                    │
│     ↓ 失败/异常                                                       │
│   返回 nil                                                           │
└───────────────────────────┬─────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────────────┐
│                         前端页面展示                                  │
│                                                                     │
│   current_price != nil                                              │
│     → 持仓详情: 显示具体价格 ($100.00)                               │
│     → 交易详情: 显示价格 + 未实现收益趋势                             │
│                                                                     │
│   current_price == nil                                              │
│     → 持仓详情: 显示 "Unknown"                                      │
│     → 交易详情: 价格区块完全不渲染                                    │
│                                                                     │
│   异常路径特殊情况:                                                  │
│   → 若有历史价格缓存，显示过时价格（用户无感知）                       │
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

**注意**: 当前测试**未覆盖 Provider 抛异常的场景**（路径 3）。

---

## 10. 设计特点总结

### 优点
1. **渐进式容错**: 连续 5 次失败才标记离线，避免偶发网络问题导致误判
2. **数据保护**: 标记离线时删除历史价格，防止错误数据影响估值
3. **优先级调度**: 未检查过的证券优先处理，确保新数据及时验证
4. **批量控制**: 每日 1000 个限制，避免性能瓶颈
5. **异常隔离**: Provider 异常不影响状态，保留历史数据

### 潜在改进点

#### 1. 异常路径的隐蔽问题
- **问题**: Provider 抛异常时，`failed_fetch_count` 不递增，但 `last_health_check_at` 被更新
- **影响**: 
  - 连续异常不会触发离线保护
  - 7 天检查周期被重置，问题可能被掩盖
  - 用户可能看到过时的价格数据
- **建议**: 
  - 区分"Provider 返回错误"和"Provider 抛异常"两种失败类型
  - 异常场景也应该递增某种失败计数（如 `exception_count`）
  - 或考虑异常场景不更新 `last_health_check_at`，让证券尽快重新检查

#### 2. 前端反馈不直观
- **问题**: 用户看到"Unknown"价格，但不知道原因是健康检查失败
- **建议**: 前端可展示 `offline` 状态，提示用户"该证券当前不可用"

#### 3. 无通知机制
- **问题**: 证券离线时没有主动通知用户或管理员
- **建议**: 重要证券离线时发送邮件/系统通知

#### 4. 离线恢复机制
- **问题**: 离线证券恢复在线后，历史价格已被删除，需要重新导入
- **建议**: 恢复在线时自动触发一次完整的价格导入

#### 5. 测试覆盖不足
- **问题**: 缺少 Provider 抛异常场景的测试
- **建议**: 添加异常路径的单元测试

---

## 11. 关键文件索引

| 文件路径 | 职责 |
|----------|------|
| `config/schedule.yml` | 定时任务配置 |
| `app/jobs/security_health_check_job.rb` | 任务入口 |
| `app/models/security/health_checker.rb` | 健康检查核心逻辑（含异常处理） |
| `app/models/security.rb` | Security 模型定义，current_price 方法 |
| `app/models/security/provided.rb` | find_or_fetch_price 实现 |
| `app/models/security/resolver.rb` | 证券解析与离线标记 |
| `app/models/security/price/importer.rb` | 价格导入逻辑 |
| `app/models/market_data_importer.rb` | 批量价格导入（受 offline 状态影响） |
| `app/models/holding.rb` | 持仓模型 |
| `app/models/trade.rb` | 交易模型（unrealized_gain_loss） |
| `app/views/holdings/show.html.erb` | 持仓详情页面 |
| `app/views/trades/_header.html.erb` | 交易详情页面 |
| `db/schema.rb` | 数据库表结构 |
| `test/models/security/health_checker_test.rb` | 测试用例 |
