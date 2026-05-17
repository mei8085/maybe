# Webhook 分发机制分析报告

## 1. 整体架构概览

系统目前支持两类外部 Webhook 供应商：Plaid（银行聚合服务）和 Stripe（支付订阅服务）。整体流程遵循「入口 → 签名校验 → 供应商识别 → 事件路由 → 领域处理 → 可观测性」的标准管道模式。

```
外部请求
    ↓
[路由层] config/routes.rb:254-258
    │  ├─ /webhooks/plaid       (US 区域)
    │  ├─ /webhooks/plaid_eu    (EU 区域)
    │  └─ /webhooks/stripe
    ↓
[控制器层] app/controllers/webhooks_controller.rb
    │  跳过 CSRF / 身份认证
    │  读取 raw body + 签名头
    ↓
[签名校验层] 各 Provider 实现
    │  Plaid: JWT + JWKS 验证
    │  Stripe: SDK 内置签名验证
    ↓
[供应商路由] Provider::Registry
    │  按来源实例化对应 Provider
    ↓
[事件分发层]
    │  Plaid: PlaidItem::WebhookProcessor (同步)
    │  Stripe: StripeEventHandlerJob (异步)
    ↓
[领域处理层]
    ├─ 对账领域: SyncJob → PlaidItem::Importer
    ├─ 订阅领域: SubscriptionEventProcessor
    └─ 安全领域: ITEM_ERROR 状态更新
    ↓
[可观测性] Sentry + Rails.logger + Sync 状态机
```

---

## 2. 入口与路由

### 2.1 路由定义

`config/routes.rb:254-258`

```ruby
namespace :webhooks do
  post "plaid"
  post "plaid_eu"
  post "stripe"
end
```

### 2.2 控制器设计

`app/controllers/webhooks_controller.rb`

**关键设计决策：**
- `skip_before_action :verify_authenticity_token` - 外部服务无法携带 CSRF token
- `skip_authentication` - 无需用户登录态

> **⚠️ 重要澄清**：第一轮报告中"所有异常捕获并上报 Sentry"的表述不准确。实际上 Plaid 与 Stripe 的异常处理策略**完全不同**，详见第 8 章的详细对比。

---

## 3. 签名校验机制

### 3.1 Plaid 签名校验

`app/models/provider/plaid.rb:14-43`

**校验流程：**
1. 从请求头获取 `Plaid-Verification`（JWT 格式）
2. 动态加载 JWKS（JSON Web Key Set）：通过 `kid` 从 Plaid API 获取对应公钥
3. JWT 解码验证算法 `ES256`
4. **时间窗口检查**：token 签发时间不超过 5 分钟（防重放）
5. **Body 完整性校验**：比较 JWT payload 中的 `request_body_sha256` 与实际 body 的 SHA256 哈希
6. 使用 `ActiveSupport::SecurityUtils.secure_compare` 防止时序攻击

```ruby
# 核心校验逻辑
issued_at = Time.at(payload["iat"])
raise JWT::VerificationError, "Webhook is too old" if Time.now - issued_at > 5.minutes

expected_hash = payload["request_body_sha256"]
actual_hash = Digest::SHA256.hexdigest(raw_body)
raise JWT::VerificationError, "Invalid webhook body hash" unless ActiveSupport::SecurityUtils.secure_compare(expected_hash, actual_hash)
```

### 3.2 Stripe 签名校验

`app/models/provider/stripe.rb:20-23`

**校验流程：**
1. 从 `HTTP_STRIPE_SIGNATURE` 头获取签名
2. 使用 Stripe SDK 内置方法 `parse_thin_event` 验证
3. 验证失败抛出 `Stripe::SignatureVerificationError`

```ruby
thin_event = client.parse_thin_event(webhook_body, sig_header, webhook_secret)
```

---

## 4. 供应商识别与 Provider 注册表

### 4.1 Provider::Registry 设计

`app/models/provider/registry.rb`

**按区域/类型实例化 Provider：**

```ruby
# Plaid 按区域区分
def plaid_provider_for_region(region)
  region.to_sym == :us ? plaid_us : plaid_eu
end

# Stripe 直接获取
def get_provider(name)
  send(name)
rescue NoMethodError
  raise Error.new("Provider '#{name}' not found in registry")
end
```

**Provider 初始化参数：**
- Plaid: 接收区域特定的配置（API 端点、client_id、secret）
- Stripe: 接收 `secret_key` 和 `webhook_secret`（从环境变量读取）

### 4.2 Provider 缺失的静默失败

> **⚠️ 关键风险点**：Registry 的私有方法在配置缺失时**返回 nil 而非抛出异常**。

```ruby
def stripe
  secret_key = ENV["STRIPE_SECRET_KEY"]
  webhook_secret = ENV["STRIPE_WEBHOOK_SECRET"]

  return nil unless secret_key.present? && webhook_secret.present?  # ← 静默返回 nil

  Provider::Stripe.new(secret_key:, webhook_secret:)
end
```

**这意味着：**
- `Provider::Registry.get_provider(:stripe)` 可能返回 `nil`
- 后续调用 `nil.process_webhook_later` 会抛出 `NoMethodError`
- 该异常是否被捕获取决于控制器的异常处理策略（详见第 8 章）

---

## 5. 事件路由分发

### 5.1 Plaid 事件路由（同步处理）

`app/models/plaid_item/webhook_processor.rb`

**路由策略**：基于 `[webhook_type, webhook_code]` 元组匹配

| webhook_type | webhook_code | 路由目标 | 领域 |
|-------------|-------------|---------|------|
| TRANSACTIONS | SYNC_UPDATES_AVAILABLE | `plaid_item.sync_later` | 对账/交易同步 |
| INVESTMENTS_TRANSACTIONS | DEFAULT_UPDATE | `plaid_item.sync_later` | 对账/投资交易同步 |
| HOLDINGS | DEFAULT_UPDATE | `plaid_item.sync_later` | 对账/持仓同步 |
| ITEM | ERROR | 更新 `status: :requires_update` | 安全/连接状态 |
| * | * | 记录 warn 日志 | - |

**关键设计：**
- 同步处理但快速返回，实际同步工作异步化（`sync_later`）
- 未找到对应 PlaidItem 时上报 Sentry，不抛出异常（保证 200 返回）
- `process` 方法内部有独立的 `rescue` 块，捕获所有处理异常并上报 Sentry

### 5.2 Stripe 事件路由（异步处理）

`app/models/provider/stripe.rb:9-18` + `app/jobs/stripe_event_handler_job.rb`

**两阶段处理：**
1. **控制器阶段**：仅验证签名，提取 `event_id`，立即返回 200
2. **异步阶段**：`StripeEventHandlerJob` 通过 `event_id` 从 Stripe API 拉取完整事件，再按类型路由

```ruby
# 第一阶段（控制器内）
thin_event = client.parse_thin_event(webhook_body, sig_header, webhook_secret)
StripeEventHandlerJob.perform_later(thin_event.id)

# 第二阶段（Job 内）
def process_event(event_id)
  event = retrieve_event(event_id)
  case event.type
  when /^customer\.subscription\./
    SubscriptionEventProcessor.new(event).process
  else
    Rails.logger.warn "Unhandled event type: #{event.type}"
  end
end
```

**当前支持的事件类型：**
- `customer.subscription.*` → 订阅领域（更新订阅状态、金额、周期等）

### 5.3 订阅领域处理器

`app/models/provider/stripe/subscription_event_processor.rb`

**职责：**
- 通过 Stripe `customer_id` 关联内部 `Family`
- 更新订阅状态、金额、周期、到期时间等字段

---

## 6. 失败重试机制

### 6.1 Sidekiq 基础配置

`config/sidekiq.yml`

```yaml
queues:
  - [scheduled, 10]
  - [high_priority, 4]
  - [medium_priority, 2]
  - [low_priority, 1]
  - [default, 1]
```

### 6.2 Job 级重试

`app/jobs/application_job.rb`

```ruby
retry_on ActiveRecord::Deadlocked  # 死锁自动重试
discard_on ActiveJob::DeserializationError  # 记录已删除的对象不重试
```

**Sidekiq 默认重试策略**：
- 最多重试 25 次
- 重试间隔指数退避（约 21 天）
- 重试耗尽后进入 Dead Job 队列

### 6.3 Sync 状态机容错

`app/models/sync.rb`

```ruby
state :pending, initial: true
state :syncing
state :completed
state :failed
state :stale

# 24 小时未完成标记为 stale
STALE_AFTER = 24.hours

# 重复 sync 请求合并：已有进行中的 sync 则扩展时间窗口
def sync_later(parent_sync: nil, window_start_date: nil, window_end_date: nil)
  Sync.transaction do
    with_lock do
      sync = self.syncs.incomplete.first
      if sync
        sync.expand_window_if_needed(window_start_date, window_end_date)
      else
        sync = self.syncs.create!(...)
        SyncJob.perform_later(sync)
      end
    end
  end
end
```

### 6.4 任务入队失败的影响

> **⚠️ 关键风险点**：`perform_later` 可能因以下原因失败，且失败时的行为差异显著：

| 失败场景 | 影响范围 | 异常类型 | 是否被捕获 | 对重试的影响 |
|---------|---------|---------|-----------|-------------|
| Redis 连接中断 | 所有异步任务 | `Redis::CannotConnectError` | Plaid: 是 (400)<br>Stripe: 否 (500) | Plaid: 外部供应商重试<br>Stripe: 外部供应商重试 |
| 任务序列化失败 | 特定任务 | `ActiveJob::SerializationError` | Plaid: 是 (400)<br>Stripe: 否 (500) | 同上 |
| Sidekiq 进程未运行 | 所有异步任务 | 入队成功但永不执行 | - | 依赖 Sidekiq 自身的重试机制 |

**特别注意**：`sync_later` 中的 `perform_later` 在数据库事务内，如果入队失败，**整个事务会回滚**，Sync 记录不会被创建。

---

## 7. 可观测性手段

### 7.1 Sentry 异常上报

**关键上报点：**

| 位置 | 异常类型 | 上报方式 |
|-----|---------|---------|
| `webhooks_controller.rb:17,34` | Plaid 所有异常 | `rescue => error` 全量捕获 |
| `webhooks_controller.rb:48,52` | Stripe JSON/签名异常 | 特定 rescue 捕获 |
| `webhooks_controller.rb` (未捕获) | Stripe 其他异常 | Rails 全局异常处理 |
| `webhook_processor.rb:32-35` | Plaid 处理阶段异常 | 内部 rescue 捕获 |
| `sync.rb:73-76` | Sync 执行失败 | 内部 rescue 捕获 |
| `sync.rb:146-149` | Post-sync 钩子异常 | 内部 rescue 捕获 |
| `webhook_processor.rb:52-54` | Plaid Item 缺失 | 主动上报 |

**上下文标签示例：**
```ruby
Sentry.capture_exception(MissingItemError.new(...)) do |scope|
  scope.set_tags(plaid_item_id: item_id)
end
```

### 7.2 观测性盲区

> **⚠️ 第一轮报告遗漏**：存在多个观测性盲区

1. **Stripe 未捕获异常**：`NoMethodError`（Provider 为 nil）、`Redis::CannotConnectError` 等异常**不会**被 Stripe 控制器的特定 rescue 捕获，依赖 Rails 全局异常处理
2. **任务入队成功但执行失败**：如 Stripe API 拉取事件失败、订阅处理失败等，仅在 Job 日志中有记录
3. **静默丢弃的事件**：未匹配到路由的 webhook 仅记录 warn 日志，不上报 Sentry

### 7.3 日志记录

**关键日志点：**
- 未处理的 webhook 类型（warn 级别）
- Sync 状态转换（info 级别）
- Stripe 事件处理开始（info 级别）
- Sync 窗口扩展（info 级别）
- Post-sync 错误（error 级别）

### 7.4 业务指标

`app/models/sync.rb:157-166`

```ruby
# 单日 sync 超过 10 次告警（可能存在循环触发）
if todays_sync_count > 10
  Sentry.capture_exception(
    Error.new("#{syncable_type} (#{syncable.id}) has exceeded 10 syncs today"),
    level: :warning
  )
end
```

---

## 8. Plaid vs Stripe：异常处理与返回码深度对比

### 8.1 控制器异常处理代码对比

**Plaid (`webhooks_controller.rb:5-19`)：**
```ruby
def plaid
  # ... 业务逻辑 ...
  render json: { received: true }, status: :ok
rescue => error  # ← 捕获 ALL 异常
  Sentry.capture_exception(error)
  render json: { error: "Invalid webhook: #{error.message}" }, status: :bad_request
end
```

**Stripe (`webhooks_controller.rb:37-56`)：**
```ruby
def stripe
  stripe_provider = Provider::Registry.get_provider(:stripe)  # ← 可能返回 nil
  begin
    # ... 业务逻辑 ...
    head :ok
  rescue JSON::ParserError => error  # ← 仅捕获 JSON 解析错误
    Sentry.capture_exception(error)
    head :bad_request
  rescue Stripe::SignatureVerificationError => error  # ← 仅捕获签名错误
    Sentry.capture_exception(error)
    head :bad_request
  end
  # ← 其他异常直接抛出，返回 500
end
```

### 8.2 异常场景行为矩阵

| 异常场景 | Plaid 行为 | Stripe 行为 |
|---------|-----------|------------|
| 签名验证失败 | 捕获 → Sentry → 400 | 捕获 → Sentry → 400 |
| JSON 解析失败 | 捕获 → Sentry → 400 | 捕获 → Sentry → 400 |
| Provider 配置缺失 (返回 nil) | 捕获 → Sentry → 400 | **未捕获 → Rails 500** |
| 任务入队失败 (Redis 问题) | 捕获 → Sentry → 400 | **未捕获 → Rails 500** |
| 业务处理阶段异常 | WebhookProcessor 内部捕获 → Sentry → 200 | Job 中异常 → Sidekiq 重试 |
| 未找到对应资源 (PlaidItem/Family) | 捕获 → Sentry → 200 | Job 中异常 → Sidekiq 重试 |

### 8.3 设计意图分析

| 维度 | Plaid 策略 | Stripe 策略 |
|-----|-----------|------------|
| 核心目标 | **保持端点健康**：始终返回 200/400，避免 Plaid 标记端点为不健康 | **快速响应**：仅验证必要信息，快速返回 200，异步处理 |
| 异常哲学 | **容错优先**：即使内部处理失败，也告诉 Plaid "已收到" | **诚实反馈**：签名/格式错误返回 400，系统错误返回 500 |
| 重试依赖 | 依赖 Plaid 对 400 的重试机制 | 依赖 Stripe 对 500 的重试 + Sidekiq 对 Job 的重试 |
| 可观测性 | 所有异常都在控制器层上报 Sentry | 部分异常遗漏，依赖 Rails 全局处理 |

### 8.4 外部供应商重试行为

- **Plaid**：对非 200 响应会重试，具体策略未明确文档化
- **Stripe**：对 4xx 响应**不重试**（认为是客户端错误），对 5xx 响应会重试（指数退避，最多 24 小时）

> **⚠️ 关键影响**：Stripe 入口的 500 错误**会触发 Stripe 重试**，而 400 错误不会。这意味着 Provider 缺失、Redis 故障等场景下，Stripe 会自动重试，直到成功或超过重试上限。

---

## 9. 架构特点与权衡

| 决策 | 优点 | 缺点 |
|-----|-----|-----|
| Plaid 同步处理签名验证 | 签名失败立即返回 400，Plaid 会重试 | 控制器内逻辑稍重 |
| Stripe 异步拉取完整事件 | 控制器快速返回 200，降低超时风险；thin event 体积小 | 需要额外 API 调用拉取完整事件 |
| Sync 去重 + 窗口扩展 | 避免重复 sync，合并相邻时间范围 | 逻辑复杂度增加 |
| Plaid 全局异常捕获 | 保证外部供应商收到 400，端点健康状态良好 | 问题类型被统一掩盖，难以区分是签名错误还是系统错误 |
| Stripe 特定异常捕获 | 签名/格式错误快速反馈，不触发不必要的重试 | 系统错误返回 500，观测性依赖 Rails 全局处理 |
| Registry 静默返回 nil | 配置缺失时不崩溃 | 延迟失败，增加调试复杂度 |

---

## 10. 扩展点与建议

### 10.1 潜在扩展方向
1. **新增供应商**：在 `Provider::Registry` 添加新方法，实现对应 `validate_webhook!` 和处理器
2. **新事件类型**：在对应 Processor 的 case 语句中增加新分支
3. **死信队列**：为 webhook 处理失败增加专门的死信队列和人工重试界面

### 10.2 缺陷修复建议

**高优先级：**
1. **统一 Stripe 异常处理**：在 Stripe 控制器增加 `rescue => error` 兜底，或在 Registry 中增加 nil 检查
2. **Registry 失败快速**：配置缺失时抛出明确异常而非返回 nil，便于及早发现问题
3. **Stripe 入队失败保护**：对 `perform_later` 增加异常捕获，避免 500 错误

**中优先级：**
4. **未处理事件告警**：对未匹配路由的 webhook 类型增加 Sentry warning 级别上报
5. **幂等性校验**：增加 webhook event_id 去重机制，避免重复处理

### 10.3 可观测性增强建议
1. 增加 webhook 处理延迟指标（从接收到处理完成的时间）
2. 按供应商/事件类型统计成功率和失败率
3. 增加 webhook 幂等性校验（处理过的 event_id 不重复处理）
4. 为 Stripe 500 错误场景增加专门的告警规则
