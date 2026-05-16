# Stripe Webhook 订阅事件处理流程

## 概述

本文档梳理了 Stripe webhook 事件进入系统后的完整处理路径，包括事件分派、订阅状态写入以及重复/乱序事件的兜底机制。

---

## 1. 事件处理完整路径

### 1.1 入口点：WebhooksController

**文件**: `app/controllers/webhooks_controller.rb:37-56`

```ruby
def stripe
  stripe_provider = Provider::Registry.get_provider(:stripe)

  begin
    webhook_body = request.body.read
    sig_header = request.env["HTTP_STRIPE_SIGNATURE"]

    stripe_provider.process_webhook_later(webhook_body, sig_header)

    head :ok
  rescue JSON::ParserError => error
    # ... 错误处理
  rescue Stripe::SignatureVerificationError => error
    # ... 错误处理
  end
end
```

**处理逻辑**:
- 路由: `POST /webhooks/stripe` (`config/routes.rb:254-258`)
- 跳过 CSRF 验证和身份认证
- 读取请求体和 Stripe 签名头
- 调用 `process_webhook_later` 进行异步处理
- 立即返回 200 OK 响应

---

### 1.2 签名验证与异步队列

**文件**: `app/models/provider/stripe.rb:20-23`

```ruby
def process_webhook_later(webhook_body, sig_header)
  thin_event = client.parse_thin_event(webhook_body, sig_header, webhook_secret)
  StripeEventHandlerJob.perform_later(thin_event.id)
end
```

**关键步骤**:
1. **签名验证**: 使用 `Stripe::StripeClient#parse_thin_event` 验证 webhook 签名
2. **事件解析**: 解析出"瘦事件"（只包含事件 ID 和类型，不包含完整数据）
3. **异步队列**: 将事件 ID 投递到 Sidekiq 队列，由 `StripeEventHandlerJob` 处理

**设计意图**:
- 快速响应 Stripe，避免超时
- 签名验证在同步流程中完成，确保请求合法性
- 实际业务处理异步执行，提高系统吞吐量

---

### 1.3 事件处理 Job

**文件**: `app/jobs/stripe_event_handler_job.rb:1-9`

```ruby
class StripeEventHandlerJob < ApplicationJob
  queue_as :default

  def perform(event_id)
    stripe_provider = Provider::Registry.get_provider(:stripe)
    Rails.logger.info "Processing Stripe event: #{event_id}"
    stripe_provider.process_event(event_id)
  end
end
```

**职责**:
- 从队列中接收事件 ID
- 调用 Stripe Provider 处理完整事件

---

### 1.4 事件获取与分派

**文件**: `app/models/provider/stripe.rb:9-18`

```ruby
def process_event(event_id)
  event = retrieve_event(event_id)

  case event.type
  when /^customer\.subscription\./
    SubscriptionEventProcessor.new(event).process
  else
    Rails.logger.warn "Unhandled event type: #{event.type}"
  end
end

private

def retrieve_event(event_id)
  client.v1.events.retrieve(event_id)
end
```

**事件分派机制**:
1. **拉取完整事件**: 通过 Stripe API 重新拉取完整事件数据（避免 webhook 数据被篡改）
2. **按类型分派**:
   - `customer.subscription.*` 类型事件 → `SubscriptionEventProcessor`
   - 其他类型 → 记录警告日志（暂不处理）

**当前处理的订阅事件类型**（通过正则匹配）:
- `customer.subscription.created` - 订阅创建
- `customer.subscription.updated` - 订阅更新
- `customer.subscription.deleted` - 订阅取消
- 等等...

---

### 1.5 订阅状态写入

**文件**: `app/models/provider/stripe/subscription_event_processor.rb:1-29`

```ruby
class Provider::Stripe::SubscriptionEventProcessor < Provider::Stripe::EventProcessor
  Error = Class.new(StandardError)

  def process
    raise Error, "Family not found for Stripe customer ID: #{subscription.customer}" unless family

    family.subscription.update(
      stripe_id: subscription.id,
      status: subscription.status,
      interval: subscription_details.plan.interval,
      amount: subscription_details.plan.amount / 100.0, # Stripe returns cents, we report dollars
      currency: subscription_details.plan.currency.upcase,
      current_period_ends_at: Time.at(subscription_details.current_period_end)
    )
  end

  private

  def family
    Family.find_by(stripe_customer_id: subscription.customer)
  end

  def subscription_details
    event_data.items.data.first
  end

  def subscription
    event_data
  end
end
```

**写入的字段**:

| 字段 | 说明 | 来源 |
|------|------|------|
| `stripe_id` | Stripe 订阅 ID | `subscription.id` |
| `status` | 订阅状态 | `subscription.status` |
| `interval` | 计费周期 (month/year) | `subscription_details.plan.interval` |
| `amount` | 金额（美元） | `subscription_details.plan.amount / 100` |
| `currency` | 货币 | `subscription_details.plan.currency` |
| `current_period_ends_at` | 当前周期结束时间 | `subscription_details.current_period_end` |

**订阅状态枚举** (`app/models/subscription.rb:7-16`):
- `incomplete` - 未完成
- `incomplete_expired` - 未完成已过期
- `trialing` - 试用中
- `active` - 活跃
- `past_due` - 逾期
- `canceled` - 已取消
- `unpaid` - 未付款
- `paused` - 已暂停

---

## 2. 重复事件与乱序处理机制

### 2.1 当前机制分析

**基于现有代码的分析**:

#### ✅ 天然的幂等性保障

1. **更新操作的幂等性** (`subscription_event_processor.rb:7-15`)
   ```ruby
   family.subscription.update(...)
   ```
   - 使用数据库 `UPDATE` 操作而非 `INSERT`
   - 相同事件重复处理只会覆盖写入相同数据
   - 不会产生重复记录或数据不一致

2. **唯一索引约束** (`db/schema.rb:686`)
   ```ruby
   t.index ["family_id"], name: "index_subscriptions_on_family_id", unique: true
   ```
   - 每个 Family 只能有一个 Subscription
   - 从数据库层面防止重复订阅记录

#### ⚠️ 缺少显式的去重机制

当前代码**没有**以下机制：
- ❌ 没有 `stripe_webhook_events` 表记录已处理事件
- ❌ 没有基于事件 ID 的幂等性检查
- ❌ 没有事件时间戳比较（处理乱序）

---

### 2.2 乱序事件的兜底策略

**当前依赖的兜底机制**:

1. **Stripe API 重新拉取事件** (`stripe.rb:85-87`)
   ```ruby
   def retrieve_event(event_id)
     client.v1.events.retrieve(event_id)
   end
   ```
   - 不直接使用 webhook 推送的数据
   - 从 Stripe 服务器拉取最新的事件状态
   - 确保处理的是权威数据

2. **状态覆盖写入**
   - 后处理的事件会覆盖先处理的事件
   - 最终以 Stripe 为准（因为每次都重新拉取）
   - 即使乱序，最终状态会收敛到正确值

3. **Stripe webhook 的最佳实践**
   - Stripe 保证至少一次投递（at-least-once）
   - 不保证顺序
   - 推荐处理逻辑具备幂等性

---

## 3. 架构设计总结

### 3.1 处理流程图

```
Stripe Webhook
     ↓
[WebhooksController#stripe]
     ↓ 签名验证
[Provider::Stripe#process_webhook_later]
     ↓ 异步队列
[StripeEventHandlerJob#perform]
     ↓ 从 Stripe API 拉取完整事件
[Provider::Stripe#process_event]
     ↓ 按类型分派
[SubscriptionEventProcessor#process]
     ↓ 更新数据库
[Subscription 表]
```

### 3.2 关键设计决策

| 决策 | 说明 | 优点 |
|------|------|------|
| **异步处理** | 签名验证同步执行业务逻辑异步执行 | 快速响应，避免 Stripe 重试 |
| **重新拉取事件** | 不信任 webhook 数据，从 Stripe API 重新拉取 | 确保数据完整性和权威性 |
| **UPDATE 幂等** | 使用 UPDATE 而非 INSERT | 天然支持重复事件处理 |
| **按类型分派** | 不同事件类型使用不同 Processor | 符合开闭原则，易于扩展 |

---

## 4. 改进建议（可选）

如果需要更强的去重和乱序保障，可以考虑：

### 4.1 添加事件处理记录表

```ruby
# 新增表: stripe_webhook_events
# - event_id (string, unique index)
# - event_type (string)
# - processed_at (datetime)
# - status (string)
```

### 4.2 在 Job 中添加幂等性检查

```ruby
def perform(event_id)
  return if StripeWebhookEvent.processed?(event_id)
  
  # ... 处理逻辑
  
  StripeWebhookEvent.mark_processed!(event_id)
end
```

### 4.3 添加时间戳验证

```ruby
def process
  # 如果本地更新时间晚于事件时间，跳过处理
  return if family.subscription.updated_at > Time.at(event.created)
  
  # ... 更新逻辑
end
```

---

## 5. 相关文件清单

| 文件路径 | 说明 |
|----------|------|
| `app/controllers/webhooks_controller.rb` | Webhook 入口控制器 |
| `app/models/provider/stripe.rb` | Stripe 服务提供者 |
| `app/jobs/stripe_event_handler_job.rb` | 事件处理 Job |
| `app/models/provider/stripe/event_processor.rb` | 事件处理器基类 |
| `app/models/provider/stripe/subscription_event_processor.rb` | 订阅事件处理器 |
| `app/models/subscription.rb` | 订阅模型 |
| `app/models/family/subscribeable.rb` | Family 订阅相关扩展 |
| `config/routes.rb` | 路由配置 |
