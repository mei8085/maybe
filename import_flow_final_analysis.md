# 导入流程最终复核分析（证据驱动版）

---

## 一、映射数据在撤销后的精确边界（100% 代码证据）

### 1.1 核心机制：dependent: :destroy 的触发条件

**代码定义**（`app/models/import.rb:39-42`）
```ruby
has_many :rows, dependent: :destroy        # 关联1：导入行数据
has_many :mappings, dependent: :destroy    # 关联2：字段映射
has_many :accounts, dependent: :destroy    # 关联3：创建的账户
has_many :entries, dependent: :destroy     # 关联4：创建的账目条目
```

**关键理解**：
> `dependent: :destroy` **仅在父对象（Import）本身被 destroy 时触发级联删除**。
> 对关联集合调用 `destroy_all` **不会**触发其他关联的级联删除。

---

### 1.2 revert 方法的精确执行追踪

**代码位置**（`app/models/import.rb:85-96`）
```ruby
def revert
  Import.transaction do
    accounts.destroy_all   # 仅删除 accounts 关联
    entries.destroy_all    # 仅删除 entries 关联
    # ❌ 注意：没有调用 rows.destroy_all
    # ❌ 注意：没有调用 mappings.destroy_all
    # ❌ 注意：没有调用 self.destroy
  end
  family.sync_later
  update! status: :pending  # ✅ 仅更新状态，Import 本身保留
end
```

---

### 1.3 映射数据的精确删除边界

| 数据类型 | revert 时是否删除？ | 判定依据 |
|---------|-------------------|---------|
| **import_rows（原始导入行数据）** | ❌ **不会删除** | revert 没有调用 `rows.destroy_all` |
| **import_mappings（字段映射配置）** | ❌ **不会删除** | revert 没有调用 `mappings.destroy_all` |
| **accounts（创建的账户）** | ✅ **会删除** | revert 显式调用 `accounts.destroy_all` |
| **entries（创建的账目条目）** | ✅ **会删除** | revert 显式调用 `entries.destroy_all` |
| **级联的 entryable**<br>(Transaction/Trade/Valuation) | ✅ **会删除** | `Entry.delegated_type :entryable, dependent: :destroy` |
| **级联的 taggings** | ✅ **会删除** | `Transaction.has_many :taggings, dependent: :destroy` |

---

### 1.4 反向级联：映射指向的外部资源删除边界

**代码证据**（`app/models/category.rb:3`、`tag.rb:5`、`account.rb:9`）
```ruby
# Category 模型
has_many :import_mappings, as: :mappable, dependent: :destroy, class_name: "Import::Mapping"

# Tag 模型  
has_many :import_mappings, as: :mappable, dependent: :destroy, class_name: "Import::Mapping"

# Account 模型
has_many :import_mappings, as: :mappable, dependent: :destroy, class_name: "Import::Mapping"
```

**方向说明**：
> 这个 `dependent: :destroy` 是**反向依赖**：当 Category/Tag/Account 被删除时，会删除指向它的 import_mapping 记录。
> **这与 Import revert 无关**，方向是相反的！

| 场景 | 映射记录（import_mappings）是否删除？ | 说明 |
|-----|-------------------------------------|------|
| 执行 Import.revert | ❌ 不删除 | revert 不操作 mappings 集合 |
| 执行 Import.destroy | ✅ 删除 | Import 级联删除所有 has_many 关联 |
| 删除某个 Category | ✅ 删除该 Category 的所有 mapping | 反向级联生效 |

---

### 1.5 设计意图推断（基于代码行为）

**为什么 revert 不删除 rows 和 mappings？**
- 用户撤销导入后，可能想调整配置后重新导入
- 保留原始数据（rows）和映射配置（mappings）可以节省用户重复工作
- 用户如果真的想完全删除，有单独的 `destroy` 接口（`ImportsController#destroy`）

**这是设计特性，不是 Bug** ✅

---

## 二、队列自动重试触发范围（证据驱动表述）

### 2.1 已证实的代码配置

**证据1：ApplicationJob 全局配置**（`app/jobs/application_job.rb:1-5`）
```ruby
class ApplicationJob < ActiveJob::Base
  retry_on ActiveRecord::Deadlocked    # ✅ 显式配置：死锁会重试
  discard_on ActiveJob::DeserializationError  # ✅ 显式配置：反序列化错误直接丢弃
  queue_as :low_priority
end
```

**证据2：ImportJob 无额外配置**（`app/jobs/import_job.rb:1-7`）
```ruby
class ImportJob < ApplicationJob
  queue_as :high_priority
  # ❌ 无额外 retry_on 配置
  def perform(import)
    import.publish
  end
end
```

**证据3：RevertImportJob 无额外配置**（`app/jobs/revert_import_job.rb:1-7`）
```ruby
class RevertImportJob < ApplicationJob
  queue_as :medium_priority
  # ❌ 无额外 retry_on 配置
  def perform(import)
    import.revert
  end
end
```

---

### 2.2 已证实会触发自动重试的情况

| 异常类型 | 是否自动重试？ | 证据 | 重试次数 |
|---------|--------------|------|---------|
| `ActiveRecord::Deadlocked` | ✅ 会重试 | `retry_on ActiveRecord::Deadlocked` | ActiveJob 默认20次 |

---

### 2.3 已证实不会触发自动重试的情况

| 异常类型 | 是否自动重试？ | 证据 |
|---------|--------------|------|
| `ActiveJob::DeserializationError` | ❌ 不会重试 | `discard_on ActiveJob::DeserializationError` |

---

### 2.4 不能下绝对结论的情况（Sidekiq 行为边界）

> ⚠️ **重要说明**：以下内容基于 Rails ActiveJob 和 Sidekiq 的通用机制推断，**非本代码库的显式配置**，不能作为100%确定结论。

#### 推断A：Sidekiq 默认重试行为是否生效？
```
已知事实：
1. 代码库使用 Sidekiq（config/sidekiq.yml、config/initializers/sidekiq.rb）
2. Sidekiq 默认对所有未捕获异常重试25次
3. ActiveJob 包装 Sidekiq 时可能改变默认行为

不确定点：
- ActiveJob 的 retry_on 是否会覆盖 Sidekiq 默认重试？
- 如果设置了 retry_on X，其他异常是重试还是不重试？

保守结论：
- 死锁（Deadlocked）：确定重试 ✅
- 其他所有异常：行为不确定 ⚠️，需要实际测试确认
```

#### 推断B：重试时的状态变化？
```
如果发生自动重试：
- status 可能仍停留在 "importing" / "reverting"
- publish/revert 方法中没有幂等检查（如 return if complete?）
- 可能导致重复执行

但这是推论，不是已证实的代码行为 ❗
```

---

### 2.5 重试风险矩阵（证据分级标注）

| 重试场景 | 触发可能性 | 风险等级 | 证据级别 |
|---------|-----------|---------|---------|
| 导入时死锁自动重试 | 高 | ⚠️ 中高 | ✅ 已证实（retry_on 配置） |
| 回滚时死锁自动重试 | 高 | ✅ 低风险 | ✅ 已证实（destroy_all 幂等） |
| 导入时其他异常自动重试 | 不确定 | 未知风险 | ⚠️ 需测试确认 |
| 回滚时其他异常自动重试 | 不确定 | 未知风险 | ⚠️ 需测试确认 |

---

## 三、期初余额落盘链路（重新确认）

### 3.1 已证实的代码行为

**证据**（`app/models/account/opening_balance_manager.rb:65-75`）
```ruby
def create_opening_anchor(balance:, date:)
  account.entries.create!(
    date: date,
    name: Valuation.build_opening_anchor_name(account.accountable_type),
    amount: balance,
    currency: account.currency,
    entryable: Valuation.new(
      kind: "opening_anchor"
    )
    # ❌ 已证实：确实没有设置 import: account.import
  )
end
```

### 3.2 两种回滚场景的精确结果

| 场景 | 期初余额 Entry 最终状态 | 判定依据 |
|-----|-----------------------|---------|
| Account 是该导入创建的<br>（account.import_id = import.id） | ✅ 会被删除 | `import.accounts.destroy_all` 删除 Account<br>→ Account 级联删除其所有 entries<br>→ 期初余额 Entry 被间接删除 |
| Account 不是该导入创建的<br>（极端场景：account.import_id 为空或其他） | ❌ 不会被删除 | Account 不会被 `accounts.destroy_all` 删除<br>→ 期初余额 Entry 也不会被删除<br>→ 且期初余额 Entry 自身 import_id 为空<br>→ 不会被 `entries.destroy_all` 删除<br>→ 形成孤儿数据 |

---

## 四、关键代码索引（精确可复核）

| 功能 | 文件 | 行号 | 可复核点 |
|-----|------|-----|---------|
| Import 关联定义 | `app/models/import.rb` | 39-42 | 4个 has_many，均有 dependent: :destroy |
| revert 方法 | `app/models/import.rb` | 85-96 | 仅调用 accounts.destroy_all + entries.destroy_all |
| create_opening_anchor | `app/models/account/opening_balance_manager.rb` | 65-75 | 确认无 import 设置 |
| ApplicationJob 重试配置 | `app/jobs/application_job.rb` | 1-5 | retry_on Deadlocked, discard_on DeserializationError |
| ImportJob 定义 | `app/jobs/import_job.rb` | 1-7 | 无额外重试配置 |
| RevertImportJob 定义 | `app/jobs/revert_import_job.rb` | 1-7 | 无额外重试配置 |

---

## 五、修正建议（风险分级）

### 5.1 高优先级：publish 幂等保护
```ruby
def publish
  # 幂等保护：防止重复执行
  return if complete?
  # 状态合法性保护
  raise "Import must be in 'importing' status" unless importing?
  
  raise MaxRowCountExceededError if row_count_exceeded?

  import!
  family.sync_later
  update! status: :complete
rescue => error
  update! status: :failed, error: error.message
end
```

### 5.2 中优先级：期初余额 import_id 修复
```ruby
def create_opening_anchor(balance:, date:)
  account.entries.create!(
    date: date,
    name: Valuation.build_opening_anchor_name(account.accountable_type),
    amount: balance,
    currency: account.currency,
    import: account.import,  # ✅ 新增：关联导入
    entryable: Valuation.new(kind: "opening_anchor")
  )
end
```

### 5.3 可选：明确所有异常的重试策略
```ruby
class ApplicationJob < ActiveJob::Base
  retry_on ActiveRecord::Deadlocked
  discard_on ActiveJob::DeserializationError
  
  # 显式设置：其他异常最多重试3次，避免无限重试
  sidekiq_options retry: 3
  
  queue_as :low_priority
end
```

---

## 六、复核说明

| 复核项 | 结论类型 | 置信度 |
|-------|---------|-------|
| 映射数据撤销后是否保留 | 已证实（代码证据） | 100% |
| 死锁自动重试 | 已证实（retry_on 配置） | 100% |
| 其他异常自动重试 | 未证实（需测试） | < 50% |
| 期初余额 import_id 缺失 | 已证实（代码证据） | 100% |
