# 交易自动分类决策逻辑报告

## 1. 系统架构概览

系统采用**规则引擎 + AI 智能分类**相结合的双层架构，实现交易的自动分类打标。核心组件包括：

- **规则引擎**：基于条件匹配的确定性分类
- **AI 分类器**：基于 GPT-4.1-mini 的智能分类
- **属性锁定机制**：控制属性是否可被自动修改

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

**规则分类的关键特性**：
- 规则使用 `enrich_attribute` 设置属性，但**不会自动锁定**
- 规则设置的分类**可能被后续规则覆盖**
- 来源记录为 "rule"

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

**AI 分类的关键特性**：
- AI 分类后**无论成功失败都会锁定**属性（app/models/family/auto_categorizer.rb:42）
- 锁定后**阻止所有后续自动处理**（包括其他规则和 AI）
- 来源记录为 "ai"

## 3. 优先级机制

### 3.1 核心原则：属性锁定（Enrichable 模块）

**锁定机制详解**（app/models/concerns/enrichable.rb:18-23）：

```ruby
scope :enrichable, ->(attrs) {
  where.not(Arel.sql("#{table_name}.locked_attributes ?| array[:keys]"), keys: attrs)
}
```

**属性锁定操作**：
- `lock_attr!(attr)`：锁定属性，记录锁定时间
- `unlock_attr!(attr)`：解锁属性
- `locked?(attr)`：检查是否已锁定

### 3.2 实际优先级体现

**优先级由锁定时机决定**：

| 来源 | 锁定时机 | 是否可被后续自动处理覆盖 |
|------|---------|-----------------------|
| **用户手动编辑** | 用户保存时锁定 | 否（最高优先级） |
| **AI 分类** | AI 处理后立即锁定（无论成功失败） | 否 |
| **规则分类** | 从不锁定 | 是（可被后续规则覆盖） |

**规则分类的行为**（app/models/rule/action_executor/set_transaction_category.rb:10-26）：
1. 检查属性是否可修改（ignore_attribute_locks 为 false 时）
2. 只对 `enrichable(:category_id)` 的交易执行
3. 设置分类，但**不锁定**
4. 记录来源为 "rule"

**AI 分类的行为**（app/models/family/auto_categorizer.rb:29-44）：
1. 只处理未分类且可修改的交易
2. 成功时设置分类（source: "ai"）
3. **无论成功失败，最后都会锁定**：`transaction.lock_attr!(:category_id)`
4. 锁定后，后续规则和 AI 都无法再自动修改

### 3.3 规则执行顺序

**全局执行触发**（app/models/family/syncer.rb:12-15）：
```ruby
family.rules.each do |rule|
  rule.apply_later
end
```

**关键注意：异步执行**
- `apply_later` 通过 `RuleJob` 异步执行（app/models/rule.rb:48-49）
- 任务队列执行顺序**不确定**，不保证按创建顺序执行
- 如果规则包含 `auto_categorize` 动作，会进一步触发 `AutoCategorizeJob`（app/models/family.rb:44-46）

**单规则内动作顺序**（app/models/rule.rb:42-46）：
```ruby
actions.each do |action|
  action.apply(matching_resources_scope, ignore_attribute_locks: ignore_attribute_locks)
end
```
动作按创建顺序依次执行（同步执行）。

### 3.4 优先级冲突场景分析

**场景 1：多个规则设置分类**
- 规则 A：将交易名称包含 "Starbucks" 的交易分类为 "餐饮"
- 规则 B：将金额 > $100 的交易分类为 "大额支出"
- 结果：最后执行的规则生效（因为规则不锁定）

**场景 2：规则 + AI 混合**
- 规则包含 `set_transaction_category` 和 `auto_categorize` 两个动作
- 执行顺序：先设置分类（不锁定），再触发 AI 分类（异步）
- 实际结果取决于任务队列执行顺序：
  - 如果规则先执行完成：AI 可能跳过（因为已分类）
  - 如果 AI 先执行：无论 AI 是否成功，都会锁定，规则设置会被跳过

**场景 3：AI 分类失败**
- AI 因置信度不足返回 null
- 交易保持 `category_id: nil`
- 但**属性已被锁定**
- 后续规则和 AI 都无法再自动处理该交易

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
  transaction.enrich_attribute(:category_id, category_id, source: "ai")
end
```

### 4.3 最终兜底：锁定即终止

**AI 处理后的无条件锁定**（app/models/family/auto_categorizer.rb:42）：
```ruby
scope.each do |transaction|
  # ... 尝试分类 ...
  
  transaction.lock_attr!(:category_id)  # 无条件锁定
end
```

**锁定的影响**：
- 交易保持 `category_id: nil`（如果分类失败）
- 但 `locked_attributes` 中记录了 `category_id` 的锁定时间
- **所有后续自动处理都会跳过**：
  - 规则的 `enrichable(:category_id)` 过滤会排除该交易
  - AI 分类的 `scope` 也会排除该交易
- **只有用户手动解锁**才能重新允许自动处理

### 4.4 兜底策略总结

| 情况 | 交易状态 | 属性锁定 | 后续自动处理 |
|------|---------|---------|------------|
| 规则分类成功 | 已分类 | 未锁定 | 可被其他规则覆盖 |
| AI 分类成功 | 已分类 | 已锁定 | 不可被覆盖 |
| AI 分类失败（置信度不足） | 未分类 | 已锁定 | 不可处理（终止） |
| AI 调用失败 | 未分类 | 未锁定 | 可重试 |

**核心结论**：
- **AI 是最终裁决者**：一旦交易被 AI 处理（无论成功失败），自动分类流程即终止
- **规则是可覆盖的**：规则之间可以相互覆盖，直到被 AI 锁定或用户锁定
- **失败即终止**：AI 失败不会"等待后续尝试"，而是通过锁定终止所有自动处理

## 5. 完整决策流程图

```
交易入库
    │
    ▼
Plaid 初始数据导入
    │
    ▼
同步触发规则执行（异步队列）
    │
    ├─► 规则 N（任务队列顺序不确定）
    │       │
    │       ├─► 条件不匹配 ──────► 跳过
    │       │
    │       └─► 条件匹配
    │               │
    │               ├─► 属性已锁定 ──► 跳过（AI 或用户已锁定）
    │               │
    │               └─► 属性可修改
    │                       │
    │                       ├─► 动作：set_transaction_category
    │                       │       │
    │                       │       ├─► 设置分类（source: "rule"）
    │                       │       │
    │                       │       └─► 不锁定 ──► 可被后续规则覆盖
    │                       │
    │                       └─► 动作：auto_categorize
    │                               │
    │                               └─► 触发 AI 分类任务（新的异步任务）
    │                                       │
    │                                       ▼
    │                                  AI 分类处理
    │                                       │
    │                                       ├─► 交易已分类 ──► 跳过（不锁定）
    │                                       │
    │                                       ├─► 属性已锁定 ──► 跳过
    │                                       │
    │                                       └─► 可处理交易
    │                                               │
    │                                               ├─► AI 调用失败 ──► 静默返回（不锁定，可重试）
    │                                               │
    │                                               └─► AI 返回结果
    │                                                       │
    │                                                       ├─► 找到匹配分类
    │                                                       │       │
    │                                                       │       └─► 设置分类（source: "ai"）
    │                                                       │
    │                                                       ├─► 未找到匹配分类（null 或不匹配）
    │                                                       │       │
    │                                                       │       └─► 保持未分类
    │                                                       │
    │                                                       └─► 无条件锁定属性
    │                                                               │
    │                                                               ▼
    │                                                         终止自动处理
    │
    └─► 其他规则...
            │
            └─► 重复上述流程（但已锁定的交易被跳过）
```

## 6. 关键设计决策

### 6.1 为什么规则不锁定而 AI 锁定？

**规则不锁定的原因**：
1. **规则可组合**：用户可能设置多个互补的规则，后设置的规则应该能覆盖前面的
2. **规则是确定性的**：规则逻辑清晰，用户容易理解和调整
3. **规则执行成本低**：规则是本地计算，重复执行成本可忽略

**AI 锁定的原因**：
1. **成本控制**：AI 调用有费用成本，避免重复调用
2. **结果稳定性**：AI 结果可能有随机性，锁定后保持一致
3. **失败即终止**：AI 无法确定的交易，不应让其他自动机制继续尝试
4. **用户介入点**：锁定后用户知道需要手动处理

### 6.2 为什么 AI 失败后也要锁定？

这是一个**有争议的设计决策**：

**可能的设计意图**：
- 避免浪费 API 调用在"无法分类"的交易上
- 让用户明确知道哪些交易需要手动处理
- 保持 AI 分类的"一次性"语义

**潜在问题**：
- 如果是暂时的原因（如商户信息缺失），锁定后无法自动重试
- 后续添加的规则也无法处理这些交易
- 用户可能不知道需要手动解锁

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
| 规则异步任务 | app/jobs/rule_job.rb | 1-7 |
| 属性锁定机制 | app/models/concerns/enrichable.rb | 18-73 |
| 设置分类执行器（不锁定） | app/models/rule/action_executor/set_transaction_category.rb | 10-26 |
| 设置标签执行器 | app/models/rule/action_executor/set_transaction_tags.rb | 10-26 |
| AI 分类触发 | app/models/rule/action_executor/auto_categorize.rb | 10-22 |
| AI 分类核心（无条件锁定） | app/models/family/auto_categorizer.rb | 9-44 |
| AI 异步任务 | app/models/family.rb | 44-46 |
| OpenAI 分类器 | app/models/provider/openai/auto_categorizer.rb | 8-119 |
| 交易资源注册表 | app/models/rule/registry/transaction_resource.rb | 1-34 |

## 8. 总结

### 8.1 优先级最终说明

**优先级层次（从高到低）**：

1. **用户手动编辑**（已锁定）
   - 用户保存时自动锁定
   - 任何自动处理都无法覆盖

2. **AI 处理过的交易**（已锁定）
   - 无论 AI 分类成功或失败
   - 锁定后终止所有自动处理

3. **规则设置的属性**（未锁定）
   - 可被后续规则覆盖
   - 直到被 AI 或用户锁定

4. **未处理的交易**（未锁定、未分类）
   - 可被任何规则或 AI 处理

### 8.2 兜底策略最终说明

**兜底策略的核心机制**：

1. **AI 置信度兜底**：<60% 置信度返回 null，宁缺毋滥
2. **分类类型兜底**：收入/支出类型必须匹配
3. **锁定兜底**：AI 处理后无条件锁定，终止自动处理链
4. **静默失败**：AI 调用失败不影响其他流程

**关键澄清**：
- ❌ ~~"分类失败后保持未分类并等待后续规则或 AI 再尝试"~~
- ✅ **"AI 分类失败后，交易保持未分类但属性被锁定，后续所有自动处理（规则和 AI）都无法再处理该交易"**

### 8.3 设计理念

系统的自动分类决策逻辑体现了以下核心设计理念：

1. **用户主权**：用户编辑始终具有最高优先级
2. **成本意识**：AI 调用昂贵，处理后立即锁定避免重复
3. **谨慎保守**：宁可未分类也不误分类
4. **可追溯性**：所有自动修改都有来源记录
5. **终止语义**：AI 是自动分类的"最后尝试"，无论成败都终止自动链

这种设计在自动化效率、成本控制和用户控制之间取得了平衡，但 AI 失败即锁定的策略需要用户理解并在必要时手动解锁。
