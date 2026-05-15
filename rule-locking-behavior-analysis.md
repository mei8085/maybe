# Rule 规则锁定行为深度分析报告

## 纠错声明

**上一轮分析偏差点**：此前分析认为 `ignore_attribute_locks=true` 会完全绕过锁定机制，但实际代码证据表明存在**双层拦截机制**，SQL 层过滤可绕过，但**模型层最终写入校验不可绕过**。

---

## 1. 完整执行链路追踪

### 1.1 调用链总览

```
  入口参数
     │
     ▼
  RuleJob.perform(rule, ignore_attribute_locks: true/false)
     │
     ▼
  rule.apply(ignore_attribute_locks: true/false)
     │
     ▼
  action.apply(scope, ignore_attribute_locks: true/false)
     │
     ▼
  SetTransactionCategory#execute(scope, value, ignore_attribute_locks: true/false)
     │
     ├─ 第一层：SQL 查询过滤（可被绕过）
     │
     └─ 循环执行 txn.enrich_attribute(:category_id, ...)
           │
           └─ 第二层：模型层 locked? 校验（不可绕过）
                 │
                 └─ ✅ 未锁定：写入 + 记录日志
                 └─ ❌ 已锁定：静默跳过，无数据库写入
```

---

## 2. 关键代码证据分层解析

### 2.1 第一层：SQL 查询过滤（可被 `ignore_attribute_locks` 绕过）

**文件**: `app/models/rule/action_executor/set_transaction_category.rb:10-17`

```ruby
def execute(transaction_scope, value: nil, ignore_attribute_locks: false)
  category = family.categories.find_by_id(value)

  scope = transaction_scope

  unless ignore_attribute_locks
    # 🔴 SQL 层过滤：仅查询 category_id 未被锁定的记录
    scope = scope.enrichable(:category_id)
  end

  scope.each do |txn|
    # ... 后续处理
  end
end
```

**Enrichable Scope 定义** (`app/models/concerns/enrichable.rb:18-22`):

```ruby
scope :enrichable, ->(attrs) {
  attrs = Array(attrs).map(&:to_s)
  json_condition = attrs.each_with_object({}) { |attr, hash| hash[attr] = true }
  # PostgreSQL JSONB 操作：检查 locked_attributes 不包含指定的 keys
  where.not(Arel.sql("#{table_name}.locked_attributes ?| array[:keys]"), keys: attrs)
}
```

**第一层过滤结论**:
| `ignore_attribute_locks` 值 | SQL 过滤行为 | 传递给循环的 scope |
|-----------------------------|--------------|-------------------|
| `false` (默认) | ✅ 应用 `enrichable(:category_id)` 过滤 | 仅包含 category_id 未锁定的交易 |
| `true` | ❌ 不应用 SQL 过滤 | 所有匹配规则条件的交易，**包括已锁定的** |

### 2.2 第二层：模型层 `locked?` 校验（**不可绕过**）

**文件**: `app/models/concerns/enrichable.rb:34-51`

```ruby
def enrich_attributes(attrs, source:, metadata: {})
  # 🟢 第二层校验：即使 SQL 层未过滤，这里仍会检查
  enrichable_attrs = Array(attrs).reject do |attr_key, attr_value|
    locked?(attr_key) ||                      # ← 这里仍会检查锁定状态！
    ignored_enrichable_attributes.include?(attr_key) ||
    self[attr_key.to_s] == attr_value
  end

  ActiveRecord::Base.transaction do
    enrichable_attrs.each do |attr, value|
      self.send("#{attr}=", value)
      # ... 记录 enrichment 日志
    end

    save
  end
end
```

**`locked?` 方法定义** (`app/models/concerns/enrichable.rb:53-55`):

```ruby
def locked?(attr)
  locked_attributes[attr.to_s].present?  # 检查 JSONB 字段中是否存在该属性
end
```

**关键发现**：
> ❗ **`ignore_attribute_locks` 参数仅影响 SQL 查询层，不影响模型层的最终校验。**
>
> 即使设置 `ignore_attribute_locks=true` 让已锁定交易进入循环，在 `enrich_attributes` 内部仍会通过 `locked?` 方法进行第二次校验，**已锁定的属性会被静默过滤，不会写入数据库**。

---

## 3. 参数值-分支-结果对照表

### 3.1 单交易维度行为矩阵

| 交易的 `category_id` 锁定状态 | `ignore_attribute_locks` 参数值 | SQL 层是否过滤此交易 | 是否进入 `scope.each` 循环 | `enrich_attributes` 是否接受 | 最终数据库结果 |
|-------------------------------|---------------------------------|---------------------|---------------------------|---------------------------|----------------|
| ❌ 未锁定 | `false` | 否，保留在 scope 中 | ✅ 是 | ✅ 是 | ✅ category_id 被更新 |
| ❌ 未锁定 | `true` | 否，保留在 scope 中 | ✅ 是 | ✅ 是 | ✅ category_id 被更新 |
| ✅ 已锁定 | `false` | ✅ 是，从 scope 中过滤 | ❌ 否 | ❌ 不执行 | ❌ 无变化 |
| ✅ 已锁定 | `true` | ❌ 否，保留在 scope 中 | ✅ 是 | ❌ 被 `locked?` 拦截 | ❌ 无变化 |

### 3.2 两种典型场景对比

#### 场景 A：`ignore_attribute_locks = false`（默认行为）

```
Rule 匹配 100 条交易
  │
  ├─ 其中 30 条 category_id 已被用户锁定
  └─ 70 条未锁定
       │
       ▼
  SQL 层 enrichable(:category_id) 过滤
       │
       ├─ 30 条锁定交易被过滤掉
       └─ 70 条进入循环
            │
            └─ 全部 70 条成功更新 category_id
```

#### 场景 B：`ignore_attribute_locks = true`

```
Rule 匹配 100 条交易
  │
  ├─ 其中 30 条 category_id 已被用户锁定
  └─ 70 条未锁定
       │
       ▼
  ❌ 无 SQL 层过滤
       │
       ▼
  全部 100 条进入循环
       │
       ├─ 70 条未锁定 → enrich_attribute 接受并更新
       └─ 30 条已锁定 → enrich_attribute 内部 reject，静默跳过
            │
            └─ 最终仍只有 70 条被更新
```

**重要结论**:
> **两种参数值最终更新的交易数量完全相同！**
>
> `ignore_attribute_locks=true` 仅仅是让已锁定交易"多走了一段路"进入循环，但在实际写入前仍会被拦截。
>
> 该参数的真实作用是**性能优化**：当确定需要批量处理大量数据时，避免 SQL 层执行额外的 JSONB 查询过滤，但**不改变最终锁定行为的语义**。

---

## 4. 规则跳过与更新条件的精确定义

### 4.1 规则跳过已锁定交易的条件

**条件 1：SQL 层过滤（`ignore_attribute_locks = false` 时）**
- 触发条件：`locked_attributes -> 'category_id'` 存在非空值
- 作用点：`scope.enrichable(:category_id)` 执行时
- 表现：交易不在 `scope.each` 枚举范围内，完全不进入执行循环

**条件 2：模型层校验（始终生效）**
- 触发条件：`txn.locked?(:category_id) == true`
- 作用点：`enrich_attributes` 方法内部 `reject` 过滤时
- 表现：交易进入循环，但属性被从 `enrichable_attrs` 中删除，不执行 `update`，不记录 enrichment 日志

### 4.2 规则继续更新未锁定交易的条件

**必须同时满足以下所有条件**:

| 条件层级 | 检查内容 | 代码位置 |
|---------|---------|---------|
| 1 | 交易匹配 Rule 的所有 conditions 过滤 | `rule.rb:65-79` |
| 2 | (可选) SQL 层未锁定，或 `ignore_attribute_locks=true` | `set_transaction_category.rb:15-17` |
| 3 | 模型层 `locked?(:category_id) == false` | `enrichable.rb:35-37` |
| 4 | 新值与旧值不相等 | `enrichable.rb:36` |
| 5 | 属性不在忽略列表中（id, created_at, updated_at） | `enrichable.rb:36, 88-90` |

---

## 5. 数据库状态变化追踪

### 5.1 执行前后数据对比

以 `category_id` 属性为例：

| 执行阶段 | `transactions.category_id` | `transactions.locked_attributes` | `data_enrichments` 记录 |
|---------|---------------------------|----------------------------------|-------------------------|
| 执行前 | `nil` | `{}` | 无 |
| Rule 执行（未锁定） | 被更新为规则指定值 | `{}` → 仍为空（规则不会自动锁定） | ✅ 新增一条 source='rule' 的记录 |
| 执行前 | `123` (用户设置) | `{"category_id": "2026-05-16T..."}` | 已有用户编辑记录 |
| Rule 执行（已锁定） | 保持 `123` 不变 | 保持锁定时间戳不变 | ❌ 无新增记录 |

### 5.2 关键观察

1. **规则不会自动锁定属性**：`enrich_attribute` 只更新数据，不调用 `lock_attr!`
2. **用户编辑才会触发锁定**：只有用户手动保存后，控制器才会调用 `lock_saved_attributes!`
3. **锁定是单向的时间戳**：一旦锁定，只有 `unlock_attr!` 可以解除，规则无法覆盖

---

## 6. 修正后的核心结论

### 6.1 对之前判断的纠正

| 之前的不准确判断 | 修正后的准确结论 | 证据代码位置 |
|-----------------|-----------------|-------------|
| `ignore_attribute_locks=true` 会绕过锁定 | ❌ 错误。仅绕过 SQL 层过滤，模型层校验仍生效，最终结果一致 | `enrichable.rb:35-37` |
| 规则可覆盖已锁定交易 | ❌ 错误。无论参数如何，已锁定交易都不会被更新 | 双层拦截机制 |
| 参数控制最终写入行为 | ❌ 错误。参数仅影响查询性能，不影响最终写入结果 | 对照表的 4 种组合 |

### 6.2 准确的设计意图

`ignore_attribute_locks` 参数的真实设计目标是：
- **性能优化**：避免在大数据量批量操作时执行额外的 JSONB SQL 过滤
- **语义不变**：用户锁定优先级高于规则的核心设计原则始终保持不变
- **容错处理**：即使 SQL 层未过滤，模型层仍有兜底保障，确保数据一致性

---

## 7. 附录：完整代码引用索引

| 逻辑点 | 文件路径 | 关键行号 |
|-------|---------|---------|
| RuleJob 入口参数传递 | `app/jobs/rule_job.rb` | 4-5 |
| Rule#apply 方法签名 | `app/models/rule.rb` | 42-46 |
| Action#apply 参数传递 | `app/models/rule/action.rb` | 6-8 |
| SetTransactionCategory SQL 过滤分支 | `app/models/rule/action_executor/set_transaction_category.rb` | 10, 15-17 |
| Enrichable scope 定义 | `app/models/concerns/enrichable.rb` | 18-22 |
| enrich_attributes 第二层锁定校验 | `app/models/concerns/enrichable.rb` | 34-37 |
| locked? 方法定义 | `app/models/concerns/enrichable.rb` | 53-55 |
