# 交易自动分类决策逻辑报告

## 1. 系统架构概览

系统采用**规则引擎 + AI 智能分类**相结合的双层架构，实现交易的自动分类打标。核心组件包括：

- **规则引擎**：基于条件匹配的确定性分类
- **AI 分类器**：基于 GPT-4.1-mini 的智能分类
- **属性锁定机制**：确保用户编辑的优先级和数据一致性

## 2. 判定规则

### 2.1 规则驱动分类（确定性）

**条件过滤类型**：
- 交易名称匹配（TransactionName）
- 交易金额范围（TransactionAmount）
- 商户匹配（TransactionMerchant）

**规则执行流程**（app/models/rule.rb:42-46）：
1. 准备查询：应用条件所需的表连接
2. 应用条件：过滤出匹配的交易
3. 执行动作：对匹配交易应用规则动作

**可用动作类型**（app/models/rule/registry/transaction_resource.rb:14-28）：
- `set_transaction_category`：设置分类
- `set_transaction_tags`：设置标签
- `set_transaction_merchant`：设置商户
- `set_transaction_name`：设置名称
- `auto_categorize`：AI 自动分类（需启用 OpenAI）
- `auto_detect_merchants`：AI 商户识别（需启用 OpenAI）

### 2.2 AI 智能分类（概率性）

**触发条件**（app/models/family/auto_categorizer.rb:78-82）：
- 交易尚未分类（category_id: nil）
- 分类属性未被锁定
- 包含必要关联数据（category, merchant, entry）

**AI 输入数据**（app/models/family/auto_categorizer.rb:66-76）：
```ruby
{
  id: transaction.id,
  amount: transaction.entry.amount.abs,
  classification: transaction.entry.classification,  # 收入/支出
  description: transaction.entry.name,
  merchant: transaction.merchant&.name
}
```

**用户分类数据**（app/models/family/auto_categorizer.rb:54-64）：
- 分类 ID、名称
- 是否为子分类
- 父分类 ID
- 分类类型（收入/支出）

**AI 判定规则**（app/models/provider/openai/auto_categorizer.rb:100-119）：
1. 为每个交易返回一个结果
2. 通过 transaction_id 关联
3. 优先匹配最具体的分类（子分类 > 父分类）
4. 分类类型必须匹配（收入/支出一致）
5. 不确定时返回 null（置信度 < 60%）
6. 宁可返回 null 也不产生误判

**技术限制**（app/models/provider/openai.rb:17-27）：
- 每批最多 25 笔交易
- 使用 GPT-4.1-mini 模型
- 严格的 JSON Schema 输出约束

## 3. 优先级机制

### 3.1 核心原则：属性锁定（Enrichable 模块）

**优先级顺序**（从高到低）：
1. **用户手动编辑** - 最高优先级
2. **先执行的规则** - 时间顺序优先
3. **AI 自动分类** - 最后尝试

### 3.2 锁定机制详解（app/models/concerns/enrichable.rb）

**锁定状态检查**：
```ruby
scope :enrichable, ->(attrs) {
  where.not(Arel.sql("#{table_name}.locked_attributes ?| array[:keys]"), keys: attrs)
}
```

**属性锁定操作**：
- `lock_attr!(attr)`：锁定属性，记录锁定时间
- `unlock_attr!(attr)`：解锁属性
- `locked?(attr)`：检查是否已锁定

### 3.3 规则执行顺序

**全局执行顺序**（app/models/family/syncer.rb:12-15）：
```ruby
family.rules.each do |rule|
  rule.apply_later
end
```

规则按数据库默认顺序执行（通常为创建顺序），没有显式的优先级字段。

**单规则内动作顺序**（app/models/rule.rb:42-46）：
```ruby
actions.each do |action|
  action.apply(matching_resources_scope, ignore_attribute_locks: ignore_attribute_locks)
end
```

动作按创建顺序依次执行。

### 3.4 实际优先级体现

**规则分类的优先级**（app/models/rule/action_executor/set_transaction_category.rb:10-26）：
1. 检查属性是否可修改（ignore_attribute_locks 为 false 时）
2. 只对 `enrichable(:category_id)` 的交易执行
3. 设置分类时记录来源为 "rule"

**AI 分类的优先级**（app/models/family/auto_categorizer.rb:29-44）：
1. 只处理未分类且可修改的交易
2. 成功分类后立即锁定：`transaction.lock_attr!(:category_id)`
3. 来源记录为 "ai"

## 4. 兜底策略

### 4.1 AI 分类兜底

**置信度兜底**（app/models/provider/openai/auto_categorizer.rb:112-114）：
```
- If you don't know the category, return "null"
  - You should always favor "null" over false positives
  - Be slightly pessimistic. Only match a category if you're 60%+ confident
```

**分类类型兜底**（app/models/provider/openai/auto_categorizer.rb:111）：
```
- Category and transaction classifications should match
  (i.e. if transaction is an "expense", the category must have classification of "expense")
```

### 4.2 处理失败情况

**AI 调用失败**（app/models/family/auto_categorizer.rb:24-27）：
```ruby
unless result.success?
  Rails.logger.error("Failed to auto-categorize...")
  return  # 静默失败，不影响其他流程
end
```

**分类匹配失败**（app/models/family/auto_categorizer.rb:32-40）：
```ruby
category_id = user_categories_input.find { |c| 
  c[:name] == auto_categorization&.category_name 
}&.dig(:id)

if category_id.present?
  # 只有找到匹配分类才设置
  transaction.enrich_attribute(:category_id, category_id, source: "ai")
end
```

### 4.3 最终兜底

**保持未分类状态**：
- 任何分类失败都不会强制设置默认分类
- 交易保持 `category_id: nil` 状态
- 等待用户手动分类或后续规则/AI 尝试

## 5. 完整决策流程图

```
交易入库
    │
    ▼
Plaid 初始数据导入
    │
    ▼
同步触发规则执行
    │
    ├─► 规则 1（条件匹配）
    │       │
    │       ├─► 条件不匹配 ──────► 跳过
    │       │
    │       └─► 条件匹配
    │               │
    │               ├─► 属性已锁定 ──► 跳过（高优先级来源已设置）
    │               │
    │               └─► 属性可修改
    │                       │
    │                       └─► 执行动作（设置分类/标签等）
    │                               │
    │                               └─► 记录来源（rule）
    │
    ├─► 规则 2（同上）
    │
    └─► ...
    │
    ▼
AI 自动分类（如有规则触发）
    │
    ├─► 交易已分类 ──────► 跳过
    │
    ├─► 属性已锁定 ──────► 跳过
    │
    └─► 可分类交易
            │
            ├─► AI 调用失败 ──► 静默跳过
            │
            ├─► AI 返回 null ─► 保持未分类
            │
            ├─► 分类不匹配 ───► 保持未分类
            │
            └─► 分类成功
                    │
                    ├─► 设置分类（source: "ai"）
                    │
                    └─► 锁定属性
                            │
                            ▼
                      等待用户确认或保持
```

## 6. 关键设计决策

### 6.1 为什么使用属性锁定而非规则优先级？

**优势**：
1. **用户优先**：用户手动编辑始终不可被覆盖
2. **时间顺序**：先到先得，逻辑简单直观
3. **来源追踪**：每个修改都有来源记录（rule/ai/user/plaid）
4. **可追溯**：DataEnrichment 记录每次修改历史

**劣势**：
- 规则执行顺序影响最终结果
- 无法通过调整优先级来控制冲突

### 6.2 为什么 AI 分类后立即锁定？

1. **避免重复处理**：同一交易不会被多次 AI 分类
2. **节省成本**：减少不必要的 API 调用
3. **用户控制**：用户可手动解锁后重新分类

### 6.3 为什么宁可 null 也不误判？

1. **用户体验**：错误分类比未分类更令人沮丧
2. **信任建立**：用户信任系统的准确性
3. **纠错成本**：修正错误分类的成本高于手动分类

## 7. 代码位置索引

| 功能模块 | 文件路径 | 核心行号 |
|---------|---------|---------|
| 规则执行入口 | app/models/rule.rb | 42-46 |
| 规则条件过滤 | app/models/rule.rb | 65-79 |
| 规则同步触发 | app/models/family/syncer.rb | 12-15 |
| 属性锁定机制 | app/models/concerns/enrichable.rb | 18-73 |
| 设置分类执行器 | app/models/rule/action_executor/set_transaction_category.rb | 10-26 |
| 设置标签执行器 | app/models/rule/action_executor/set_transaction_tags.rb | 10-26 |
| AI 分类触发 | app/models/rule/action_executor/auto_categorize.rb | 10-22 |
| AI 分类核心 | app/models/family/auto_categorizer.rb | 9-44 |
| OpenAI 分类器 | app/models/provider/openai/auto_categorizer.rb | 8-119 |
| 交易资源注册表 | app/models/rule/registry/transaction_resource.rb | 1-34 |

## 8. 总结

系统的自动分类决策逻辑体现了以下核心设计理念：

1. **确定性优先**：规则驱动的分类先于 AI 分类
2. **用户主权**：用户编辑始终具有最高优先级
3. **谨慎保守**：宁可未分类也不误分类
4. **可追溯性**：所有自动修改都有来源记录
5. **成本控制**：AI 分类后立即锁定，避免重复调用

这种设计在自动化和用户控制之间取得了良好平衡，既减少了用户的手动操作负担，又保留了用户对数据的最终控制权。
