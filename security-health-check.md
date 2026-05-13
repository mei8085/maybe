# Security 健康检查系统报告

## 1. 概述

本报告基于代码精确分析了 Maybe 金融应用中的证券健康检查系统，重点区分：
- **健康检查任务**（后台定时任务）中的异常处理逻辑
- **页面请求**（用户访问时）中 `current_price` 实时取价的执行路径
- 两条路径的异常处理差异
- "直接报错"的精确触发条件

---

## 2. 两条执行路径的核心区分

### 2.1 路径一：健康检查任务（后台定时任务）

**执行时机**: 每周一至周五 凌晨 2:00 AM EST

**入口文件**: `app/models/security/health_checker.rb:49-63`

```ruby
def run_check
  Rails.logger.info("Running health check for #{security.ticker}")

  if latest_provider_price
    handle_success
  else
    handle_failure
  end
rescue => e
  # ── 异常被捕获 ──
  Sentry.capture_exception(e) do |scope|
    scope.set_tags(security_id: @security.id)
  end
ensure
  # ── 始终执行 ──
  security.update!(last_health_check_at: Time.current)
end
```

**关键特征**:
- `rescue => e` 块捕获所有异常
- 异常不会冒泡到任务上层
- `ensure` 块确保 `last_health_check_at` 始终更新

---

### 2.2 路径二：页面请求时的 `current_price`（用户访问时）

**执行时机**: 用户访问持仓详情页、交易详情页等页面时

**入口链条**:
```
app/views/holdings/show.html.erb
  → @holding.security.current_price
    → app/models/security.rb:14-18
      → find_or_fetch_price (Provided 模块)
        → app/models/security/provided.rb:39-62
```

**关键代码**:

`app/models/security.rb:14-18`:
```ruby
def current_price
  @current_price ||= find_or_fetch_price
  return nil if @current_price.nil?
  Money.new(@current_price.price, @current_price.currency)
end
```

`app/models/security/provided.rb:39-62`:
```ruby
def find_or_fetch_price(date: Date.current, cache: true)
  price = prices.find_by(date: date)

  return price if price.present?

  # Make sure we have a data provider before fetching
  return nil unless provider.present?
  response = provider.fetch_security_price(
    symbol: ticker,
    exchange_operating_mic: exchange_operating_mic,
    date: date
  )
  # ⚠️ 关键：这里没有 rescue 块！

  return nil unless response.success?

  price = response.data
  Security::Price.find_or_create_by!(...) if cache
  price
end
```

**关键特征**:
- **没有** `rescue` 块
- 如果 `provider.fetch_security_price` 抛异常，异常会**向上冒泡**
- 最终可能导致页面渲染错误（500）

---

## 3. 健康检查任务的异常处理

### 3.1 健康检查任务中的两条子路径

#### 子路径 1A：Provider 返回失败响应

**定义**: `provider.fetch_security_price` 正常返回（无异常），但 `response.success? == false`

**代码证据** (`health_checker.rb:72-84`):
```ruby
def latest_provider_price
  return nil unless provider.present?

  response = provider.fetch_security_price(...)

  return nil unless response.success?  # ← 返回 nil，不抛异常

  response.data.price
end
```

**测试证据** (`test/support/provider_test_helper.rb:10-16`):
```ruby
def provider_error_response(error)
  Provider::Response.new(
    success?: false,  # ← 成功响应对象，只是 success? 为 false
    data: nil,
    error: error
  )
end
```

**状态变化**:
| 字段 | 变化 |
|------|------|
| `offline` | 1-5次失败：不变；≥6次：`false → true` |
| `failed_fetch_count` | `+1`，≥6次设为 `6` |
| `failed_fetch_at` | 更新为当前时间 |
| `last_health_check_at` | 更新为当前时间 |
| 价格数据 | ≥6次时被删除 |

---

#### 子路径 1B：Provider 调用抛异常

**定义**: `provider.fetch_security_price` 执行过程中抛出 Ruby 异常

**代码证据** (`health_checker.rb:57-63`):
```ruby
rescue => e
  Sentry.capture_exception(e) do |scope|
    scope.set_tags(security_id: @security.id)
  end
ensure
  security.update!(last_health_check_at: Time.current)
end
```

**状态变化**:
| 字段 | 变化 | 原因 |
|------|------|------|
| `offline` | **不变** | `handle_failure` 未执行 |
| `failed_fetch_count` | **不变** | `handle_failure` 未执行 |
| `failed_fetch_at` | **不变** | `handle_failure` 未执行 |
| `last_health_check_at` | **更新** | `ensure` 块执行 |
| 价格数据 | **保留** | `convert_to_offline_security!` 未执行 |

---

### 3.2 健康检查任务状态对比汇总

| 字段 | 子路径 1A（失败响应）1-5次 | 子路径 1A（失败响应）≥6次 | 子路径 1B（调用异常） | 成功 |
|------|------------------------|-----------------------|---------------------|------|
| `offline` | 不变 | `false → true` | **不变** | 设为 `false` |
| `failed_fetch_count` | `+1` | 设为 `6` | **不变** | 设为 `0` |
| `failed_fetch_at` | 更新 | 更新 | **不变** | 设为 `nil` |
| `last_health_check_at` | 更新 | 更新 | 更新 | 更新 |
| 价格数据 | 保留 | **删除** | **保留** | 保留 |

---

## 4. 页面请求时 `current_price` 的执行路径

### 4.1 执行流程图

```
用户访问页面
  ↓
视图渲染调用 @holding.security.current_price
  ↓
app/models/security.rb:14-18
def current_price
  @current_price ||= find_or_fetch_price
  return nil if @current_price.nil?
  Money.new(...)
end
  ↓
app/models/security/provided.rb:39-62
def find_or_fetch_price(date: Date.current, cache: true)
  price = prices.find_by(date: date)  ← 第一步：查数据库

  return price if price.present?      ← 有缓存则直接返回

  # 第二步：数据库没有，尝试 Provider
  return nil unless provider.present?

  # ⚠️ 没有 rescue！
  response = provider.fetch_security_price(...)

  return nil unless response.success?  ← 失败响应返回 nil

  # 成功则缓存并返回
  ...
end
```

### 4.2 三条子路径

#### 子路径 2A：数据库有今日价格缓存

**触发条件**: `prices.find_by(date: today)` 找到记录

**代码位置**: `provided.rb:40-42`

**执行结果**:
- 直接返回数据库中的价格
- 不调用 Provider
- 页面正常显示价格

---

#### 子路径 2B：数据库无缓存，Provider 返回失败响应

**触发条件**:
- `prices.find_by(date: today)` 返回 `nil`
- `provider.fetch_security_price` 返回 `response.success? == false`

**代码位置**: `provided.rb:46-52`

**执行结果**:
```ruby
response = provider.fetch_security_price(...)
return nil unless response.success?  # ← 返回 nil，不抛异常
```

**返回形态**: `find_or_fetch_price` 返回 `nil`

**上游处理** (`security.rb:14-18`):
```ruby
def current_price
  @current_price ||= find_or_fetch_price  # nil
  return nil if @current_price.nil?       # 返回 nil
  Money.new(...)
end
```

**最终返回**: `current_price` 返回 `nil`

---

#### 子路径 2C：数据库无缓存，Provider 调用抛异常

**触发条件**:
- `prices.find_by(date: today)` 返回 `nil`
- `provider.fetch_security_price` 执行中抛出异常

**关键发现**: `find_or_fetch_price` **没有 `rescue` 块**

**代码证据** (`provided.rb:39-62`):
```ruby
def find_or_fetch_price(date: Date.current, cache: true)
  price = prices.find_by(date: date)
  return price if price.present?

  return nil unless provider.present?

  # ⚠️ 这里没有 rescue 保护！
  response = provider.fetch_security_price(
    symbol: ticker,
    exchange_operating_mic: exchange_operating_mic,
    date: date
  )
  # 如果上面这行抛异常，异常会向上冒泡
  ...
end
```

**执行结果**:
- 异常从 `provider.fetch_security_price` 抛出
- `find_or_fetch_price` 没有捕获
- `current_price` 没有捕获
- 视图渲染时没有捕获
- 异常向上冒泡到 Rails 框架
- 生产环境：显示 500 错误页面
- 开发环境：显示异常堆栈

---

## 5. 前端三种可见结果的精确触发条件

### 5.1 结果一：显示价格（正常）

**触发条件（满足任一即可）**:

**条件 A**: 数据库有今日价格缓存
```
find_or_fetch_price
  → prices.find_by(date: today) → 非 nil
  → 直接返回数据库价格
  → current_price → Money 对象
  → 视图: format_money(price)
  → 显示: "$100.00"
```

**条件 B**: 数据库无缓存，Provider 调用成功
```
find_or_fetch_price
  → prices.find_by → nil
  → provider.fetch_security_price → 成功响应
  → response.success? == true
  → 缓存并返回价格
  → current_price → Money 对象
  → 显示: "$100.00"
```

---

### 5.2 结果二：显示 "Unknown"

**触发条件（必须同时满足）**:

1. 数据库无今日价格缓存
2. Provider 调用**正常返回**（无异常抛出）
3. `response.success? == false` 或价格数据为空

**执行链路**:
```
视图: @holding.security.current_price
  ↓
current_price 方法
  ↓
find_or_fetch_price
  ↓
prices.find_by(date: today) → nil
  ↓
provider.fetch_security_price(...)
  ↓
┌────────────────────────────────────────────┐
│ 正常返回（无异常）                          │
│ response.success? == false                 │
│                                            │
│ 注意：这不是异常！                          │
│ 这是 provider_error_response 的行为        │
└────────────────────────────────────────────┘
  ↓
return nil unless response.success?  → 返回 nil
  ↓
find_or_fetch_price → nil
  ↓
current_price → nil
  ↓
视图处理:
  持仓页: <%= current_price ? format_money(...) : t(".unknown") %>
         → 显示 "Unknown"
  交易页: <% if current_price.present? %> ... <% end %>
         → 条件不满足，区块不渲染
```

**证据链** (`test/support/provider_test_helper.rb:10-16`):
```ruby
def provider_error_response(error)
  Provider::Response.new(
    success?: false,  # ← 正常的 Response 对象，只是 success? 为 false
    data: nil,
    error: error
  )
end
```

---

### 5.3 结果三：直接报错（500 页面）

**触发条件（必须同时满足）**:

1. 数据库无今日价格缓存
2. `provider.fetch_security_price` **抛出 Ruby 异常**
3. 没有上层 `rescue` 块捕获

**精确执行链路（可复核）**:

#### 第一步：调用入口

**文件**: `app/models/security/provided.rb:39-62`

```ruby
def find_or_fetch_price(date: Date.current, cache: true)
  price = prices.find_by(date: date)  # ← 假设返回 nil

  return price if price.present?      # ← 不执行

  return nil unless provider.present? # ← 假设 provider 存在，不返回

  # ── 关键行 ──
  response = provider.fetch_security_price(
    symbol: ticker,
    exchange_operating_mic: exchange_operating_mic,
    date: date
  )
  # ↑ 如果这行抛异常...

  return nil unless response.success?  # ← 不会执行到这里
  ...
end
```

#### 第二步：关键分支

| 分支条件 | 代码位置 | 结果 |
|---------|---------|------|
| `provider.present? == false` | `provided.rb:45` | 返回 `nil`，显示 "Unknown" |
| `prices.find_by` 找到记录 | `provided.rb:40-42` | 返回缓存价格，正常显示 |
| `fetch_security_price` 正常返回 | `provided.rb:46-52` | 根据 `success?` 返回价格或 `nil` |
| `fetch_security_price` **抛异常** | `provided.rb:46-50` | **异常上冒泡** |

#### 第三步：返回形态

**异常场景**:
- `provider.fetch_security_price` 抛出 `Net::ReadTimeout`、`SocketError`、`Faraday::ClientError` 等
- `find_or_fetch_price` 没有 `rescue` 块
- 异常向上冒泡

#### 第四步：页面表现

- **开发环境**: 显示异常堆栈页面（红色错误页面）
- **生产环境**: 显示通用 500 错误页面

---

## 6. 最小证据链（可复核）

### 6.1 证据链 A：健康检查任务有 `rescue`

**调用入口**: `app/models/security/health_checker.rb:49`

```ruby
def run_check
  # ... 业务逻辑 ...
rescue => e          # ← 第 57 行：有 rescue
  Sentry.capture_exception(e)
ensure               # ← 第 61 行：有 ensure
  security.update!(last_health_check_at: Time.current)
end
```

**关键分支**:
- 第 52-56 行：`if latest_provider_price ... else ... handle_failure`
- 第 57-60 行：`rescue => e` 捕获异常
- 第 61-63 行：`ensure` 始终执行

**返回形态**:
- 成功：`handle_success` 执行
- 失败响应：`handle_failure` 执行
- 异常：`rescue` 捕获，上报 Sentry，不执行 `handle_failure`

**页面表现**:
- 健康检查是后台任务，不直接影响页面
- 间接影响：通过 `offline` 状态和价格数据影响后续页面渲染

---

### 6.2 证据链 B：页面请求无 `rescue`

**调用入口**: `app/models/security/provided.rb:39`

```ruby
def find_or_fetch_price(date: Date.current, cache: true)
  price = prices.find_by(date: date)
  return price if price.present?

  return nil unless provider.present?
  response = provider.fetch_security_price(...)  # ← 第 46 行
  # ⚠️ 这一行没有 rescue 保护！

  return nil unless response.success?
  ...
end
```

**关键分支**（自上而下一步步检查）:

| 步骤 | 代码位置 | 条件 | 结果 |
|------|---------|------|------|
| 1 | `provided.rb:40` | `price = prices.find_by(date: date)` | 有缓存 → 正常显示 |
| 2 | `provided.rb:45` | `provider.present?` | 无 provider → 返回 `nil` |
| 3 | `provided.rb:46-50` | `provider.fetch_security_price(...)` | **抛异常 → 500 页面** |
| 4 | `provided.rb:52` | `response.success?` | false → 返回 `nil` → "Unknown" |
| 5 | `provided.rb:54-61` | 成功 | 缓存并返回价格 → 正常显示 |

**返回形态**:
- 步骤 1、2、5：正常返回（价格或 `nil`）
- 步骤 4：正常返回 `nil`（不抛异常）
- **步骤 3**：**异常上冒泡**

**页面表现**:
- 步骤 1、5：显示价格（正常）
- 步骤 2、4：显示 "Unknown"
- **步骤 3**：显示 500 错误页面

---

## 7. 两条路径异常处理对比

| 对比项 | 健康检查任务（路径一） | 页面请求（路径二） |
|--------|---------------------|------------------|
| **入口方法** | `HealthChecker#run_check` | `Security#current_price` → `find_or_fetch_price` |
| **文件位置** | `app/models/security/health_checker.rb` | `app/models/security/provided.rb` |
| **`rescue` 块** | ✅ 有 (`health_checker.rb:57-60`) | ❌ **无** |
| **`ensure` 块** | ✅ 有 (`health_checker.rb:61-63`) | ❌ 无 |
| **异常行为** | 捕获后上报 Sentry，继续执行 | **异常上冒泡到 Rails** |
| **对用户影响** | 无直接影响（后台任务） | **可能导致 500 页面** |
| **失败响应处理** | `handle_failure` 执行，状态更新 | 返回 `nil`，显示 "Unknown" |

---

## 8. 端到端完整流程图

```
┌─────────────────────────────────────────────────────────────────────┐
│                     健康检查任务（凌晨 2:00 AM）                       │
│                      app/models/security/health_checker.rb            │
└─────────────────────────────────────────────────────────────────────┘
                          │
          ┌───────────────┴───────────────┐
          ↓                               ↓
    ┌─────────────┐                 ┌─────────────┐
    │ 失败响应    │                 │ 调用异常    │
    │ success?=false│                │ 抛 Exception │
    └──────┬──────┘                 └──────┬──────┘
           ↓                               ↓
┌─────────────────────────────┐   ┌─────────────────────────────┐
│ handle_failure 执行          │   │ rescue 捕获                 │
│                             │   │                             │
│ offline: false→true (≥6次)  │   │ offline: 不变               │
│ failed_count: +1            │   │ failed_count: 不变          │
│ 价格: 删除 (≥6次)            │   │ 价格: 保留                 │
│                             │   │                             │
│ ensure: last_health_check   │   │ ensure: last_health_check   │
└──────────────┬──────────────┘   └──────────────┬──────────────┘
               ↓                                  ↓
┌─────────────────────────────────────────────────────────────────────┐
│                 价格导入任务（晚上 10:00 PM）                         │
│                      app/models/market_data_importer.rb               │
│                                                                     │
│ Security.online.find_each                                           │
│   → offline=false 的证券尝试导入新价格                               │
│   → offline=true 的证券被跳过                                       │
└─────────────────────────────────────────────────────────────────────┘
                          │
                          ↓
┌─────────────────────────────────────────────────────────────────────┐
│                      用户访问页面（任意时间）                          │
│                                                                     │
│ 视图: @holding.security.current_price                               │
│   → app/models/security.rb:14-18                                    │
│     → find_or_fetch_price                                           │
│       → app/models/security/provided.rb:39-62                       │
└─────────────────────────────────────────────────────────────────────┘
                          │
        ┌─────────────────┼─────────────────┐
        ↓                 ↓                 ↓
┌───────────────┐ ┌───────────────┐ ┌───────────────┐
│ 有数据库缓存  │ │ 无缓存且      │ │ 无缓存且      │
│ 返回价格      │ │ Provider失败响应│ │ Provider抛异常 │
└───────┬───────┘ └───────┬───────┘ └───────┬───────┘
        ↓                 ↓                 ↓
┌───────────────┐ ┌───────────────┐ ┌───────────────┐
│ 显示价格      │ │ 显示 Unknown  │ │ 500 错误页面  │
│ $100.00       │ │               │ │               │
└───────────────┘ └───────────────┘ └───────────────┘
```

---

## 9. 状态字段更新时机

### 9.1 健康检查任务触发的状态更新

| 场景 | `offline` | `failed_fetch_count` | `last_health_check_at` | 价格数据 |
|------|-----------|---------------------|-----------------------|---------|
| 成功响应 | `false` | `0` | 更新 | 保留 |
| 失败响应 (1-5次) | 不变 | `+1` | 更新 | 保留 |
| 失败响应 (≥6次) | `true` | `6` | 更新 | **删除** |
| 调用异常 | **不变** | **不变** | **更新** | **保留** |

### 9.2 页面请求触发的状态更新

**页面请求中的 `find_or_fetch_price`**:
- 不更新 `offline`、`failed_fetch_count`、`last_health_check_at`
- 成功时可能更新 `security_prices` 表（缓存新价格）
- 失败时不更新任何状态

---

## 10. 关键发现总结

### 10.1 发现一：两条路径的异常处理不一致

- **健康检查任务**: 有 `rescue` 块，异常被安全捕获
- **页面请求**: 无 `rescue` 块，异常可能导致 500 错误

### 10.2 发现二："失败响应"不等于"异常"

- `provider_error_response` 返回的是正常的 `Response` 对象，只是 `success? == false`
- 这种情况在两条路径中都被优雅处理（返回 `nil`）
- 不会导致 500 错误

### 10.3 发现三："直接报错"的精确条件

**不是**所有 Provider 调用失败都会导致 500 错误。只有同时满足：
1. 数据库无今日价格缓存
2. `provider.fetch_security_price` **抛出 Ruby 异常**
3. 没有上层 `rescue` 块捕获

**才会导致 500 页面**。

---

## 11. 潜在改进建议

### 建议 1：为 `find_or_fetch_price` 添加 `rescue` 保护

**问题**: 用户页面可能因 Provider 异常而崩溃

**建议修改**:
```ruby
def find_or_fetch_price(date: Date.current, cache: true)
  price = prices.find_by(date: date)
  return price if price.present?

  return nil unless provider.present?

  begin
    response = provider.fetch_security_price(
      symbol: ticker,
      exchange_operating_mic: exchange_operating_mic,
      date: date
    )
  rescue => e
    Sentry.capture_exception(e)
    return nil  # 优雅降级
  end

  return nil unless response.success?

  price = response.data
  Security::Price.find_or_create_by!(...) if cache
  price
end
```

### 建议 2：统一健康检查任务的异常处理

**问题**: 调用异常不计入 `failed_fetch_count`，可能掩盖持续问题

**建议**:
```ruby
rescue => e
  Sentry.capture_exception(e) do |scope|
    scope.set_tags(security_id: @security.id)
  end
  handle_failure  # 异常也应该走失败处理逻辑
ensure
  security.update!(last_health_check_at: Time.current)
end
```

---

## 12. 测试覆盖现状

| 测试场景 | 覆盖路径 | 状态 |
|---------|---------|------|
| 失败响应递增计数器 | 健康检查 - 失败响应 | ✅ 已覆盖 |
| 连续失败 6 次标记离线 | 健康检查 - 失败响应 | ✅ 已覆盖 |
| 成功重置计数器 | 健康检查 - 成功 | ✅ 已覆盖 |
| **调用异常场景** | 健康检查 - 调用异常 | ❌ **未覆盖** |
| **页面请求异常场景** | 页面请求 - 调用异常 | ❌ **未覆盖** |

---

## 13. 关键文件索引

| 文件路径 | 职责 | 关键代码行 |
|----------|------|-----------|
| `app/models/security/health_checker.rb` | 健康检查逻辑 | 49-63 (`rescue` + `ensure`) |
| `app/models/security.rb` | `current_price` 方法 | 14-18 |
| `app/models/security/provided.rb` | `find_or_fetch_price` | 39-62 (**无 `rescue`**) |
| `test/support/provider_test_helper.rb` | 测试辅助方法 | 10-16 (`provider_error_response`) |
| `app/views/holdings/show.html.erb` | 持仓详情视图 | 22-25 (显示 "Unknown") |
| `config/schedule.yml` | 定时任务配置 | 16-20 |
