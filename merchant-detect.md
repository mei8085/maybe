# 商户自动识别链路报告

## 概述

本文档详细描述了从原始交易描述到最终匹配商户记录的完整识别链路，包括跨模块的判定规则、执行顺序、优先级机制和兜底分支。

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

## 二、完整识别执行时序总览

### 2.1 执行顺序流程图

```
交易数据入口
    │
    ├─→ 阶段 1: Plaid 同步识别 [最先执行]
    │       时机: 交易同步时自动执行
    │       条件: Plaid 返回 merchant_entity_id + merchant_name
    │       来源: plaid
    │       行为: enrich_attribute + 隐式不锁定（依赖后续阶段锁定）
    │
    ├─→ 阶段 2: 用户手动编辑 [最高优先级，可随时执行]
    │       时机: 用户通过 API 编辑交易
    │       条件: 用户提交 merchant_id 参数
    │       行为: 直接 save + lock_saved_attributes! 锁定
    │       注意: 不记录 DataEnrichment 审计日志
    │
    └─→ 阶段 3: 规则引擎执行 [第二优先级]
            │
            ├─→ 动作 A: SetTransactionMerchant (规则直接指定)
            │       时机: 规则匹配触发
            │       默认条件: merchant_id 未锁定 (enrichable)
            │       强制条件: ignore_attribute_locks: true 可绕过锁定
            │       来源: rule
            │       行为: enrich_attribute + 不自动锁定
            │
            └─→ 动作 B: AutoDetectMerchants (触发 AI 批量识别)
                    时机: 规则匹配触发
                    条件: merchant_id 未锁定 (enrichable)
                    行为: 调度异步任务 (每批 20 笔)
                        ↓
                        └─→ 阶段 4: AI 自动识别 [最后执行]
                                时机: 异步任务执行
                                条件: merchant_id == nil AND 未锁定
                                来源: ai
                                行为: 匹配商户 OR 创建 AI 商户 OR 识别失败
                                关键: 无论成功与否，强制 lock_attr!(:merchant_id)
```

---

## 三、阶段 1: Plaid 同步自动识别

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
1. **前置条件**: Plaid 必须同时返回 `merchant_entity_id` AND `merchant_name`
2. **查找/创建**: 按 `source: "plaid"` + `name` 唯一键查找或创建 ProviderMerchant
3. **兜底分支**: 任一条件不满足 → 返回 nil，不设置 merchant_id
4. **锁定行为**: 仅设置值，不执行 `lock_attr!`，属性保持可 enrichable 状态

---

## 四、阶段 2: 用户手动编辑（最高优先级）

### 4.1 处理流程 (`app/controllers/api/v1/transactions_controller.rb:107-121`)

```ruby
def update
  if @entry.update(entry_params_for_update)
    @entry.sync_account_later
    @entry.lock_saved_attributes!  # 锁定所有被修改的属性
    # ...
  end
end
```

### 4.2 锁定机制 (`app/models/concerns/enrichable.rb:69-73`)

```ruby
def lock_saved_attributes!
  saved_changes.keys.reject do |attr|
    ignored_enrichable_attributes.include?(attr)
  end.each do |attr|
    lock_attr!(attr)
  end
end
```

**关键行为**:
1. **执行时机**: 用户创建或更新交易后自动执行
2. **锁定范围**: 所有被修改的属性（包括 merchant_id）
3. **审计日志**: ❌ 不通过 `enrich_attribute`，不记录 DataEnrichment
4. **优先级**: ⭐⭐⭐⭐⭐ 最高优先级，锁定后阻止所有后续自动识别

---

## 五、阶段 3: 规则引擎触发

### 5.1 规则动作类型

#### A. SetTransactionMerchant (规则直接指定商户)
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
1. **前置检查**: 商户必须存在于家庭商户列表中（FamilyMerchant）
2. **默认锁定检查**: 仅处理 `enrichable(:merchant_id)` 的交易
3. **强制覆盖模式**: `ignore_attribute_locks: true` 可绕过锁定机制
4. **锁定行为**: 仅设置值，不执行 `lock_attr!`

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
- **锁定检查**: 仅调度 `enrichable(:merchant_id)` 的交易，不可绕过
- **文件**: `app/jobs/auto_detect_merchants_job.rb:1-7`

---

## 六、阶段 4: AI 自动识别核心逻辑

### 6.1 入口与范围过滤 (`app/models/family/auto_merchant_detector.rb:1-98`)

```ruby
def scope
  family.transactions.where(id: transaction_ids, merchant_id: nil)
                     .enrichable(:merchant_id)
                     .includes(:merchant, :entry)
end
```

**前置过滤条件 (必须同时满足)**:
1. ✅ 在指定 `transaction_ids` 范围内
2. ✅ 当前 `merchant_id` 为 `nil` (未匹配)
3. ✅ `merchant_id` 属性未被锁定 (`enrichable`)

### 6.2 LLM API 调用 (OpenAI)
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

### 6.3 核心匹配与兜底分支

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

  # 🔴 关键: 无论成功与否，强制锁定属性防止重复识别
  transaction.lock_attr!(:merchant_id)
end
```

---

## 七、商户匹配优先级与生效条件总结

### 7.1 按执行时序与优先级排序

| 顺序 | 阶段 | 商户类型 | 来源 | 生效条件 | 锁定行为 |
|------|------|----------|------|----------|----------|
| 1 | Plaid 同步 | ProviderMerchant | plaid | Plaid 返回 merchant_entity_id + merchant_name | 不锁定 |
| 2 | 规则指定 | FamilyMerchant | rule | 商户存在 + (属性未锁定 OR ignore_attribute_locks) | 不锁定 |
| 3 | AI 识别匹配 | FamilyMerchant | ai | LLM 识别名称与用户已有商户精确匹配 | 识别后锁定 |
| 4 | AI 识别创建 | ProviderMerchant | ai | LLM 返回 business_name + business_url | 识别后锁定 |
| 5 | 用户编辑 | FamilyMerchant/ ProviderMerchant | (无) | 用户通过 API 提交 merchant_id | 编辑后强制锁定 |

> **注意**: 用户编辑虽在表格中排第 5，但实际可在任意时间点执行，且锁定后会阻止所有后续自动识别。

### 7.2 LLM 内部判定层级

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

### 7.3 属性锁定机制详解

**Enrichable 模块** (`app/models/concerns/enrichable.rb:1-91`)

```ruby
# 锁定单个属性，记录锁定时间
def lock_attr!(attr)
  update!(locked_attributes: locked_attributes.merge(attr.to_s => Time.current))
end

# 查询范围: 仅包含未锁定指定属性的记录
scope :enrichable, ->(attrs) {
  attrs = Array(attrs).map(&:to_s)
  where.not(Arel.sql("#{table_name}.locked_attributes ?| array[:keys]"), keys: attrs)
}
```

**锁定触发时机**:

| 触发点 | 锁定范围 | 说明 |
|--------|----------|------|
| AI 识别完成后 | `merchant_id` | 无论识别成功或失败，强制锁定 |
| 用户编辑交易后 | 所有被修改的属性 | 通过 `lock_saved_attributes!` |
| 规则 SetTransactionMerchant | ❌ 不锁定 | 仅设置值，不自动锁定 |
| Plaid 同步识别 | ❌ 不锁定 | 仅设置值，不自动锁定 |

---

## 八、兜底分支与异常处理汇总

### 8.1 各阶段兜底条件一览

| 阶段 | 兜底条件 | 处理方式 |
|------|----------|----------|
| **Plaid 同步** | 缺少 merchant_entity_id 或 merchant_name | 返回 nil，不设置 merchant_id |
| **规则 SetTransactionMerchant** | 商户 ID 不存在 | 静默跳过，不做处理 |
| **规则 SetTransactionMerchant** | 属性已锁定且未启用 ignore_attribute_locks | 跳过该交易 |
| **LLM 识别** | 置信度 < 80% | 返回 business_name: null, business_url: null |
| **LLM 识别** | 通用交易名称 (Paycheck, Grocery store 等) | 返回 null |
| **AI 商户匹配** | LLM 返回 null | 不设置 merchant_id，但强制锁定属性 |
| **AI 商户匹配** | 未匹配到用户商户但有 business_url | 创建 ProviderMerchant (source: ai) |

### 8.2 AI 识别失败锁定的影响与兜底

#### 🔴 关键行为：识别失败仍锁定属性

**代码位置**: `app/models/family/auto_merchant_detector.rb:62`

```ruby
# 无论成功与否，锁定属性防止重复识别
transaction.lock_attr!(:merchant_id)
```

#### 影响分析

| 影响项 | 说明 |
|--------|------|
| **不再重试** | 锁定后 `enrichable(:merchant_id)` 范围排除该交易，后续 AI 识别永不再重试 |
| **规则限制** | 普通规则执行（无 ignore_attribute_locks）无法修改该交易的 merchant_id |
| **Plai 同步覆盖** | Plaid 同步如在 AI 识别之后执行，也无法覆盖（因属性已锁定） |

#### 兜底方案

| 场景 | 兜底方式 |
|------|----------|
| 用户需要修改被锁定的商户 | ✅ 通过 API 手动编辑（用户编辑不受锁定限制） |
| 规则需要强制覆盖 | ✅ 启用 `ignore_attribute_locks: true` |
| 需要重新触发 AI 识别 | ⚠️ 需手动解锁：`transaction.unlock_attr!(:merchant_id)` |

---

## 九、DataEnrichment 审计日志

### 9.1 来源枚举（代码依据）
**文件**: `app/models/data_enrichment.rb:4`

```ruby
enum :source, { rule: "rule", plaid: "plaid", synth: "synth", ai: "ai" }
```

> **重要**: `user` 来源不存在于代码枚举中。用户手动编辑不通过 `enrich_attribute` 流程，因此不会记录 DataEnrichment 审计日志。

### 9.2 日志记录逻辑

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

**Source 实际来源**:
- `plaid` - Plaid API 自动识别
- `rule` - 规则引擎设置
- `ai` - OpenAI LLM 自动识别
- `synth` - 系统内置合成数据

---

## 十、关键约束与边界条件

### 10.1 批量限制
- OpenAI API 单次请求最大 25 笔交易 (`app/models/provider/openai.rb:31`)
- 规则调度按每批 20 笔异步执行

### 10.2 唯一性约束
- FamilyMerchant: `family_id + name` 唯一
- ProviderMerchant: `source + name` 唯一

### 10.3 幂等性保证
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
| `app/models/data_enrichment.rb` | 数据丰富审计日志模型 |
| `app/controllers/api/v1/transactions_controller.rb` | 交易编辑 API |
