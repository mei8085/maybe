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

> ⚠️ **本章已修订** - 重点修正了乱序覆盖风险分析，并补充了 Job 失败重试与异常分支的影响分析

### 2.1 乱序事件的覆盖风险分析

**核心问题**：代码无条件覆盖写入，晚到的旧事件会产生严重的状态回滚

**真实场景示例**：

| 事件发生时间 | 事件描述 | 事件类型 | Stripe 端状态流转 | Sidekiq 执行顺序 | 处理后本地状态 |
|--------------|---------|----------|-------------------|------------------|----------------|
| T09:55 | 试用转正式 | `customer.subscription.updated` | trialing → active | ③ 最后执行（乱序） | active ❌ 状态被错误回滚！|
| T10:00 | 其他更新 | `customer.subscription.updated` | active → active | ① 最早执行 | active ✅ |
| T10:05 | 用户取消订阅 | `customer.subscription.updated` | active → canceled | ② 中间执行 | canceled ✅ |

> 注：Sidekiq 执行顺序与事件发生时间无关，受网络延迟、重试机制、队列优先级等因素影响。

**问题根源** (`subscription_event_processor.rb:7-14`)：
```ruby
family.subscription.update(
  stripe_id: subscription.id,
  status: subscription.status,  # ⚠️ 无条件覆盖！
  interval: subscription_details.plan.interval,
  amount: subscription_details.plan.amount / 100.0,
  currency: subscription_details.plan.currency.upcase,
  current_period_ends_at: Time.at(subscription_details.current_period_end)
)
```

**覆盖风险的本质**：
1. **无时间戳检查**：不比较事件创建时间与本地记录的更新时间
2. **全字段覆盖**：所有 6 个字段全部无条件写入，没有部分更新策略
3. **队列执行顺序不可控**：Sidekiq 不保证严格按入队顺序执行（网络延迟、重试、队列优先级等因素都可能导致乱序）

---

### 2.2 当前代码已存在的保护机制

#### ✅ 保护 1：重复事件的幂等性保障

```ruby
family.subscription.update(...)
```
- 使用数据库 `UPDATE` 操作而非 `INSERT`
- **相同事件**重复处理：覆盖写入相同数据 → 无副作用
- 适用场景：Stripe 重复投递同一个 `event_id`

#### ✅ 保护 2：唯一索引约束防重复订阅

```ruby
t.index ["family_id"], name: "index_subscriptions_on_family_id", unique: true
```
- 每个 Family 只能有一条 Subscription 记录
- 从数据库层面防止重复订阅记录（但不防状态回滚）

#### ✅ 保护 3：从 Stripe API 重新拉取完整事件

```ruby
def retrieve_event(event_id)
  client.v1.events.retrieve(event_id)
end
```
- 不直接信任 webhook 推送的 payload
- 从 Stripe 服务器拉取权威事件数据
- 防止 webhook 数据被篡改或损坏
- ❗ **但拉取的仍是历史快照，不是订阅的当前状态**

---

### 2.3 缺失的去重与乱序保护机制

当前代码**完全缺失**以下关键保护：

| 缺失机制 | 风险 |
|---------|------|
| ❌ 事件处理记录表 | 无法识别已处理过的 event_id，重复事件会重复执行 UPDATE |
| ❌ 事件时间戳比较 | 无法判断"旧事件晚到"情况，旧状态覆盖新状态 |
| ❌ Stripe 最新状态兜底查询 | 直接使用事件快照，不查询订阅当前真实状态 |
| ❌ 数据库乐观锁 | 并发更新时可能产生竞态条件 |
| ❌ 按事件类型差异化处理 | `customer.subscription.deleted` 事件与 `updated` 事件使用相同逻辑 |

---

### 2.4 Job 失败重试与异常分支的影响

#### Job 重试配置分析

`ApplicationJob` 基类配置 (`app/jobs/application_job.rb:1-5`)：
```ruby
class ApplicationJob < ActiveJob::Base
  retry_on ActiveRecord::Deadlocked   # 死锁会自动重试
  discard_on ActiveJob::DeserializationError  # 反序列化错误直接丢弃
  queue_as :low_priority
end
```

`StripeEventHandlerJob` 自身配置 (`app/jobs/stripe_event_handler_job.rb:72`)：
```ruby
queue_as :default  # 覆盖基类配置，使用 default 队列
```

> 事实校对：`StripeEventHandlerJob` **继承**了 `ApplicationJob` 的 `retry_on` 和 `discard_on` 配置，并**覆盖**了队列名为 `:default`。

#### 异常场景分析

**场景 1：网络波动导致 Stripe API 调用失败**

```ruby
# stripe.rb:85-87
def retrieve_event(event_id)
  client.v1.events.retrieve(event_id)  # ⚠️ 无异常捕获！
end
```
- **后果**：网络超时、Stripe API 5xx 错误 → Job 抛出未捕获异常 → 触发 Sidekiq 默认重试机制（最多 25 次，指数退避）
- **风险**：重试成功时，事件数据可能已过时 → 重试时可能覆盖更新的状态

**场景 2：Family 未找到异常**

```ruby
# subscription_event_processor.rb:5
raise Error, "Family not found for Stripe customer ID: #{subscription.customer}" unless family
```
- **后果**：Customer ID 关联延迟（例如订阅创建事件先于用户注册完成） → Job 失败进入重试队列
- **风险**：重试成功时，后续事件可能已先处理完成 → 状态回滚

**场景 3：数据库死锁**

```ruby
retry_on ActiveRecord::Deadlocked
```
- **保护机制**：Rails 内置的 `retry_on` 会捕获死锁异常并重试
- **风险**：重试窗口内其他事件可能已更新状态 → 死锁解除后覆盖新状态

**场景 4：反序列化错误**

```ruby
discard_on ActiveJob::DeserializationError
```
- **后果**：直接丢弃，永不重试
- **严重风险**：该事件对应的状态变更永久丢失

---

### 2.5 乱序事件的兜底策略（当前）

**当前依赖的兜底机制（非常有限）：**

1. **状态覆盖写入的"最终可能正确"假设**
   - 如果所有事件最终都成功执行
   - 且最后执行的是时间上最新的事件
   - 那么最终状态是正确的
   - ❗ 注意：如果最新事件先失败后重试，中间夹杂旧事件 → 最终状态可能错误

2. **Stripe webhook 的最佳实践**
   - Stripe 保证至少一次投递（at-least-once delivery）
   - Stripe 不保证投递顺序
   - 官方推荐：处理逻辑必须具备幂等性和乱序容忍能力

---

## 3. 架构设计总结

### 3.1 处理流程图

```
Stripe Webhook
     ↓
[WebhooksController#stripe]
     ↓ 签名验证
[Provider::Stripe#process_webhook_later]
     ↓ 异步入队
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
| **异步处理** | 签名验证同步执行，业务逻辑异步执行 | 快速响应，避免 Stripe 重试 |
| **重新拉取事件** | 不信任 webhook 数据，从 Stripe API 重新拉取 | 确保数据完整性和权威性 |
| **UPDATE 幂等** | 使用 UPDATE 而非 INSERT | 天然支持重复事件处理 |
| **按类型分派** | 不同事件类型使用不同 Processor | 符合开闭原则，易于扩展 |

---

## 4. 相关文件清单

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
| `app/jobs/application_job.rb` | Job 基类（重试配置） |

---

## 5. 本轮修订清单

### 第一轮修订（已有）

#### 格式修复
- ✅ 修正了 2.1 节乱序场景示例表的格式断裂问题（`subscription.updated` 缺少闭合反引号）
- ✅ 修正了表格列标题表述，使含义更清晰
- ✅ 补齐了所有未闭合的强调标记（`**` 加粗标记）
- ✅ 统一了代码引用格式的一致性

#### 事实校对
- ✅ 修正了 Job 队列名的错误：`StripeEventHandlerJob` 实际使用 `queue_as :default` 而非 `low_priority`
- ✅ 明确说明 Job 重试配置是继承自 `ApplicationJob`
- ✅ 修正了事件类型的完整命名：`customer.subscription.updated`（补上 `customer.` 前缀）
- ✅ 调整了乱序示例表的时间顺序，使场景更符合真实情况
- ✅ 添加了注释说明"处理顺序"指的是 Sidekiq 执行顺序

#### 表述优化
- ✅ 移除了容易产生歧义的双重放大镜 emoji
- ✅ 修正了"事件已过时"等不完整表述
- ✅ 补充了 Sidekiq 默认重试次数的表述（"最多 25 次，指数退避"）
- ✅ 优化了覆盖风险本质三点的表述，使其更准确
- ✅ 删除了第 4 章实现建议，仅保留分析内容
- ✅ 新增第 4 章改为相关文件清单

---

### 第二轮修订（本次新增）

#### 格式修复
- ✅ 修正了第 323 行未闭合的加粗标记：`**当前依赖的兜底机制（非常有限）**`
- ✅ 优化了流程图表述，将"异步队列"改为"异步入队"，表述更准确

#### 事实表述修正
- ✅ 重构了乱序示例表：将"时间线"拆分为"事件发生时间"和"Sidekiq 执行顺序"两列，彻底消除歧义
- ✅ 修正了表格注释表述：删除"上表按处理顺序"的歧义表述，改为明确说明 Sidekiq 执行顺序的影响因素
- ✅ 优化了时间戳检查表述：将"不比较事件时间"改为"不比较事件创建时间"，更准确
- ✅ 重构了 Job 重试配置分析：分开展示基类配置和子类配置，逻辑更清晰
- ✅ 将 4 个异常场景统一编号为"场景 1"至"场景 4"，结构更一致
- ✅ 优化了兜底策略中的提醒表述：在 ❗ 后添加空格并改为"注意："引导

#### 表述精炼
- ✅ 删除了 2.4 节标题中的"🔍🔍" emoji，保持格式统一
- ✅ 统一了异常场景的表述风格：每行以"后果"和"风险"开头，结构清晰
- ✅ 删除了流程图中容易引起误解的"从 Stripe API 拉取完整事件"前的多余空格
- ✅ 优化了关键设计决策表的说明列表述，删除多余空格
