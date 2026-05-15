# 导入流程纠错复核 - 修订版分析

## 一、撤销回滚的精确数据清单（含判定依据）

### 1.1 回滚核心逻辑定位
**代码位置**：`app/models/import.rb:85-96`
```ruby
def revert
  Import.transaction do
    accounts.destroy_all
    entries.destroy_all
  end
  family.sync_later
  update! status: :pending
rescue => error
  update! status: :revert_failed, error: error.message
end
```

### 1.2 关联删除的判定依据
Import 模型的关联定义 `app/models/import.rb:39-42`：
```ruby
has_many :rows, dependent: :destroy        # 关联1：导入行数据
has_many :mappings, dependent: :destroy    # 关联2：字段映射
has_many :accounts, dependent: :destroy    # 关联3：创建的账户
has_many :entries, dependent: :destroy     # 关联4：创建的账目条目
```

**重要说明**：
- `accounts.destroy_all` 仅删除 `accounts.import_id = 当前导入.id` 的记录
- `entries.destroy_all` 仅删除 `entries.import_id = 当前导入.id` 的记录
- **不会**触发 Import 模型本身的 `dependent: :destroy`（因为 Import 没有被删除）

### 1.3 回滚时一定会删除的数据

| 数据表 | 删除条件 | 判定依据 | 级联删除内容 |
|-------|---------|---------|-------------|
| `accounts` | `import_id = 当前导入.id` | `import.accounts.destroy_all` | accountable(Depository等)、entries、holdings、balances |
| `entries` | `import_id = 当前导入.id` | `import.entries.destroy_all` | entryable(Transaction/Trade/Valuation) |
| `transactions` | 通过 entry 级联 | `Entry.delegated_type :entryable, dependent: :destroy` | taggings |
| `trades` | 通过 entry 级联 | 同上 | - |
| `valuations` | 通过 entry 级联 | 同上 | - |
| `taggings` | 通过 transaction 级联 | `Transaction.has_many :taggings, dependent: :destroy` | - |

### 1.4 回滚时一定不会删除的数据

| 数据表 | 不删除原因 | 备注 |
|-------|-----------|------|
| `imports` (本身) | revert 只更新 status，不删除 import | rows 和 mappings 会保留，可重新发布 |
| `import_rows` | revert 不触发 import.rows.destroy | 保留原始数据 |
| `import_mappings` | revert 不触发 import.mappings.destroy | 保留映射配置 |
| `categories` | find_or_create 创建，无 import_id 关联 | 全局资源，可被其他导入复用 |
| `tags` | 同上 | 全局资源 |
| `securities` | TradeImport 中 find_or_create | 全局资源，被 nullify 而非 destroy |
| **期初余额 Entry** | `OpeningBalanceManager` 创建时**未设置 import_id** | ⚠️ 会成为孤儿数据！见下文 |

### 1.5 条件删除数据（AccountMapping 场景）

**代码位置**：`app/models/import/account_mapping.rb:35-47`
```ruby
def create_mappable!
  account = import.family.accounts.create_or_find_by!(name: key) do |new_account|
    new_account.balance = 0
    new_account.import = import  # ⚠️ 仅在 create 时执行，find 时不执行
    new_account.currency = import.family.currency
    new_account.accountable = Depository.new
  end
end
```

| 场景 | import_id 设置 | 回滚时是否删除 |
|-----|---------------|---------------|
| 新建的账户 | ✅ 设置 | ✅ 会被删除 |
| 找到的已有账户 | ❌ 不设置 | ❌ 不会被删除 |

---

## 二、AccountImport 期初余额落盘链路的确定结论

### 2.1 完整落盘链路追踪

**代码位置**：`app/models/account_import.rb:4-29`
```ruby
def import!
  transaction do
    rows.each do |row|
      # 步骤1：创建 Account，设置 import_id
      account = family.accounts.build(
        name: row.name,
        balance: row.amount.to_d,
        currency: row.currency,
        accountable: accountable_class.new,
        import: self  # ✅ 这里设置了 import_id
      )
      account.save!

      # 步骤2：设置期初余额
      manager = Account::OpeningBalanceManager.new(account)
      result = manager.set_opening_balance(balance: row.amount.to_d)
    end
  end
end
```

**代码位置**：`app/models/account/opening_balance_manager.rb:65-75`
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
    # ❌ 这里缺少：import: account.import
  )
end
```

### 2.2 确定结论

| 问题 | 结论 | 证据 |
|-----|------|------|
| 期初余额 Entry 是否设置 import_id? | ❌ **没有设置** | `create_opening_anchor` 方法中没有传入 import 属性 |
| revert 时这条 Entry 是否会被删除? | ❌ **不会被删除** | `entries.destroy_all` 仅删除 `import_id = ?` 的条目 |
| Account 被删除时会怎样? | ✅ **级联删除** | `Account.has_many :entries, dependent: :destroy` |

### 2.3 两种回滚场景的不同结果

**场景A：AccountImport 创建的全新账户**
```
revert 执行 accounts.destroy_all
    ↓
  删除 Account (import_id 匹配)
    ↓ dependent: :destroy
  删除该 Account 的所有 entries（包括期初余额 Entry）
    ↓
  期初余额 Entry 被间接删除 ✅
```
**结果**：期初余额最终被间接删除，无残留

**场景B：Account 不是该导入创建的（极端场景）**
```
revert 执行 entries.destroy_all
    ↓
  仅删除 import_id 匹配的 entries
    ↓
  期初余额 Entry 无 import_id，**不会被删除** ❌
    ↓
  Account 也不会被删除（import_id 不匹配）
    ↓
  形成孤儿数据
```
**结果**：期初余额 Entry 残留，但 AccountImport 通常不会出现此场景

---

## 三、失败重试的风险边界（区分手动与自动）

### 3.1 重试入口与门禁矩阵

| 重试类型 | 调用入口 | 状态门禁 | publishable?检查 | 风险等级 |
|---------|---------|---------|-----------------|---------|
| **用户手动重试导入** | Controller#publish → `publish_later` | ✅ 完整检查 | ✅ 检查 | ✅ 安全 |
| **用户手动重试回滚** | Controller#revert → `revert_later` | ✅ 完整检查 | 不适用 | ✅ 安全 |
| **Sidekiq自动重试导入** | Job直接调用 `publish` | ❌ 仅检查行数 | ❌ 不检查 | ⚠️ 中高风险 |
| **Sidekiq自动重试回滚** | Job直接调用 `revert` | ❌ 无状态检查 | 不适用 | ✅ 低风险 |

### 3.2 用户手动重试的完整链路

**重试导入失败**：
```
用户点击"重新导入"
    ↓
ImportsController#publish
    ↓
@import.publish_later
    ├─ ✅ 检查 row_count_exceeded?
    ├─ ✅ 检查 publishable?
    │   └─ cleaned?（rows.all?(&:valid?)）
    │   └─ mappings.all?(&:valid?)
    ├─ ✅ 状态更新：failed → importing
    └─ 📥 ImportJob.perform_later
```
✅ **安全边界**：所有校验在入队前完成，Job排队期间数据变化可能导致 publishable? 失效。

**重试回滚失败**：
```
用户点击"重试撤销"
    ↓
ImportsController#revert
    ↓
@import.revert_later
    ├─ ✅ 检查 revertable?（complete? || revert_failed?）
    ├─ ✅ 状态更新：revert_failed → reverting
    └─ 📥 RevertImportJob.perform_later
```
✅ **安全边界**：状态门禁在入队前生效，防止非预期状态下执行回滚。

### 3.3 Sidekiq自动重试的风险边界

**当前配置**：`app/jobs/application_job.rb:2`
```ruby
retry_on ActiveRecord::Deadlocked
```
→ **仅死锁会自动重试**，其他异常（如校验失败、网络异常）不会自动重试

**自动重试导入的风险场景**：
```
1. publish_later 检查通过，status → importing，Job入队
2. Job执行到 import! 中途抛出 ActiveRecord::Deadlocked
3. Sidekiq触发自动重试，重新执行 publish
4. publish 仅检查 row_count_exceeded? ✅
5. ❌ 不检查 status == importing（可能已完成）
6. ❌ 不检查 publishable?（数据可能已变化）
7. 重复执行 import! → 重复入账
```

**自动重试回滚的风险场景**：
```
1. revert_later 检查通过，status → reverting，Job入队
2. Job执行到 accounts.destroy_all 中途死锁
3. Sidekiq触发自动重试，重新执行 revert
4. ❌ 不检查 status == reverting
5. 再次执行 destroy_all → 幂等操作，无副作用 ✅
```
✅ **结论**：destroy_all 是幂等的，回滚自动重试是安全的

### 3.4 极端风险场景

**场景1：恶意直接调用 publish**
```ruby
# 绕过 Controller，直接在 console 执行
import = Import.complete.first
import.publish  # ❌ 没有状态检查！重复执行
    ↓
重复创建 entries → 数据严重重复
```

**场景2：Job执行成功但状态更新失败**
```
1. import! 成功，所有数据已落盘
2. family.sync_later 成功
3. update!(status: :complete) 失败（如DB超时）
4. 状态仍为 importing
5. 用户/系统重试 → 重复入账
```

---

## 四、关键代码索引

| 功能 | 文件 | 行号 | 说明 |
|-----|------|-----|------|
| Import 关联定义 | `app/models/import.rb` | 39-42 | 4个 has_many |
| revert 方法 | `app/models/import.rb` | 85-96 | 事务内 destroy_all |
| publish 方法 | `app/models/import.rb` | 65-75 | 门禁缺失 |
| publish_later 方法 | `app/models/import.rb` | 56-63 | 门禁完整 |
| create_opening_anchor | `app/models/account/opening_balance_manager.rb` | 65-75 | 缺少 import_id |
| AccountMapping#create_mappable! | `app/models/import/account_mapping.rb` | 35-47 | create_or_find_by 行为 |
| Entry delegated_type | `app/models/entry.rb` | 10 | dependent: :destroy |
| Job死锁重试 | `app/jobs/application_job.rb` | 2 | 仅重试 Deadlocked |

---

## 五、修正建议（精确可执行）

### 5.1 补全 publish 状态门禁（高优先级）
```ruby
def publish
  # 幂等保护：已完成则直接返回
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

### 5.2 补全 revert 状态门禁（中优先级）
```ruby
def revert
  # 状态合法性保护
  raise "Import must be in 'reverting' status" unless reverting?

  Import.transaction do
    accounts.destroy_all
    entries.destroy_all
  end
  family.sync_later
  update! status: :pending
rescue => error
  update! status: :revert_failed, error: error.message
end
```

### 5.3 修复期初余额 Entry import_id 缺失（高优先级）
```ruby
# app/models/account/opening_balance_manager.rb:65-75
def create_opening_anchor(balance:, date:)
  account.entries.create!(
    date: date,
    name: Valuation.build_opening_anchor_name(account.accountable_type),
    amount: balance,
    currency: account.currency,
    import: account.import,  # ✅ 新增：关联导入
    entryable: Valuation.new(
      kind: "opening_anchor"
    )
  )
end
```

### 5.4 扩展重试配置（可选）
```ruby
# app/jobs/import_job.rb
class ImportJob < ApplicationJob
  queue_as :high_priority
  
  # 限制重试次数，避免死循环
  sidekiq_options retry: 3

  def perform(import)
    import.publish
  end
end
```
