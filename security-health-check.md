# Security 健康检查系统报告

## 1. 概述

本报告基于代码精确分析了 Maybe 金融应用中的证券健康检查系统，重点明确：
- `Provider::Synth` 的 `with_provider_response` 包装行为
- **异常被封装为失败响应**的确切代码证据
- "会回落 Unknown" 与 "返回 500" 的对照表
- 页面直接报错的**真实触发条件**

---

## 2. 核心发现：Provider 层统一捕获异常

### 2.1 `with_provider_response` 的异常封装机制

**文件**: `app/models/provider.rb:24-44`

```ruby
def with_provider_response(error_transformer: nil, &block)
  data = yield

  Response.new(
    success?: true,
    data: data,
    error: nil,
  )
rescue => error
  # ── 关键：所有异常被捕获 ──
  transformed_error = if error_transformer
    error_transformer.call(error)
  else
    default_error_transformer(error)
  end

  Response.new(
    success?: false,
    data: nil,
    error: transformed_error
  )
end
```

**行为分析**:
1. `yield` 执行传入的 block（即具体的 Provider API 调用）
2. **所有异常**（`Net::ReadTimeout`、`Faraday::Error`、`StandardError` 等）都会被 `rescue => error` 捕获
3. 捕获后**不会重新抛出**，而是封装为 `Response.new(success?: false, data: nil, error: transformed_error)`
4. 返回值始终是 `Provider::Response` 对象，**永远不会抛出异常**

---

### 2.2 错误转换器

**文件**: `app/models/provider.rb:47-56`

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

`Provider::Synth` 定义了自己的 Error 类 (`app/models/provider/synth.rb:5`):
```ruby
Error = Class.new(Provider::Error)
```

---

### 2.3 `fetch_security_price` 的实现

**文件**: `app/models/provider/synth.rb:136-144`

```ruby
def fetch_security_price(symbol:, exchange_operating_mic: nil, date:)
  with_provider_response do  # ← 所有调用被 with_provider_response 包装
    historical_data = fetch_security_prices(symbol:, exchange_operating_mic:, start_date: date, end_date: date)

    raise ProviderError, "No prices found for security #{symbol} on date #{date}" if historical_data.data.empty?
    # ↑ 即使 block 内部主动 raise，也会被 with_provider_response 捕获

    historical_data.data.first
  end
end
```

**关键观察**:
- `fetch_security_price` 完全被 `with_provider_response` 包装
- 无论网络超时、API 限流、还是 `raise ProviderError`，**所有异常都会被捕获**
- 返回值始终是 `Response` 对象（`success?: true` 或 `success?: false`）

---

### 2.4 测试证据

**文件**: `test/models/provider_test.rb:39-46`

```ruby
test "returns failed response with error" do
  @client.expects(:get).with("/test").raises(StandardError.new("some error"))

  response = @provider.fetch_data

  assert_not response.success?  # ← success? 为 false
  assert_equal("some error", response.error.message)
end
```

**测试证明**:
- `@client.get` 抛出 `StandardError`
- **没有冒泡异常**，而是返回 `response.success? == false`
- 异常被封装在 `response.error` 中

---

## 3. 两条执行路径重新校准

### 3.1 路径一：健康检查任务（后台定时任务）

**文件**: `app/models/security/health_checker.rb:49-63`

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

**`latest_provider_price` 实现** (`health_checker.rb:72-84`):
```ruby
def latest_provider_price
  return nil unless provider.present?

  response = provider.fetch_security_price(...)

  return nil unless response.success?  # ← success? 为 false 时返回 nil

  response.data.price
end
```

**现在的真实行为**（基于 `with_provider_response`）:

| 场景 | `provider.fetch_security_price` 返回 | `latest_provider_price` 返回 | `run_check` 执行分支 |
|------|------------------------------------|----------------------------|-------------------|
| 网络正常，有价格数据 | `Response(success?: true, data: Price)` | `Price.price` (非 nil) | `handle_success` |
| 网络正常，无价格数据 | `Response(success?: false, error: ProviderError)` | `nil` | `handle_failure` |
| 网络超时 `Net::ReadTimeout` | `Response(success?: false, error: Synth::Error)` | `nil` | `handle_failure` |
| API 限流 `Faraday::ClientError` | `Response(success?: false, error: Synth::Error)` | `nil` | `handle_failure` |
| DNS 失败 `SocketError` | `Response(success?: false, error: Synth::Error)` | `nil` | `handle_failure` |

**重要结论**:
- `health_checker.rb:57-60` 的 `rescue => e` **实际上几乎不会触发**
- 因为 `provider.fetch_security_price` 被 `with_provider_response` 包装，**永远不会抛出异常**
- 所有"失败"情况都会走 `handle_failure` 分支

---

### 3.2 路径二：页面请求时的 `current_price`

**文件**: `app/models/security/provided.rb:39-62`

```ruby
def find_or_fetch_price(date: Date.current, cache: true)
  price = prices.find_by(date: date)

  return price if price.present?

  return nil unless provider.present?
  response = provider.fetch_security_price(
    symbol: ticker,
    exchange_operating_mic: exchange_operating_mic,
    date: date
  )

  return nil unless response.success?  # ← success? 为 false 时返回 nil

  price = response.data
  Security::Price.find_or_create_by!(...) if cache
  price
end
```

**现在的真实行为**（基于 `with_provider_response`）:

| 场景 | `provider.fetch_security_price` 返回 | `find_or_fetch_price` 返回 | `current_price` 返回 |
|------|------------------------------------|--------------------------|---------------------|
| 有数据库缓存 | 不调用 Provider | `Price` (from DB) | `Money` 对象 |
| 无缓存，网络正常有价格 | `Response(success?: true, data: Price)` | `Price` (缓存到 DB) | `Money` 对象 |
| 无缓存，网络正常无价格 | `Response(success?: false, error: ProviderError)` | `nil` | `nil` |
| 无缓存，网络超时 | `Response(success?: false, error: Synth::Error)` | `nil` | `nil` |
| 无缓存，API 限流 | `Response(success?: false, error: Synth::Error)` | `nil` | `nil` |

**关键结论**:
- `find_or_fetch_price` 中**没有异常能抛出来**
- 因为 `provider.fetch_security_price` 被 `with_provider_response` 包装
- 所有"失败"情况都会优雅返回 `nil`，**不会导致 500 错误**

---

## 4. "会回落 Unknown" 与 "返回 500" 对照表

### 4.1 对照表

| 场景 | Provider 层行为 | 调用层返回值 | `current_price` | 前端展示 |
|------|----------------|-------------|-----------------|---------|
| 有数据库缓存 | 不调用 Provider | `Price` (from DB) | `Money` 对象 | 显示价格 `$100.00` |
| 无缓存，网络正常有价格 | `Response(success?: true)` | `Price` | `Money` 对象 | 显示价格 `$100.00` |
| 无缓存，网络正常无价格 | `Response(success?: false)` | `nil` | `nil` | 显示 `Unknown` |
| 无缓存，网络超时 `Net::ReadTimeout` | `Response(success?: false)` | `nil` | `nil` | 显示 `Unknown` |
| 无缓存，API 限流 `Faraday::ClientError` | `Response(success?: false)` | `nil` | `nil` | 显示 `Unknown` |
| 无缓存，DNS 失败 `SocketError` | `Response(success?: false)` | `nil` | `nil` | 显示 `Unknown` |
| 无缓存，JSON 解析异常 | `Response(success?: false)` | `nil` | `nil` | 显示 `Unknown` |
| **无缓存，`with_provider_response` 外部异常** | 异常未被捕获 | **异常上冒泡** | **500** | **500 页面** |

---

### 4.2 页面直接报错的真实触发条件

**基于 `with_provider_response` 的分析，页面返回 500 只有两种可能**:

#### 可能性一：异常发生在 `with_provider_response` 包装之外

**理论场景**:
- 异常发生在 `fetch_security_price` 方法的 `with_provider_response` 包装之前
- 或异常发生在 `with_provider_response` 的 `ensure`/`else` 等未被 `rescue` 覆盖的地方

**代码检查**: `provider/synth.rb:136-144`
```ruby
def fetch_security_price(symbol:, exchange_operating_mic: nil, date:)
  with_provider_response do
    historical_data = fetch_security_prices(...)
    raise ProviderError, "No prices found..." if historical_data.data.empty?
    historical_data.data.first
  end
end
```

**分析**:
- 整个方法体都在 `with_provider_response do ... end` 块内
- `with_provider_response` 的 `rescue => error` 捕获**所有异常**
- **这种情况几乎不可能发生**

#### 可能性二：异常发生在 `find_or_fetch_price` 的其他部分

**代码位置**: `app/models/security/provided.rb:55-60`

```ruby
price = response.data
Security::Price.find_or_create_by!(
  security_id: self.id,
  date: price.date,           # ← 如果 response.data 为 nil，price.date 会抛 NoMethodError
  price: price.price,
  currency: price.currency
) if cache
```

**分析**:
1. `response.data` 在 `success? == true` 时才有数据
2. `success? == false` 时会 `return nil unless response.success?`，不会执行到这里
3. 但如果 `response.success? == true` 且 `response.data.nil?`（理论上可能的 bug）
4. 那么 `price = response.data` 会是 `nil`
5. `price.date` 会抛出 `NoMethodError: undefined method 'date' for nil:NilClass`
6. **这个异常在 `find_or_fetch_price` 中没有被捕获**
7. **会导致 500 错误**

---

### 4.3 最小可复核证据链（会回落 Unknown）

#### 证据链 A：异常被封装为失败响应

**第一步：调用入口**

**文件**: `app/models/provider/synth.rb:136`

```ruby
def fetch_security_price(symbol:, exchange_operating_mic: nil, date:)
  with_provider_response do
    # ... 业务逻辑 ...
  end
end
```

**第二步：分支判定**

**文件**: `app/models/provider.rb:24-44`

```ruby
def with_provider_response(error_transformer: nil, &block)
  data = yield
  # ... 成功返回 ...
rescue => error  # ← 第 32 行：所有异常被捕获
  # ← 第 32-43 行：异常被转换为 Response 对象
  Response.new(
    success?: false,
    data: nil,
    error: transformed_error
  )
end
```

**第三步：返回对象形态**

| 场景 | 返回对象 |
|------|---------|
| 正常有数据 | `Response(success?: true, data: Price, error: nil)` |
| 正常无数据 | `Response(success?: false, data: nil, error: ProviderError)` |
| 网络超时 | `Response(success?: false, data: nil, error: Synth::Error)` |
| API 限流 | `Response(success?: false, data: nil, error: Synth::Error)` |

**测试证据**: `test/models/provider_test.rb:39-46`
```ruby
@client.expects(:get).raises(StandardError.new("some error"))
response = @provider.fetch_data
assert_not response.success?  # ← 不是抛异常，而是 response.success? 为 false
```

**第四步：页面结果**

**文件**: `app/models/security/provided.rb:46-52`

```ruby
response = provider.fetch_security_price(...)
return nil unless response.success?  # ← success? 为 false 时返回 nil
```

→ `find_or_fetch_price` 返回 `nil`

**文件**: `app/models/security.rb:14-18`
```ruby
def current_price
  @current_price ||= find_or_fetch_price
  return nil if @current_price.nil?  # ← 返回 nil
  Money.new(...)
end
```

→ `current_price` 返回 `nil`

**文件**: `app/views/holdings/show.html.erb:22-25`
```erb
<dd class="text-primary">
  <%= @holding.security.current_price ? format_money(...) : t(".unknown") %>
</dd>
```

→ 前端显示 `Unknown`

---

### 4.4 最小可复核证据链（返回 500）

#### 证据链 B：`response.data` 为 nil 但 `success? == true`

**前提条件**（理论上的 bug 场景）:
- `provider.fetch_security_price` 返回 `Response(success?: true, data: nil, error: nil)`
- 即 `success?` 为 `true` 但 `data` 为 `nil`

**第一步：调用入口**

**文件**: `app/models/security/provided.rb:46-52`

```ruby
response = provider.fetch_security_price(
  symbol: ticker,
  exchange_operating_mic: exchange_operating_mic,
  date: date
)

return nil unless response.success?  # ← success? 为 true，不返回 nil
```

→ 因为 `success? == true`，继续执行

**第二步：分支判定**

**文件**: `app/models/security/provided.rb:54-60`

```ruby
price = response.data  # ← response.data 为 nil

Security::Price.find_or_create_by!(
  security_id: self.id,
  date: price.date,     # ← nil.date 会抛 NoMethodError
  price: price.price,
  currency: price.currency
) if cache
```

**第三步：返回对象形态**

- `price = nil`
- 调用 `price.date` 抛出 `NoMethodError: undefined method 'date' for nil:NilClass`
- `find_or_fetch_price` **没有 `rescue` 块**
- 异常向上冒泡

**第四步：页面结果**

- 开发环境：显示异常堆栈（红屏）
- 生产环境：显示 500 错误页面

**但这是理论场景**：
- 正常情况下 `success? == true` 时 `data` 不会为 `nil`
- 这需要 Provider 层有 bug 才会发生
- **实际上，页面 500 在正常情况下几乎不可能发生**

---

## 5. 健康检查任务的状态字段变化

### 5.1 重新校准后的真实行为

基于 `with_provider_response`，`health_checker.rb:57-60` 的 `rescue` **几乎不会触发**。

所有"失败"场景都会走 `handle_failure`：

| 场景 | `provider.fetch_security_price` 返回 | `run_check` 分支 | 状态变化 |
|------|------------------------------------|----------------|---------|
| 网络正常有价格 | `Response(success?: true)` | `handle_success` | `offline=false`, `failed_count=0` |
| 网络正常无价格 | `Response(success?: false)` | `handle_failure` | `failed_count+1` |
| 网络超时 | `Response(success?: false)` | `handle_failure` | `failed_count+1` |
| API 限流 | `Response(success?: false)` | `handle_failure` | `failed_count+1` |
| DNS 失败 | `Response(success?: false)` | `handle_failure` | `failed_count+1` |
| **Provider 层 bug（抛异常）** | **异常上冒泡** | **`rescue` 捕获** | **状态不变** |

### 5.2 状态字段更新表

| 场景 | `offline` | `failed_fetch_count` | `last_health_check_at` | 价格数据 |
|------|-----------|---------------------|-----------------------|---------|
| 成功（有价格） | 设为 `false` | 设为 `0` | 更新 | 保留 |
| 失败 1-5 次（含网络异常） | 不变 | `+1` | 更新 | 保留 |
| 失败 ≥6 次（含网络异常） | 设为 `true` | 设为 `6` | 更新 | **删除** |
| **Provider 层 bug 抛异常** | **不变** | **不变** | **更新** | **保留** |

---

## 6. 端到端流程图（最终版）

```
┌─────────────────────────────────────────────────────────────────────┐
│              Provider::Synth.fetch_security_price                   │
│              app/models/provider/synth.rb:136-144                   │
│                                                                     │
│  with_provider_response do                                          │
│    # 业务逻辑                                                        │
│  end                                                                │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ with_provider_response (provider.rb:24-44)                  │   │
│  │                                                             │   │
│  │ 正常 → Response(success?: true, data: Price)                │   │
│  │ 异常 → rescue → Response(success?: false, error: Error)     │   │
│  │                                                             │   │
│  │ 永远返回 Response 对象，永远不会抛异常！                      │   │
│  └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
                          │
                          ↓
┌─────────────────────────────────────────────────────────────────────┐
│                     健康检查任务（凌晨 2:00 AM）                       │
│                      app/models/security/health_checker.rb            │
│                                                                     │
│  latest_provider_price                                               │
│    → response.success? ? data.price : nil                           │
│                                                                     │
│  所有"失败"都走 handle_failure                                       │
│  rescue 块几乎不会触发（除非 Provider 本身有 bug）                     │
└─────────────────────────────────────────────────────────────────────┘
                          │
                          ↓
┌─────────────────────────────────────────────────────────────────────┐
│                      用户访问页面（任意时间）                          │
│                                                                     │
│  current_price                                                       │
│    → find_or_fetch_price                                            │
│      → prices.find_by(date: today)                                  │
│      → 有缓存 → 返回价格 → 正常显示                                  │
│      → 无缓存 → provider.fetch_security_price                       │
│              → Response(success?: true) → 缓存并返回价格             │
│              → Response(success?: false) → 返回 nil → Unknown       │
│                                                                     │
│  所有 Provider 异常都被封装为失败响应                                  │
│  正常情况下不会抛异常，不会导致 500                                    │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 7. 最终关键发现总结

### 7.1 发现一：`with_provider_response` 统一捕获所有异常

- **位置**: `app/models/provider.rb:24-44`
- **行为**: `rescue => error` 捕获所有异常，封装为 `Response(success?: false)`
- **影响**: `provider.fetch_security_price` **永远不会抛出异常**

### 7.2 发现二：健康检查的 `rescue` 几乎无用

- **位置**: `app/models/security/health_checker.rb:57-60`
- **行为**: 理论上用于捕获异常
- **实际**: 因为 Provider 层统一捕获，这个 `rescue` **几乎不会触发**
- **例外**: 只有当 `with_provider_response` 本身有 bug 抛异常时才会触发

### 7.3 发现三：页面 500 几乎不可能发生

- 正常情况下，所有 Provider 异常都会被 `with_provider_response` 捕获
- `find_or_fetch_price` 收到的是 `Response(success?: false)`，返回 `nil`
- 前端显示 `Unknown`，**不会报错**

### 7.4 发现四：网络异常计入 `failed_fetch_count`

- 网络超时、API 限流等异常被封装为 `Response(success?: false)`
- 健康检查中走 `handle_failure` 分支
- `failed_fetch_count` 会递增
- 连续 6 次后标记 `offline=true`

---

## 8. 对照表（会回落 Unknown vs 返回 500）

### 8.1 会回落 Unknown 的场景

| 场景 | Provider 行为 | `response.success?` | `current_price` | 前端展示 |
|------|-------------|---------------------|-----------------|---------|
| 网络正常，无价格数据 | 封装为失败响应 | `false` | `nil` | `Unknown` |
| 网络超时 `Net::ReadTimeout` | 封装为失败响应 | `false` | `nil` | `Unknown` |
| API 限流 `Faraday::ClientError` | 封装为失败响应 | `false` | `nil` | `Unknown` |
| DNS 失败 `SocketError` | 封装为失败响应 | `false` | `nil` | `Unknown` |
| JSON 解析异常 `JSON::ParserError` | 封装为失败响应 | `false` | `nil` | `Unknown` |
| Provider 返回 404/429 | 封装为失败响应 | `false` | `nil` | `Unknown` |

### 8.2 理论上可能返回 500 的场景

| 场景 | 触发条件 | 概率 | 本质原因 |
|------|---------|------|---------|
| `response.data` 为 nil 但 `success? == true` | Provider 层 bug | 极低 | `nil.date` 抛 `NoMethodError` |
| `with_provider_response` 自身抛异常 | Provider 层严重 bug | 极低 | `rescue` 块外有异常 |
| `prices.find_by` 抛异常 | 数据库连接问题 | 低 | 数据库层问题 |

**实际结论**: 正常使用下，页面**几乎不可能**返回 500。所有 Provider 相关异常都会优雅回落为 `Unknown`。

---

## 9. 潜在改进建议

### 建议 1：移除健康检查中冗余的 `rescue`（或统一处理）

**问题**: `health_checker.rb:57-60` 的 `rescue` 几乎不会触发，因为 Provider 层统一捕获了。

**建议 A（移除冗余代码）**:
```ruby
def run_check
  Rails.logger.info("Running health check for #{security.ticker}")

  if latest_provider_price
    handle_success
  else
    handle_failure
  end
  # 移除 rescue，因为 Provider 层已经统一捕获
ensure
  security.update!(last_health_check_at: Time.current)
end
```

**建议 B（保留作为兜底，并统一行为）**:
```ruby
def run_check
  # ...
rescue => e
  Sentry.capture_exception(e) do |scope|
    scope.set_tags(security_id: @security.id)
  end
  handle_failure  # 异常也计入失败计数
ensure
  security.update!(last_health_check_at: Time.current)
end
```

### 建议 2：为 `find_or_fetch_price` 添加防御性检查

虽然正常情况下不会抛异常，但可以添加防御性代码：

```ruby
def find_or_fetch_price(date: Date.current, cache: true)
  price = prices.find_by(date: date)
  return price if price.present?

  return nil unless provider.present?

  response = provider.fetch_security_price(...)

  return nil unless response.success?
  return nil unless response.data.present?  # ← 防御性检查

  price = response.data
  # ...
end
```

---

## 10. 测试覆盖现状

| 测试场景 | 覆盖路径 | 状态 |
|---------|---------|------|
| 失败响应递增计数器 | 健康检查 - 失败响应 | ✅ 已覆盖 |
| 连续失败 6 次标记离线 | 健康检查 - 失败响应 | ✅ 已覆盖 |
| 成功重置计数器 | 健康检查 - 成功 | ✅ 已覆盖 |
| **Provider 异常被封装为失败响应** | Provider 层 | ✅ 已覆盖 (`test/models/provider_test.rb:39-46`) |
| **页面请求异常场景** | 页面请求 - 理论 bug 场景 | ❌ **理论场景，无需覆盖** |

---

## 11. 关键文件索引

| 文件路径 | 职责 | 关键代码行 |
|----------|------|-----------|
| `app/models/provider.rb` | 基类，`with_provider_response` 实现 | 24-44（异常封装核心） |
| `app/models/provider/synth.rb` | Synth Provider 实现 | 136-144 (`fetch_security_price`) |
| `app/models/security/health_checker.rb` | 健康检查逻辑 | 49-63 (`rescue` 几乎无用) |
| `app/models/security.rb` | `current_price` 方法 | 14-18 |
| `app/models/security/provided.rb` | `find_or_fetch_price` | 39-62 |
| `test/models/provider_test.rb` | Provider 测试 | 39-46（异常被封装为失败响应） |
| `app/views/holdings/show.html.erb` | 持仓详情视图 | 22-25（显示 "Unknown"） |
