# Security 健康检查系统报告

## 1. 概述

本报告详细阐述了 Maybe 金融应用中的证券（Security）健康检查系统，重点关注：
- "调用异常"与"返回失败响应"两条链路的严格区分
- 各状态字段在不同场景下的真实变化
- 从状态变化 → 价格读取 → 页面展示的完整前端反馈链路
- 按"显示旧价格 / Unknown / 直接报错"三种结果分类的触发条件

---

## 2. 关键概念澄清

### 2.1 什么是"调用异常"？

**定义**: Provider 调用过程中抛出未捕获的 Ruby 异常

**触发场景**:
- 网络超时 (`Net::ReadTimeout`)
- DNS 解析失败 (`SocketError`)
- API 限流/认证失败 (`Faraday::ClientError`)
- Provider 内部抛出 `StandardError` 或其子类

**代码捕获点** (`app/models/security/health_checker.rb:57-60`):
```ruby
rescue => e
  Sentry.capture_exception(e) do |scope|
    scope.set_tags(security_id: @security.id)
  end
end
```

---

### 2.2 什么是"返回失败响应"？

**定义**: Provider 调用成功完成（无异常抛出），但返回的响应对象 `success? == false`

**触发场景**:
- Provider 正常返回错误响应（如 HTTP 404、429）
- Provider 响应成功但价格数据为空 (`response.data.price.nil?`)

**代码处理点** (`app/models/security/health_checker.rb:52-56`, `72-84`):
```ruby
if latest_provider_price
  handle_success
else
  handle_failure
end

def latest_provider_price
  return nil unless provider.present?

  response = provider.fetch_security_price(...)
  return nil unless response.success?  # 失败响应 → 返回 nil

  response.data.price  # 价格为空 → 也返回 nil
end
```

---

## 3. 定时任务调度

### 3.1 调度配置

**位置**: `config/schedule.yml:16-20`

**调度规则**:
- **Cron 表达式**: `0 2 * * 1-5`
- **执行时间**: 每周一至周五 凌晨 2:00 AM EST / 3:00 AM EDT
- **任务类**: `SecurityHealthCheckJob`

### 3.2 批量处理策略

**位置**: `app/models/security/health_checker.rb:18-42`

| 配置项 | 值 | 说明 |
|--------|-----|------|
| `DAILY_BATCH_SIZE` | 1000 | 每天最多检查 1000 个已检查过的证券 |
| `HEALTH_CHECK_INTERVAL` | 7 天 | 已检查过的证券 7 天后才需要重新检查 |
| `MAX_CONSECUTIVE_FAILURES` | 5 次 | 连续失败 5 次标记为离线 |

**优先级排序**:
1. **从未检查过的证券** (`last_health_check_at: nil`) → 无数量限制
2. **到期检查的证券** (超过 7 天未检查) → 每日最多 1000 个

---

## 4. 健康检查两条链路的严格对比

### 4.1 run_check 完整执行流程

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
  # ── 路径 A：调用异常 ──
  # 只上报 Sentry，不更新状态字段
  Sentry.capture_exception(e) do |scope|
    scope.set_tags(security_id: @security.id)
  end
ensure
  # ── 始终执行 ──
  # 无论成功/失败/异常，last_health_check_at 都会更新
  security.update!(last_health_check_at: Time.current)
end
```

**核心区分点**:
- `latest_provider_price` 返回 `nil` → 进入 **`handle_failure`**（路径 B）
- `latest_provider_price` 执行中抛异常 → 进入 **`rescue`**（路径 A）
- 两条路径都会执行 `ensure` 块

---

### 4.2 路径 A：Provider 调用异常

**触发条件**:
- `provider.fetch_security_price` 内部抛出异常（网络超时、DNS 失败等）
- 异常在 `rescue => e` 块被捕获

**执行流程**:
```
latest_provider_price 执行中抛出异常
  → rescue: Sentry.capture_exception（仅上报）
  → ensure: last_health_check_at = Time.current
  → handle_success 和 handle_failure 都 不执行
```

**状态字段变化**（关键！）:

| 字段 | 变化 | 原因 |
|------|------|------|
| `offline` | **保持不变** | `handle_failure` 未执行 |
| `failed_fetch_count` | **保持不变** | `handle_failure` 未执行 |
| `failed_fetch_at` | **保持不变** | `handle_failure` 未执行 |
| `last_health_check_at` | **更新为当前时间** | `ensure` 块始终执行 |

**异常路径的隐蔽特性**:
1. **异常不计入失败计数**: `failed_fetch_count` 不递增
2. **异常不会触发离线保护**: `offline` 保持不变
3. **异常不会删除价格**: 历史价格数据保留
4. **异常重置检查周期**: `last_health_check_at` 更新，7 天后才会再次检查

---

### 4.3 路径 B：Provider 返回失败响应

**触发条件**（二选一）:
- `response.success? == false`（Provider 正常返回错误响应）
- `response.data.price.nil?`（响应成功但无价格数据）

**执行流程**:
```
latest_provider_price 返回 nil（无异常抛出）
  → handle_failure
  → ensure: last_health_check_at = Time.current
```

**状态字段变化**（分阶段）:

**阶段 B1：1-5 次连续失败** (`failed_fetch_count < 5`)

| 字段 | 变化 |
|------|------|
| `offline` | 保持不变（仍为 `false`） |
| `failed_fetch_count` | `n` → `n + 1`（递增） |
| `failed_fetch_at` | 更新为当前时间 |
| `last_health_check_at` | 更新为当前时间 |

**阶段 B2：第 6 次及以上连续失败** (`failed_fetch_count >= 5`)

| 字段 | 变化 |
|------|------|
| `offline` | `false` → `true`（标记离线） |
| `failed_fetch_count` | `5` → `6`（设置为 MAX+1） |
| `failed_fetch_at` | 更新为当前时间 |
| `last_health_check_at` | 更新为当前时间 |
| `security_prices` 表 | **所有记录被删除** |

**handle_failure 代码**:
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

### 4.4 两条链路状态字段对比表

| 字段 | 路径 A（调用异常） | 路径 B（失败响应 1-5 次） | 路径 B（失败响应 ≥6 次） |
|------|-------------------|------------------------|-----------------------|
| `offline` | **不变** | 不变 | `false → true` |
| `failed_fetch_count` | **不变** | `n → n+1` | `5 → 6` |
| `failed_fetch_at` | **不变** | 更新 | 更新 |
| `last_health_check_at` | 更新 | 更新 | 更新 |
| 价格数据 | **保留** | 保留 | **删除** |
| 下次检查时间 | 7 天后 | 7 天后 | 7 天后（但 offline 可能被跳过） |

---

## 5. 从状态字段到价格读取的真实行为差异

### 5.1 影响层 1：MarketDataImporter 价格导入

**位置**: `app/models/market_data_importer.rb:25-34`

```ruby
def import_security_prices
  Security.online.find_each do |security|  # 只导入 offline=false 的证券
    security.import_provider_prices(...)
  end
end
```

**关键影响**:
- 两条链路都执行在健康检查任务（凌晨 2:00 AM）
- 价格导入任务执行在**晚些时候**（凌晨 10:00 PM），见 `config/schedule.yml:1-5`
- `offline` 状态决定是否被纳入导入列表

**路径 A（异常）的影响**:
- `offline` 保持不变（假设为 `false`）
- **仍然会被** `MarketDataImporter` 尝试导入新价格
- 历史价格数据保留

**路径 B（失败响应 ≥6 次）的影响**:
- `offline` 被设为 `true`
- **被跳过**，不会被导入新价格
- 历史价格数据已被删除

---

### 5.2 影响层 2：价格导入器行为

**位置**: `app/models/security/price/importer.rb:71-94`

```ruby
def provider_prices
  @provider_prices ||= begin
    response = security_provider.fetch_security_prices(...)

    if response.success?
      response.data.index_by(&:date)
    else
      # 失败响应：记录警告，返回空哈希
      Rails.logger.warn(...)
      Sentry.capture_exception(...)
      {}  # 返回空，不抛异常
    end
  end
end
```

**注意**: 价格导入器**没有 `rescue` 块**捕获异常。如果调用抛异常，会导致整个导入任务失败（可能影响其他证券）。

---

### 5.3 影响层 3：current_price 方法

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
  price = prices.find_by(date: date)  # 第一步：查数据库

  return price if price.present?      # 有缓存则直接返回

  # 第二步：数据库没有，尝试从 Provider 获取
  return nil unless provider.present?
  response = provider.fetch_security_price(...)

  return nil unless response.success?  # 失败响应 → 返回 nil

  # 成功则缓存并返回
  price = response.data
  Security::Price.find_or_create_by!(...) if cache
  price
end
```

**current_price 返回决策树**:

```
current_price 被调用
  ↓
find_or_fetch_price
  ↓
┌─────────────────────────────────────┐
│ 1. prices.find_by(date: today)      │
│    数据库有今日价格？                 │
└──────┬──────────────────────────────┘
       │ 是                          │ 否
       ↓                            ↓
┌─────────────┐            ┌─────────────────────────────┐
│ 返回数据库  │            │ 2. provider.present?        │
│ 价格（旧或新）│            │    Provider 已配置？         │
└─────────────┘            └──────┬──────────────────────┘
                                 │ 是                     │ 否
                                 ↓                        ↓
                      ┌─────────────────────┐       ┌─────────┐
                      │ 3. 调用 Provider     │       │ 返回 nil│
                      │ fetch_security_price│       └─────────┘
                      └──────────┬──────────┘
                                 │
                    ┌────────────┴────────────┐
                    │ 调用是否抛异常？          │
                    └─────┬──────────────────┬─┘
                          │ 是               │ 否
                          ↓                  ↓
                   ┌───────────┐    ┌────────────────────────┐
                   │ 异常上冒泡 │    │ 4. response.success?    │
                   │ 到上层    │    │    返回成功？            │
                   └───────────┘    └──────────┬─────────────┘
                                                │ 是      │ 否
                                                ↓        ↓
                                         ┌─────────┐ ┌─────────┐
                                         │缓存并返回│ │ 返回 nil│
                                         │新价格   │ │         │
                                         └─────────┘ └─────────┘
```

**关键发现**:
1. `find_or_fetch_price` **没有 `rescue` 块**
2. 如果 Provider 调用抛异常，异常会**向上冒泡**到上层
3. 只有"返回失败响应"（`response.success? == false`）才会优雅地返回 `nil`

---

## 6. 前端三种可见结果的触发条件

### 6.1 结果一：显示旧价格

**定义**: 前端页面显示价格数据，但该价格可能已过时

**触发条件矩阵**:

| 条件层 | 路径 A（调用异常） | 路径 B（失败响应 1-5 次） | 路径 B（失败响应 ≥6 次） |
|--------|-------------------|------------------------|-----------------------|
| 健康检查阶段 | `offline` 保持 `false` | `offline` 保持 `false` | `offline` 设为 `true` |
| 健康检查阶段 | 价格数据**保留** | 价格数据**保留** | 价格数据**被删除** |
| 价格导入阶段 | 可能被导入（若 `offline=false`） | 可能被导入（若 `offline=false`） | **被跳过** |
| `security_prices` 表 | 有历史价格 | 有历史价格 | **无价格** |
| `find_or_fetch_price` | 第一步命中缓存 → 返回旧价格 | 第一步命中缓存 → 返回旧价格 | 第一步无缓存 |

**完整链路（以路径 A 为例）**:

```
健康检查时 Provider 抛异常
  ↓
offline 保持 false
failed_fetch_count 不变
last_health_check_at 更新
价格数据保留（关键！）
  ↓
MarketDataImporter 运行（晚上 10:00 PM）
  由于 offline=false，该证券仍在导入列表中
  若导入时 Provider 也异常 → 无新价格
  但旧价格数据仍在 security_prices 表中
  ↓
用户访问持仓详情页
  ↓
current_price 被调用
  ↓
find_or_fetch_price
  ↓
prices.find_by(date: today) → 命中旧价格缓存
  ↓
返回旧价格（可能已过时多天）
  ↓
前端显示: "$98.50"（用户不知道这是旧价格）
```

**前端页面展示**:
- **持仓详情页**: 显示具体价格（如 `$98.50`）
- **交易详情页**: 显示价格 + 未实现收益趋势

**用户感知**:
- 价格看起来正常
- 但实际上可能已过时（Provider 已不可用多天）
- 收益计算基于过时价格，可能不准确

---

### 6.2 结果二：显示 "Unknown"

**定义**: 前端页面显示国际化文本 "Unknown" 或相关区块消失

**国际化来源** (`config/locales/views/holdings/en.yml:37`):
```yaml
en:
  holdings:
    show:
      unknown: Unknown
```

**触发条件矩阵**:

| 条件层 | 路径 B（失败响应 ≥6 次） | 路径 A（无历史价格） | 其他路径（数据库无缓存） |
|--------|-----------------------|---------------------|----------------------|
| 健康检查阶段 | `offline` 设为 `true` | `offline` 保持不变 | - |
| 健康检查阶段 | 价格**被删除** | 价格数据**保留**（但可能无今日数据） | - |
| 价格导入阶段 | **被跳过**（offline=true） | 可能被导入 | - |
| `security_prices` 表 | **无任何价格** | 可能有旧数据但无今日数据 | 无今日数据 |
| `find_or_fetch_price` | 第一步无缓存 → 尝试 Provider | 第一步无今日缓存 → 尝试 Provider | 第一步无今日缓存 |
| Provider 尝试 | 返回失败响应 → `nil` | 返回失败响应或抛异常 | 返回失败响应 → `nil` |

**完整链路（以路径 B2 为例）**:

```
健康检查连续失败 6 次
  ↓
offline 设为 true
failed_fetch_count 设为 6
prices.delete_all（所有历史价格被删除！）
last_health_check_at 更新
  ↓
MarketDataImporter 运行
  由于 offline=true，该证券被跳过
  无新价格导入
  security_prices 表仍为空
  ↓
用户访问持仓详情页
  ↓
current_price 被调用
  ↓
find_or_fetch_price
  ↓
prices.find_by(date: today) → nil（无任何价格）
  ↓
调用 provider.fetch_security_price
  ↓
response.success? == false → 返回 nil
  ↓
current_price 返回 nil
  ↓
前端处理:
  持仓详情页: <%= t(".unknown") %> → 显示 "Unknown"
  交易详情页: <% if current_price.present? %> → 条件不满足，区块不渲染
```

**前端页面展示**:

**持仓详情页** (`app/views/holdings/show.html.erb:22-25`):
```erb
<dd class="text-primary">
  <%= @holding.security.current_price ? format_money(@holding.security.current_price) : t(".unknown") %>
</dd>
```
→ **显示**: `Unknown`

**交易详情页** (`app/views/trades/_header.html.erb:55-69`):
```erb
<% if trade.security.current_price.present? %>
  <div>当前市价: ...</div>
  <% if trade.unrealized_gain_loss.present? %>
    <div>总收益: ...</div>
  <% end %>
<% end %>
```
→ **显示**: 当前市价和总收益区块**完全不渲染**

**用户感知**:
- 持仓详情：市价显示为 "Unknown"
- 交易详情：缺少当前市价和收益信息
- 投资组合估值可能不准确（依赖其他计算方式）

---

### 6.3 结果三：直接报错

**定义**: 用户访问页面时，应用抛出未处理的异常，显示错误页面

**触发条件（关键！）**:

`find_or_fetch_price` **没有 `rescue` 块**。如果用户访问页面时 Provider 调用抛异常，异常会向上冒泡到 Rails 框架，导致：
- 开发环境：显示异常堆栈和错误信息
- 生产环境：显示通用 500 错误页面

**触发链路**:

```
用户访问持仓详情页
  ↓
控制器查询 @holding 和相关数据
  ↓
视图渲染时调用 @holding.security.current_price
  ↓
find_or_fetch_price(date: today)
  ↓
prices.find_by(date: today) → nil（数据库无今日缓存）
  ↓
调用 provider.fetch_security_price(...)
  ↓
┌─────────────────────────────────────────────────────────────────┐
│ Provider 调用抛异常（网络超时、DNS 失败等）                        │
│                                                                 │
│ 注意：find_or_fetch_price 没有 rescue 块！                        │
│                                                                 │
│ 异常向上冒泡 → 上层 → Rails 框架 → 错误页面                       │
└─────────────────────────────────────────────────────────────────┘
  ↓
用户看到：
  开发环境: 异常堆栈页面（红色错误页面）
  生产环境: "Something went wrong" 500 错误页面
```

**代码证据**:

`app/models/security/provided.rb:39-62`:
```ruby
def find_or_fetch_price(date: Date.current, cache: true)
  price = prices.find_by(date: date)
  return price if price.present?

  return nil unless provider.present?
  
  # ⚠️ 下面这行调用如果抛异常，没有 rescue 保护！
  response = provider.fetch_security_price(
    symbol: ticker,
    exchange_operating_mic: exchange_operating_mic,
    date: date
  )

  return nil unless response.success?

  price = response.data
  Security::Price.find_or_create_by!(...) if cache
  price
end
```

**对比：健康检查器有 rescue，视图层没有**

| 位置 | 是否有 `rescue` 块 | 异常行为 |
|------|------------------|---------|
| `HealthChecker.run_check` | **有** | 捕获异常，上报 Sentry，继续执行 |
| `find_or_fetch_price` | **没有** | 异常向上冒泡，导致页面报错 |

---

### 6.4 三种结果对比汇总

| 结果 | 健康检查阶段 | 价格数据状态 | current_price 返回 | 前端展示 |
|------|------------|-------------|-------------------|---------|
| **显示旧价格** | 路径 A 或 路径 B1 | 有历史价格缓存 | Money 对象（旧价格） | `$98.50`（正常显示） |
| **显示 Unknown** | 路径 B2 或 无缓存 | 无今日价格 | `nil` | `Unknown` 或 区块消失 |
| **直接报错** | 任意路径（视图层调用时异常） | 无今日缓存 | **异常上冒泡** | 500 错误页面 |

| 结果 | 用户感知 | 实际问题严重程度 |
|------|---------|----------------|
| **显示旧价格** | 看起来正常 | **隐蔽严重**（数据已过时） |
| **显示 Unknown** | 知道有问题 | 中度（用户知道数据不可用） |
| **直接报错** | 页面崩溃 | 严重（影响用户体验） |

---

## 7. 端到端流程图（完整版）

```
┌─────────────────────────────────────────────────────────────────────┐
│                     健康检查任务（凌晨 2:00 AM）                       │
└─────────────────────────────────────────────────────────────────────┘
                          │
          ┌───────────────┴───────────────┐
          ↓                               ↓
    ┌─────────────┐                 ┌─────────────┐
    │ 路径 A      │                 │ 路径 B      │
    │ 调用异常    │                 │ 返回失败响应│
    └──────┬──────┘                 └──────┬──────┘
           ↓                               ↓
┌─────────────────────────────┐   ┌─────────────────────────────┐
│ offline: 保持不变            │   │ 1-5次:                      │
│ failed_fetch_count: 保持不变  │   │   offline: 保持 false       │
│ failed_fetch_at: 保持不变     │   │   failed_fetch_count: +1    │
│ last_health_check_at: 更新   │   │   failed_fetch_at: 更新      │
│ 价格数据: 保留               │   │   价格数据: 保留             │
│                             │   │                             │
│                             │   │ ≥6次:                       │
│                             │   │   offline: true            │
│                             │   │   failed_fetch_count: 6    │
│                             │   │   价格数据: 删除            │
└──────────────┬──────────────┘   └──────────────┬──────────────┘
               ↓                                  ↓
┌─────────────────────────────────────────────────────────────────────┐
│                 价格导入任务（晚上 10:00 PM）                         │
│                                                                     │
│ Security.online.find_each                                           │
│   → offline=false 的证券尝试导入新价格                               │
│   → offline=true 的证券被跳过                                       │
└─────────────────────────────────────────────────────────────────────┘
                          │
          ┌───────────────┴───────────────┐
          ↓                               ↓
┌─────────────────────┐         ┌─────────────────────┐
│ 路径 A:             │         │ 路径 B2:            │
│ offline=false       │         │ offline=true        │
│ 旧价格仍在数据库     │         │ 价格已被删除         │
│ 可能尝试导入新价格   │         │ 被跳过，无新价格     │
└──────────┬──────────┘         └──────────┬──────────┘
           ↓                               ↓
┌─────────────────────────────────────────────────────────────────────┐
│                      用户访问页面（任意时间）                          │
│                                                                     │
│ current_price 被调用                                                 │
│   → find_or_fetch_price                                              │
│     → 第一步：查数据库缓存                                           │
│     → 第二步：缓存不存在则调用 Provider                               │
└─────────────────────────────────────────────────────────────────────┘
                          │
        ┌─────────────────┼─────────────────┐
        ↓                 ↓                 ↓
┌───────────────┐ ┌───────────────┐ ┌───────────────┐
│ 第一步命中缓存 │ │ 第一步无缓存   │ │ 第一步无缓存   │
│ 返回旧价格     │ │ 第二步成功    │ │ 第二步异常    │
└───────┬───────┘ └───────┬───────┘ └───────┬───────┘
        ↓                 ↓                 ↓
┌───────────────┐ ┌───────────────┐ ┌───────────────┐
│ 显示旧价格     │ │ 显示 Unknown  │ │ 直接报错      │
│ $98.50        │ │ Unknown       │ │ 500 页面      │
└───────────────┘ └───────────────┘ └───────────────┘
```

---

## 8. 数据模型与状态字段

### 8.1 数据库表结构

**位置**: `db/schema.rb:623-640`

```ruby
create_table "securities", force: :cascade do |t|
  t.string "ticker", null: false
  t.boolean "offline", default: false, null: false       # 核心状态
  t.datetime "failed_fetch_at"                           # 最后失败时间
  t.integer "failed_fetch_count", default: 0, null: false # 连续失败计数
  t.datetime "last_health_check_at"                      # 最后检查时间
end
```

### 8.2 状态字段更新时机对照表

| 字段 | 路径 A（异常） | 路径 B1（失败 1-5 次） | 路径 B2（失败 ≥6 次） | 成功 |
|------|---------------|---------------------|--------------------|------|
| `offline` | 不变 | 不变 | `false → true` | 设为 `false` |
| `failed_fetch_count` | **不变** | `+1` | 设为 `6` | 设为 `0` |
| `failed_fetch_at` | **不变** | 更新 | 更新 | 设为 `nil` |
| `last_health_check_at` | 更新 | 更新 | 更新 | 更新 |

---

## 9. 关键发现与潜在问题

### 9.1 关键发现总结

1. **两条链路的本质区别**
   - 路径 A（调用异常）：`rescue` 块捕获，状态字段**几乎不变**
   - 路径 B（失败响应）：`handle_failure` 执行，状态字段**有变化**

2. **`last_health_check_at` 的特殊行为**
   - `ensure` 块确保**无论什么情况**都会更新
   - 这意味着异常也会重置 7 天检查周期

3. **异常路径的隐蔽性**
   - 连续多次异常不会触发离线保护
   - 用户可能看到过时的价格数据
   - 只有开发人员通过 Sentry 知道问题

4. **视图层无异常保护**
   - `find_or_fetch_price` 没有 `rescue` 块
   - 用户访问页面时 Provider 异常会导致 500 错误

### 9.2 潜在改进建议

#### 建议 1：统一异常和失败响应的处理
- **问题**: 调用异常不计入失败计数，但失败响应会
- **建议**:
  ```ruby
  rescue => e
    Sentry.capture_exception(e) do |scope|
      scope.set_tags(security_id: @security.id)
    end
    handle_failure  # 异常也应该走失败处理逻辑
  ```

#### 建议 2：异常场景不更新 `last_health_check_at`
- **问题**: 异常会重置 7 天周期，问题可能被掩盖
- **建议**: 将 `last_health_check_at` 更新移到 `ensure` 之外，或区分成功/异常

#### 建议 3：为 `find_or_fetch_price` 添加异常保护
- **问题**: 用户页面可能因 Provider 异常而崩溃
- **建议**:
  ```ruby
  def find_or_fetch_price(date: Date.current, cache: true)
    # ...
    begin
      response = provider.fetch_security_price(...)
    rescue => e
      Sentry.capture_exception(e)
      return nil  # 优雅降级，返回 nil 而不是抛异常
    end
    # ...
  end
  ```

#### 建议 4：前端展示 `offline` 状态
- **问题**: 用户看到 "Unknown" 但不知道原因
- **建议**: 前端可展示 `offline` 状态，提示 "该证券当前不可用"

#### 建议 5：添加异常路径的测试
- **问题**: 当前测试未覆盖 Provider 抛异常的场景
- **建议**: 添加单元测试验证异常路径的行为

---

## 10. 测试覆盖现状

**位置**: `test/models/security/health_checker_test.rb`

| 测试用例 | 覆盖路径 | 状态 |
|----------|---------|------|
| `failure incrementor increases for each health check failure` | 路径 B1 | ✅ 已覆盖 |
| `after enough consecutive health check failures, security goes offline` | 路径 B2 | ✅ 已覆盖 |
| `failure incrementor resets to 0 when health check succeeds` | 成功路径 | ✅ 已覆盖 |
| **Provider 抛异常场景** | 路径 A | ❌ **未覆盖** |
| **视图层异常场景** | `find_or_fetch_price` 异常 | ❌ **未覆盖** |

---

## 11. 关键文件索引

| 文件路径 | 职责 |
|----------|------|
| `config/schedule.yml` | 定时任务配置 |
| `app/jobs/security_health_check_job.rb` | 任务入口 |
| `app/models/security/health_checker.rb` | 健康检查核心逻辑（含 rescue/ensure） |
| `app/models/security.rb` | Security 模型，`current_price` 方法 |
| `app/models/security/provided.rb` | `find_or_fetch_price` 实现（**无 rescue**） |
| `app/models/security/resolver.rb` | 证券解析 |
| `app/models/security/price/importer.rb` | 价格导入逻辑 |
| `app/models/market_data_importer.rb` | 批量价格导入（受 offline 状态影响） |
| `app/models/holding.rb` | 持仓模型 |
| `app/models/trade.rb` | 交易模型 |
| `app/views/holdings/show.html.erb` | 持仓详情页面（显示 Unknown） |
| `app/views/trades/_header.html.erb` | 交易详情页面 |
| `config/locales/views/holdings/en.yml` | 国际化文件（Unknown 定义） |
| `db/schema.rb` | 数据库表结构 |
| `test/models/security/health_checker_test.rb` | 测试用例（缺异常场景） |
