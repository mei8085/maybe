# Webhook 分发机制分析报告

## 证据来源标注说明

本报告中所有结论按证据来源分为两类：

- **✅ 代码可证**：结论可通过仓库内的代码直接验证
- **📚 外部规则**：结论依赖外部供应商文档、Rails 框架行为或通用中间件特性，代码库中无直接证据

---

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
    │  Plaid: PlaidItem::WebhookProcessor (同步，内部吞异常)
    │  Stripe: StripeEventHandlerJob (异步)
    ↓
[领域处理层]
    ├─ 对账领域: SyncJob → PlaidItem::Importer
    ├─ 订阅领域: SubscriptionEventProcessor
    └─ 安全领域: ITEM_ERROR 状态更新
    ↓
[可观测性] Sentry + Rails.logger + Sync 状态机
```

**证据标注**：
- ✅ 代码可证：路由定义、控制器代码、各层类的存在均可在代码库中直接验证

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

**证据标注**：
- ✅ 代码可证：路由文件中明确定义

### 2.2 控制器设计

`app/controllers/webhooks_controller.rb`

**关键设计决策**：
- `skip_before_action :verify_authenticity_token` - 外部服务无法携带 CSRF token
- `skip_authentication` - 无需用户登录态

**证据标注**：
- ✅ 代码可证：控制器代码中明确声明

---

## 3. 签名校验机制

### 3.1 Plaid 签名校验

`app/models/provider/plaid.rb:14-43`

**校验流程**：
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

**证据标注**：
- ✅ 代码可证：校验逻辑完全在 `validate_webhook!` 方法中实现
- 📚 外部规则：JWT 标准、ES256 算法规范、JWKS 机制为行业通用标准

### 3.2 Stripe 签名校验

`app/models/provider/stripe.rb:20-23`

**校验流程**：
1. 从 `HTTP_STRIPE_SIGNATURE` 头获取签名
2. 使用 Stripe SDK 内置方法 `parse_thin_event` 验证
3. 验证失败抛出 `Stripe::SignatureVerificationError`

```ruby
thin_event = client.parse_thin_event(webhook_body, sig_header, webhook_secret)
```

**证据标注**：
- ✅ 代码可证：调用 `parse_thin_event` 方法，捕获 `Stripe::SignatureVerificationError` 异常
- 📚 外部规则：`parse_thin_event` 的具体校验逻辑封装在 Stripe SDK 内部

---

## 4. 供应商识别与 Provider 注册表

### 4.1 Provider::Registry 设计

`app/models/provider/registry.rb`

**按区域/类型实例化 Provider**：

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

**证据标注**：
- ✅ 代码可证：Registry 代码中明确定义

**Provider 初始化参数**：
- Plaid: 接收区域特定的配置（API 端点、client_id、secret）
- Stripe: 接收 `secret_key` 和 `webhook_secret`（从环境变量读取）

**证据标注**：
- ✅ 代码可证：各 Provider 的 `initialize` 方法明确定义

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

**这意味着**：
- `Provider::Registry.get_provider(:stripe)` 可能返回 `nil`
- 后续调用 `nil.process_webhook_later` 会抛出 `NoMethodError`
- 该异常是否被捕获取决于控制器的异常处理策略（详见第 8 章）

**证据标注**：
- ✅ 代码可证：Registry 的私有方法中明确使用 `return nil` 条件判断

---

## 5. 事件路由分发

### 5.1 Plaid 事件路由（同步处理 + 内部吞异常）

`app/models/plaid_item/webhook_processor.rb`

**路由策略**：基于 `[webhook_type, webhook_code]` 元组匹配

| webhook_type | webhook_code | 路由目标 | 领域 |
|-------------|-------------|---------|------|
| TRANSACTIONS | SYNC_UPDATES_AVAILABLE | `plaid_item.sync_later` | 对账/交易同步 |
| INVESTMENTS_TRANSACTIONS | DEFAULT_UPDATE | `plaid_item.sync_later` | 对账/投资交易同步 |
| HOLDINGS | DEFAULT_UPDATE | `plaid_item.sync_later` | 对账/持仓同步 |
| ITEM | ERROR | 更新 `status: :requires_update` | 安全/连接状态 |
| * | * | 记录 warn 日志 | - |

**证据标注**：
- ✅ 代码可证：`case [ webhook_type, webhook_code ]` 语句中明确定义

**关键设计**：
- 同步处理但快速返回，实际同步工作异步化（`sync_later`）
- **`process` 方法内部有独立的 `rescue` 块，捕获所有处理异常并上报 Sentry，但不重新抛出**（`webhook_processor.rb:32-35`）
- 这意味着：**process 方法永远不会向外抛出异常**，控制器层的 rescue 无法捕获到处理阶段的异常

**证据标注**：
- ✅ 代码可证：`process` 方法末尾的 `rescue => e` 块只调用 `Sentry.capture_exception(e)`，不重新抛出

### 5.2 Stripe 事件路由（异步处理）

`app/models/provider/stripe.rb:9-18` + `app/jobs/stripe_event_handler_job.rb`

**两阶段处理**：
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

**证据标注**：
- ✅ 代码可证：`process_webhook_later` 和 `perform` 方法中明确定义
- 📚 外部规则：Stripe thin event 与完整 event 的关系是 Stripe API 的设计

**当前支持的事件类型**：
- `customer.subscription.*` → 订阅领域（更新订阅状态、金额、周期等）

**证据标注**：
- ✅ 代码可证：`when /^customer\.subscription\./` 正则匹配

### 5.3 订阅领域处理器

`app/models/provider/stripe/subscription_event_processor.rb`

**职责**：
- 通过 Stripe `customer_id` 关联内部 `Family`
- 更新订阅状态、金额、周期、到期时间等字段

**证据标注**：
- ✅ 代码可证：`process` 方法中明确定义

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

**证据标注**：
- ✅ 代码可证：Sidekiq 配置文件中明确定义

### 6.2 Job 级重试

`app/jobs/application_job.rb`

```ruby
retry_on ActiveRecord::Deadlocked  # 死锁自动重试
discard_on ActiveJob::DeserializationError  # 记录已删除的对象不重试
```

**证据标注**：
- ✅ 代码可证：ApplicationJob 中明确定义

**Sidekiq 默认重试策略**（代码库未显式配置，为框架默认值）：
- 最多重试 25 次
- 重试间隔指数退避（约 21 天）
- 重试耗尽后进入 Dead Job 队列

**证据标注**：
- 📚 外部规则：Sidekiq 官方文档定义的默认行为，代码库中无自定义配置

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

**证据标注**：
- ✅ 代码可证：Sync 模型中明确定义状态机和 `sync_later` 方法

### 6.4 任务入队失败的真实影响

> **⚠️ 关键澄清**：Plaid 的 `sync_later` 在 `WebhookProcessor.process` 内部被调用，**入队异常会被 process 方法的 rescue 吞掉**，最终 HTTP 返回 200 而非 400。

| 失败场景 | 影响范围 | 异常类型 | Plaid 捕获者 | Plaid HTTP 状态 | Stripe 捕获者 | Stripe HTTP 状态 |
|---------|---------|---------|-------------|----------------|--------------|-----------------|
| Redis 连接中断 | 所有异步任务 | `Redis::CannotConnectError` | WebhookProcessor 内部 | **200** | 未捕获 | **500** |
| 任务序列化失败 | 特定任务 | `ActiveJob::SerializationError` | WebhookProcessor 内部 | **200** | 未捕获 | **500** |
| Sidekiq 进程未运行 | 所有异步任务 | 入队成功但永不执行 | - | 200 | - | 200 |

**证据标注**：
- ✅ 代码可证：WebhookProcessor 内部吞异常逻辑可直接验证
- 📚 外部规则：`Redis::CannotConnectError` 是 Redis 客户端的标准异常类型

**特别注意**：`sync_later` 中的 `perform_later` 在数据库事务内，如果入队失败，**整个事务会回滚**，Sync 记录不会被创建，但 HTTP 仍然返回 200。

**证据标注**：
- ✅ 代码可证：`Sync.transaction do ... end` 包裹了整个逻辑
- 📚 外部规则：ActiveRecord 事务回滚机制是 Rails 框架特性

---

## 7. 可观测性手段

### 7.1 Sentry 异常上报

**关键上报点**：

| 位置 | 异常类型 | 上报方式 | HTTP 状态 |
|-----|---------|---------|----------|
| `webhooks_controller.rb:17,34` | Plaid 签名验证/Provider 缺失/JSON 解析 | 控制器 rescue 捕获 | 400 |
| `webhooks_controller.rb:48,52` | Stripe JSON/签名异常 | 特定 rescue 捕获 | 400 |
| `webhooks_controller.rb` (未捕获) | Stripe 其他异常 (NoMethodError, Redis 等) | Rails 全局异常处理 | 500 |
| `webhook_processor.rb:32-35` | Plaid 处理阶段异常 (sync_later 失败等) | WebhookProcessor 内部 rescue | 200 |
| `webhook_processor.rb:52-54` | Plaid Item 缺失 | 主动上报 | 200 |
| `sync.rb:73-76` | Sync 执行失败 | Sync 内部 rescue | - |
| `sync.rb:146-149` | Post-sync 钩子异常 | Sync 内部 rescue | - |

**证据标注**：
- ✅ 代码可证：所有 `Sentry.capture_exception` 调用点均可在代码中直接找到
- 📚 外部规则：未捕获异常会触发 Rails 全局异常处理（如果配置了 Sentry 集成）

**上下文标签示例**：
```ruby
Sentry.capture_exception(MissingItemError.new(...)) do |scope|
  scope.set_tags(plaid_item_id: item_id)
end
```

**证据标注**：
- ✅ 代码可证：`handle_missing_item` 方法中明确定义

### 7.2 观测性盲区

**Plaid 处理阶段异常**：sync_later 失败、数据库 update! 失败等被内部吞掉，返回 200，**外部供应商不会重试**，只能依赖 Sentry 告警发现问题。

**Stripe 未捕获异常**：`NoMethodError`（Provider 为 nil）、`Redis::CannotConnectError` 等异常不会被 Stripe 控制器的特定 rescue 捕获，依赖 Rails 全局异常处理。

**任务入队成功但执行失败**：如 Stripe API 拉取事件失败、订阅处理失败等，仅在 Job 日志中有记录。

**静默丢弃的事件**：未匹配到路由的 webhook 仅记录 warn 日志，不上报 Sentry。

**证据标注**：
- ✅ 代码可证：通过分析异常捕获路径和日志调用点可直接验证

### 7.3 日志记录

**关键日志点**：
- 未处理的 webhook 类型（warn 级别）
- Sync 状态转换（info 级别）
- Stripe 事件处理开始（info 级别）
- Sync 窗口扩展（info 级别）
- Post-sync 错误（error 级别）

**证据标注**：
- ✅ 代码可证：所有 `Rails.logger` 调用点均可在代码中直接找到

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

**证据标注**：
- ✅ 代码可证：`report_warnings` 方法中明确定义

---

## 8. Plaid vs Stripe：异常处理与返回码深度对比

### 8.1 Plaid 两层异常捕获机制

**代码结构 (`webhooks_controller.rb:5-19` + `webhook_processor.rb:12-35`)：**

```ruby
# 控制器层：第一层捕获
def plaid
  webhook_body = request.body.read
  plaid_verification_header = request.headers["Plaid-Verification"]
  client = Provider::Registry.plaid_provider_for_region(:us)
  client.validate_webhook!(plaid_verification_header, webhook_body)  # ↑ 以上异常 → 400
  
  PlaidItem::WebhookProcessor.new(webhook_body).process  # ↓ 内部吞异常 → 200
  
  render json: { received: true }, status: :ok
rescue => error  # 仅捕获 process 调用之前的异常
  Sentry.capture_exception(error)
  render json: { error: "Invalid webhook: #{error.message}" }, status: :bad_request
end

# WebhookProcessor 层：第二层捕获（内部吞异常）
def process
  # ... 业务逻辑：sync_later, update! 等 ...
rescue => e
  # 只上报 Sentry，不重新抛出
  Sentry.capture_exception(e)
end
```

**证据标注**：
- ✅ 代码可证：两层 rescue 结构可直接从代码中验证

**Plaid 异常路径分界图**：

```
Plaid 请求
    ↓
┌─ 控制器层 ──────────────────────────────────────────┐
│  1. request.body.read                               │
│  2. Provider::Registry.plaid_provider_for_region    │ → 异常 → rescue → 400
│  3. client.validate_webhook!                        │
│  4. WebhookProcessor.new (JSON.parse)               │
├─────────────────────────────────────────────────────┤
│  5. processor.process                               │
│     ┌─ WebhookProcessor.process ─────────────────┐  │
│     │  case [webhook_type, webhook_code]         │  │
│     │  when ... → plaid_item.sync_later          │  │ → 异常 → 内部 rescue → Sentry
│     │  when ... → plaid_item.update!             │  │
│     │  rescue => e → Sentry.capture_exception(e) │  │
│     └────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘
    ↓
  返回 200
```

**证据标注**：
- ✅ 代码可证：异常分界可通过代码分析直接验证

### 8.2 Stripe 单层特定异常捕获

**代码结构 (`webhooks_controller.rb:37-56`)：**

```ruby
def stripe
  stripe_provider = Provider::Registry.get_provider(:stripe)  # ↑ 可能返回 nil
  
  begin
    webhook_body = request.body.read
    sig_header = request.env["HTTP_STRIPE_SIGNATURE"]
    stripe_provider.process_webhook_later(webhook_body, sig_header)
    head :ok
  rescue JSON::ParserError => error  # ← 仅捕获 JSON 解析错误
    Sentry.capture_exception(error)
    head :bad_request
  rescue Stripe::SignatureVerificationError => error  # ← 仅捕获签名错误
    Sentry.capture_exception(error)
    head :bad_request
  end
  # ↓ 其他异常 (NoMethodError, Redis 等) 直接抛出 → 500
end
```

**证据标注**：
- ✅ 代码可证：仅捕获两种特定异常的结构可直接从代码中验证

### 8.3 异常场景行为矩阵（与代码完全一致）

| 异常场景 | Plaid 行为 | Stripe 行为 |
|---------|-----------|------------|
| 签名验证失败 | 控制器 rescue → Sentry → **400** | 特定 rescue → Sentry → **400** |
| JSON 解析失败 (initialize 中) | 控制器 rescue → Sentry → **400** | 特定 rescue → Sentry → **400** |
| Provider 配置缺失 (返回 nil) | 控制器 rescue → Sentry → **400** | 未捕获 → Rails 500 → **500** |
| 任务入队失败 (Redis 问题) | WebhookProcessor 内部 → Sentry → **200** | 未捕获 → Rails 500 → **500** |
| 数据库操作失败 (update!/create!) | WebhookProcessor 内部 → Sentry → **200** | Job 中异常 → Sidekiq 重试 → **200** |
| 业务处理阶段异常 (sync 逻辑) | WebhookProcessor 内部 → Sentry → **200** | Job 中异常 → Sidekiq 重试 → **200** |
| 未找到对应资源 (PlaidItem/Family) | WebhookProcessor 内部 → Sentry → **200** | Job 中异常 → Sidekiq 重试 → **200** |

**证据标注**：
- ✅ 代码可证：每一行的行为均可通过代码路径分析直接验证

### 8.4 设计意图对比

| 维度 | Plaid 策略 | Stripe 策略 |
|-----|-----------|------------|
| 核心目标 | **保持端点健康**：尽可能返回 200，避免 Plaid 标记端点为不健康 | **快速响应**：仅验证必要信息，快速返回 200/400，异步处理 |
| 异常哲学 | **容错优先**：即使内部处理完全失败，也告诉 Plaid "已收到"，依赖 Sentry 人工介入 | **诚实反馈**：签名/格式错误返回 400，系统错误返回 500 |
| 重试依赖 | 仅签名验证等前置错误会触发 Plaid 重试；处理阶段错误**无外部重试** | 500 错误触发 Stripe 重试；Job 异常依赖 Sidekiq 重试 |
| 可观测性 | 所有异常都上报 Sentry，但处理阶段异常返回 200，需主动监控 | 系统错误返回 500，可通过 HTTP 监控发现；Job 异常依赖 Sidekiq |

**证据标注**：
- ✅ 代码可证：设计意图可从代码结构和注释（如 "To always ensure we return a 200 to Plaid"）推断
- ⚠️ 注意："核心目标"和"异常哲学"为基于代码的推断，非代码字面量表达

### 8.5 外部供应商重试行为（非代码可证部分）

> **⚠️ 重要说明**：以下关于外部供应商重试行为的描述**无法从当前代码库直接验证**，仅为基于行业常识的合理推断，不作为定论。

| 供应商 | 4xx 响应（推测） | 5xx 响应（推测） | 关键影响（推测） |
|-------|----------------|----------------|-----------------|
| Plaid | 可能重试 | 可能重试 | 处理阶段异常返回 200，**推测 Plaid 不会重试**，数据可能永久丢失 |
| Stripe | 通常不重试（认为是客户端错误） | 通常重试（指数退避） | Provider 缺失/Redis 故障返回 500，**推测 Stripe 会自动重试** |

**证据标注**：
- 📚 外部规则：供应商重试策略需查阅 Plaid/Stripe 官方文档确认
- ❗ 不确定：代码库中无任何关于供应商重试策略的配置或注释，以上为合理推断

> **最严重的潜在风险（基于上述推测）**：Plaid webhook 签名验证通过后，如果 sync_later 因 Redis 故障入队失败，系统返回 200，**如果 Plaid 对 200 响应不重试**，该次交易更新通知可能永久丢失，只能通过后续的其他 webhook 或定时同步补回。

**证据标注**：
- ⚠️ 风险提示：此风险的成立依赖于 Plaid 的重试策略，需查阅官方文档确认

---

## 9. 架构特点与权衡

| 决策 | 优点 | 缺点 |
|-----|-----|-----|
| Plaid 两层异常捕获 | 端点健康度高，几乎不会被 Plaid 标记为不健康 | 处理阶段异常返回 200，无外部重试，数据可能丢失 |
| Plaid WebhookProcessor 内部吞异常 | 代码注释明确说明是为了保证 200 返回 | 异常类型被掩盖，无法通过 HTTP 状态码区分问题 |
| Stripe 异步拉取完整事件 | 控制器快速返回 200，降低超时风险；thin event 体积小 | 需要额外 API 调用拉取完整事件 |
| Stripe 仅捕获特定异常 | 签名/格式错误快速反馈，不触发不必要的重试 | 系统错误返回 500，观测性依赖 Rails 全局处理 |
| Sync 去重 + 窗口扩展 | 避免重复 sync，合并相邻时间范围 | 逻辑复杂度增加 |
| Registry 静默返回 nil | 配置缺失时不崩溃 | 延迟失败，增加调试复杂度 |

**证据标注**：
- ✅ 代码可证：各决策的代码实现均可直接验证
- 💡 架构分析："优点"和"缺点"为基于代码的架构分析结论

---

## 10. 扩展点与建议

### 10.1 潜在扩展方向
1. **新增供应商**：在 `Provider::Registry` 添加新方法，实现对应 `validate_webhook!` 和处理器
2. **新事件类型**：在对应 Processor 的 case 语句中增加新分支
3. **死信队列**：为 webhook 处理失败增加专门的死信队列和人工重试界面

**证据标注**：
- 💡 架构建议：基于现有架构的合理扩展方向

### 10.2 缺陷修复建议（按优先级）

**高优先级**：
1. **Plaid 处理阶段异常补偿**：对于 sync_later 入队失败等场景，考虑引入本地重试机制或将事件存入数据库待处理，避免数据永久丢失
2. **统一 Stripe 异常处理**：在 Stripe 控制器增加 `rescue => error` 兜底，或在 Registry 中增加 nil 检查
3. **Registry 失败快速**：配置缺失时抛出明确异常而非返回 nil，便于及早发现问题

**中优先级**：
4. **未处理事件告警**：对未匹配路由的 webhook 类型增加 Sentry warning 级别上报
5. **幂等性校验**：增加 webhook event_id 去重机制，避免重复处理
6. **Plaid 处理异常监控**：对 WebhookProcessor 内部捕获的异常设置专门的 Sentry 告警规则

**证据标注**：
- 💡 修复建议：基于代码分析的合理改进建议

### 10.3 可观测性增强建议
1. 增加 webhook 处理延迟指标（从接收到处理完成的时间）
2. 按供应商/事件类型统计成功率和失败率
3. 为 Plaid 200 但内部异常的场景增加专门的 metrics 指标
4. 为 Stripe 500 错误场景增加专门的告警规则

**证据标注**：
- 💡 可观测性建议：基于现有观测性盲区的合理改进建议

---

## 附录：不确定性声明

本报告中所有标记为 📚 外部规则或 ⚠️ 不确定的结论，建议在进行生产环境决策前：
1. 查阅对应供应商的官方 API 文档确认重试策略
2. 通过实际测试验证异常场景下的行为
3. 评估数据丢失风险并设计相应的补偿机制
