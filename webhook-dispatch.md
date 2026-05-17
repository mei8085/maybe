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
- 所有异常捕获并上报 Sentry，保证始终返回 200/400 给外部供应商（避免被标记为不健康端点）

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
        # 创建新 sync 并异步执行
      end
    end
  end
end
```

---

## 7. 可观测性手段

### 7.1 Sentry 异常上报

**关键上报点：**
1. Webhook 控制器全局异常捕获（`webhooks_controller.rb:17,31,53`）
2. Plaid WebhookProcessor 内部异常捕获（`webhook_processor.rb:32-35`）
3. Sync 执行失败（`sync.rb:73-76`）
4. Post-sync 钩子异常（`sync.rb:146-149`）
5. Plaid Item 缺失告警（`webhook_processor.rb:52-54`）

**上下文标签示例：**
```ruby
Sentry.capture_exception(MissingItemError.new(...)) do |scope|
  scope.set_tags(plaid_item_id: item_id)
end
```

### 7.2 日志记录

**关键日志点：**
- 未处理的 webhook 类型（warn 级别）
- Sync 状态转换（info 级别）
- Stripe 事件处理开始（info 级别）
- Sync 窗口扩展（info 级别）
- Post-sync 错误（error 级别）

### 7.3 业务指标

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

## 8. 架构特点与权衡

| 决策 | 优点 | 缺点 |
|-----|-----|-----|
| Plaid 同步处理签名验证 | 签名失败立即返回 400，Plaid 会重试 | 控制器内逻辑稍重 |
| Stripe 异步拉取完整事件 | 控制器快速返回 200，降低超时风险；thin event 体积小 | 需要额外 API 调用拉取完整事件 |
| Sync 去重 + 窗口扩展 | 避免重复 sync，合并相邻时间范围 | 逻辑复杂度增加 |
| 所有异常捕获不抛出 | 保证外部供应商收到 200，端点不被标记不健康 | 问题可能被掩盖，依赖 Sentry 告警 |

---

## 9. 扩展点与建议

### 9.1 潜在扩展方向
1. **新增供应商**：在 `Provider::Registry` 添加新方法，实现对应 `validate_webhook!` 和处理器
2. **新事件类型**：在对应 Processor 的 case 语句中增加新分支
3. **死信队列**：为 webhook 处理失败增加专门的死信队列和人工重试界面

### 9.2 可观测性增强建议
1. 增加 webhook 处理延迟指标（从接收到处理完成的时间）
2. 按供应商/事件类型统计成功率和失败率
3. 增加 webhook 幂等性校验（处理过的 event_id 不重复处理）
