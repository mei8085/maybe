# 商户自动识别链路报告

## 概述

本文档详细描述了从原始交易描述到最终匹配商户记录的完整识别链路，包括跨模块的判定规则、优先级机制和兜底分支。

---

## 一、商户数据模型层级

### 1.1 模型继承关系

```
Merchant (基类)
├── FamilyMerchant (用户创建的商户)
└── ProviderMerchant (第三方/AI识别的商户)
    ├── source: plaid (Plaid API 提供)
    ├── source: synth (系统内置)
    └── source: ai (AI 自动识别)
```

**文件位置**:
- `app/models/merchant.rb:1-10`
- `app/models/provider_merchant.rb:1-6`

### 1.2 关键属性

| 属性 | 说明 |
|------|------|
| `name` | 商户名称 (唯一约束：family_id+name 或 source+name) |
| `type` | 商户类型 (FamilyMerchant / ProviderMerchant) |
| `source` | 商户来源 (仅 ProviderMerchant: plaid/synth/ai) |
| `website_url` | 商户网站 URL |
| `logo_url` | 商户 Logo URL |

---

## 二、完整识别链路总览

```
交易数据入口
    │
    ├─→ 1. Plaid 同步阶段 (PlaidEntry::Processor)
    │       ├─→ 匹配 Plaid merchant_entity_id
    │       └─→ 创建/复用 ProviderMerchant (source: plaid)
    │
    ├─→ 2. 规则引擎阶段 (Rule::ActionExecutor)
    │       ├─→ SetTransactionMerchant (规则手动指定)
    │       └─→ AutoDetectMerchants (触发 AI 自动识别)
    │               └─→ 异步调度: AutoDetectMerchantsJob
    │
    └─→ 3. AI 自动识别阶段 (Family::AutoMerchantDetector)
            ├─→ 过滤 enrichable 交易
            ├─→ 调用 OpenAI LLM API
            ├─→ 匹配用户已有商户 (优先级高)
            ├─→ 创建 AI ProviderMerchant (兜底)
            └─→ 锁定 merchant_id 属性
```

---

## 三、阶段一：Plaid 同步自动识别

### 3.1 处理流程 (`app/models/plaid_entry/processor.rb:39-45`)

```ruby
if merchant
  entry.transaction.enrich_attribute(
    :merchant_id,
    merchant.id,
    source: "plaid"
  )
end
```

### 3.2 商户匹配逻辑 (`app/models/plaid_entry/processor.rb:80-94`)

```ruby
def merchant
  merchant_id = plaid_transaction["merchant_entity_id"]
  merchant_name = plaid_transaction["merchant_name"]

  return nil unless merchant_id.present? && merchant_name.present?

  ProviderMerchant.find_or_create_by!(
    source: "plaid",
    name: merchant_name,
  ) do |m|
    m.provider_merchant_id = merchant_id
    m.website_url = plaid_transaction["website"]
    m.logo_url = plaid_transaction["logo_url"]
  end
end
```

**判定规则**:
1. **必须同时满足**: Plaid 返回 `merchant_entity_id` AND `merchant_name`
2. **查找/创建**: 按 `source: "plaid"` + `name` 唯一键查找或创建
3. **兜底分支**: 任一条件不满足 → 跳过，留待后续阶段处理

---

## 四、阶段二：规则引擎触发

### 4.1 规则动作类型

#### A. SetTransactionMerchant (手动指定商户)
**文件**: `app/models/rule/action_executor/set_transaction_merchant.rb:1-27`

```ruby
def execute(transaction_scope, value: nil, ignore_attribute_locks: false)
  merchant = family.merchants.find_by_id(value)
  return unless merchant

  scope = ignore_attribute_locks ? transaction_scope : transaction_scope.enrichable(:merchant_id)

  scope.each do |txn|
    txn.enrich_attribute(:merchant_id, merchant.id, source: "rule")
  end
end
```

**判定规则**:
1. **前置检查**: 商户必须存在于家庭商户列表中
2. **锁定检查**: 默认仅处理 `enrichable(:merchant_id)` 的交易
3. **强制覆盖**: `ignore_attribute_locks: true` 可绕过锁定机制

#### B. AutoDetectMerchants (触发 AI 批量识别)
**文件**: `app/models/rule/action_executor/auto_detect_merchants.rb:1-23`

```ruby
def execute(transaction_scope, value: nil, ignore_attribute_locks: false)
  enrichable_transactions = transaction_scope.enrichable(:merchant_id)
  return if enrichable_transactions.empty?

  enrichable_transactions.in_batches(of: 20).each_with_index do |transactions, idx|
    rule.family.auto_detect_transaction_merchants_later(transactions)
  end
end
```

**调度机制**:
- **批量大小**: 每批 20 笔交易
- **异步执行**: 通过 `AutoDetectMerchantsJob` 放入 `medium_priority` 队列
- **文件**: `app/jobs/auto_detect_merchants_job.rb:1-7`

---

## 五、阶段三：AI 自动识别核心逻辑

### 5.1 入口与范围过滤 (`app/models/family/auto_merchant_detector.rb:1-98`)

```ruby
def scope
  family.transactions.where(id: transaction_ids, merchant_id: nil)
                     .enrichable(:merchant_id)
                     .includes(:merchant, :entry)
end
```

**前置过滤条件**:
1. ✅ 在指定 `transaction_ids` 范围内
2. ✅ 当前 `merchant_id` 为 `nil` (未匹配)
3. ✅ `merchant_id` 属性未被锁定 (`enrichable`)

### 5.2 LLM API 调用 (OpenAI)
**文件**: `app/models/provider/openai/auto_merchant_detector.rb:1-146`

#### 输入数据结构

```ruby
def transactions_input
  scope.map do |transaction|
    {
      id: transaction.id,
      amount: transaction.entry.amount.abs,
      classification: transaction.entry.classification,
      description: transaction.entry.name,          # 原始交易描述
      merchant: transaction.merchant&.name
    }
  end
end

def user_merchants_input
  family.merchants.map do |merchant|
    { id: merchant.id, name: merchant.name }
  end
end
```

#### LLM 系统提示指令

```
You are an assistant to a consumer personal finance app.

Closely follow ALL the rules below while auto-detecting business names and website URLs:

- Return 1 result per transaction
- Correlate each transaction by ID (transaction_id)
- Do not include the subdomain in the business_url
- User merchants are considered "manual" user-generated merchants and should only be used in 100% clear cases
- Be slightly pessimistic. We favor returning "null" over returning a false positive.
- NEVER return a name or URL for generic transaction names (e.g. "Paycheck", "Laundromat")

Determining a value:

1. First attempt to determine the name + URL from your knowledge of global businesses
2. If no certain match, attempt to match one of the user-provided merchants
3. If no match, return "null"

Confidence threshold: 80%+ → return value, else return "null"
```

### 5.3 核心匹配与兜底分支

```ruby
scope.each do |transaction|
  auto_detection = result.data.find { |c| c.transaction_id == transaction.id }

  # 分支 1: 优先匹配用户已有的商户 (FamilyMerchant)
  merchant_id = user_merchants_input.find do |m|
    m[:name] == auto_detection&.business_name
  end&.dig(:id)

  # 分支 2: 兜底 - 创建 AI 识别的 ProviderMerchant
  if merchant_id.nil? && auto_detection&.business_url.present? && auto_detection&.business_name.present?
    ai_provider_merchant = ProviderMerchant.find_or_create_by!(
      source: "ai",
      name: auto_detection.business_name,
      website_url: auto_detection.business_url,
    ) do |pm|
      pm.logo_url = "#{default_logo_provider_url}/#{auto_detection.business_url}"
    end
  end

  merchant_id = merchant_id || ai_provider_merchant&.id

  # 分支 3: 最终兜底 - 识别失败则不设置 merchant_id
  if merchant_id.present?
    transaction.enrich_attribute(:merchant_id, merchant_id, source: "ai")
  end

  # 无论成功与否，锁定属性防止重复识别
  transaction.lock_attr!(:merchant_id)
end
```

---

## 六、匹配优先级与判定规则总结

### 6.1 商户匹配优先级 (从高到低)

| 优先级 | 商户类型 | 来源 | 说明 |
|--------|----------|------|------|
| 1️⃣ 最高 | FamilyMerchant | 用户手动创建 | LLM 识别结果与用户已有商户精确匹配 |
| 2️⃣ 高 | ProviderMerchant | Plaid API | 交易同步时 Plaid 提供的商户实体 |
| 3️⃣ 中 | ProviderMerchant | AI (OpenAI) | LLM 识别后创建的新商户 |
| 4️⃣ 最低 | FamilyMerchant | Rule 规则 | 用户配置的规则手动指定 |

### 6.2 LLM 内部判定层级

```
LLM 判定流程:
    │
    ├─→ 第一层: 基于全局知识库识别知名商户
    │       (如: "amzn 123" → Amazon / amazon.com)
    │
    ├─→ 第二层: 匹配用户提供的商户列表
    │       (仅 100% 明确匹配时使用)
    │
    └─→ 第三层: 置信度 < 80% → 返回 null
```

### 6.3 防止重复处理机制

**Enrichable 模块** (`app/models/concerns/enrichable.rb:1-91`)

```ruby
# 锁定属性，防止后续规则/AI 覆盖
def lock_attr!(attr)
  update!(locked_attributes: locked_attributes.merge(attr.to_s => Time.current))
end

# 仅处理未锁定的属性
scope :enrichable, ->(attrs) {
  attrs = Array(attrs).map(&:to_s)
  where.not(Arel.sql("#{table_name}.locked_attributes ?| array[:keys]"), keys: attrs)
}
```

**关键行为**:
- AI 识别完成后 **无论成功与否**，都会执行 `lock_attr!(:merchant_id)`
- 被锁定的交易不会再进入后续的自动识别流程
- 用户手动编辑具有最高优先级，可解锁并覆盖

---

## 七、兜底分支汇总

| 阶段 | 兜底条件 | 处理方式 |
|------|----------|----------|
| **Plaid 同步** | 缺少 merchant_entity_id 或 merchant_name | 跳过，merchant_id 保持 nil |
| **规则执行** | 商户 ID 不存在 | 静默跳过，不做处理 |
| **LLM 识别** | 置信度 < 80% | 返回 business_name: null, business_url: null |
| **LLM 识别** | 通用交易名称 (Paycheck, Grocery store 等) | 返回 null |
| **商户匹配** | LLM 返回 null | 不设置 merchant_id，但锁定属性 |
| **商户匹配** | 未匹配到用户商户但有 business_url | 创建 ProviderMerchant (source: ai) |

---

## 八、数据流转追踪

### 8.1 DataEnrichment 审计日志

每次 `enrich_attribute` 调用都会创建审计记录:

```ruby
def log_enrichment(attribute_name:, attribute_value:, source:, metadata: {})
  de = DataEnrichment.find_or_create_by(
    enrichable: self,
    attribute_name: attribute_name,
    source: source,
  )
  de.value = attribute_value
  de.metadata = metadata
  de.save
end
```

**Source 来源枚举**:
- `plaid` - Plaid API 自动识别
- `rule` - 规则引擎设置
- `ai` - OpenAI LLM 自动识别
- `user` - 用户手动编辑 (最高优先级)

---

## 九、关键约束与边界条件

### 9.1 批量限制
- OpenAI API 单次请求最大 25 笔交易 (`app/models/provider/openai.rb:31`)
- 规则调度按每批 20 笔异步执行

### 9.2 唯一性约束
- FamilyMerchant: `family_id + name` 唯一
- ProviderMerchant: `source + name` 唯一

### 9.3 幂等性保证
- ProviderMerchant 使用 `find_or_create_by!` 确保幂等
- `locked_attributes` 机制防止重复处理

---

## 附录：相关文件索引

| 文件路径 | 说明 |
|----------|------|
| `app/models/family/auto_merchant_detector.rb` | AI 识别核心逻辑 |
| `app/models/provider/openai/auto_merchant_detector.rb` | OpenAI LLM API 封装 |
| `app/models/plaid_entry/processor.rb` | Plaid 同步商户识别 |
| `app/models/concerns/enrichable.rb` | 属性锁定与丰富机制 |
| `app/models/rule/action_executor/auto_detect_merchants.rb` | 规则触发 AI 识别 |
| `app/models/rule/action_executor/set_transaction_merchant.rb` | 规则设置商户 |
| `app/models/provider_merchant.rb` | 第三方商户模型 |
| `app/models/merchant.rb` | 商户基类 |
