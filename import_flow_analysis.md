# 导入流程事务边界与回滚机制分析

## 一、核心状态机设计

### 1.1 状态定义 (`app/models/import.rb:24-31`)

```ruby
enum :status, {
  pending: "pending",        # 待处理/配置中
  complete: "complete",      # 导入完成
  importing: "importing",    # 导入执行中
  reverting: "reverting",    # 回滚执行中
  revert_failed: "revert_failed", # 回滚失败
  failed: "failed"           # 导入失败
}, validate: true, default: "pending"
```

### 1.2 状态流转图

```
pending → importing → complete → reverting → pending
   ↓          ↓          ↓            ↓
 failed      failed   revert_failed  revert_failed
```

### 1.3 关键状态校验方法

```ruby
# 可发布条件：配置完成 + 所有行有效 + 所有映射有效
def publishable?
  cleaned? && mappings.all?(&:valid?)
end

# 可回滚条件：已完成 或 回滚失败（可重试）
def revertable?
  complete? || revert_failed?
end
```

---

## 二、用户确认流程（Publish）

### 2.1 调度顺序

```
用户点击确认
    ↓
1. ImportsController#publish
    ↓
2. Import#publish_later 【非事务】
   ├─ 状态校验：publishable?
   ├─ 行数校验：row_count_exceeded?
   ├─ 状态更新：status → :importing
   └─ 入队：ImportJob.perform_later(self) 【高优先级队列】
    ↓
3. 后台 Job 执行：ImportJob#perform
    ↓
4. Import#publish
   ├─ 调用 import!（子类实现）【事务内】
   ├─ family.sync_later       【异步】
   └─ 状态更新：status → :complete
    ↓
5. 异常处理：status → :failed，记录 error 信息
```

### 2.2 事务边界分析

#### TransactionImport (`app/models/transaction_import.rb:2-33`)
```ruby
def import!
  transaction do
    # 步骤1：创建所有映射资源（Category、Tag、Account）
    mappings.each(&:create_mappable!)
    
    # 步骤2：构建所有 Transaction + Entry
    transactions = rows.map do |row|
      Transaction.new(
        category: mappings.categories.mappable_for(row.category),
        tags: row.tags_list.map { |tag| mappings.tags.mappable_for(tag) }.compact,
        entry: Entry.new(
          account: mapped_account,
          date: row.date_iso,
          amount: row.signed_amount,
          name: row.name,
          currency: row.currency,
          notes: row.notes,
          import: self  # 关键关联！
        )
      )
    end
    
    # 步骤3：批量导入（级联创建 Entry）
    Transaction.import!(transactions, recursive: true)
  end
end
```

#### AccountImport (`app/models/account_import.rb:4-29`)
```ruby
def import!
  transaction do
    rows.each do |row|
      # 步骤1：创建 Account，关联 import_id
      account = family.accounts.build(
        name: row.name,
        balance: row.amount.to_d,
        currency: row.currency,
        accountable: accountable_class.new,
        import: self  # 关键关联！
      )
      account.save!
      
      # 步骤2：设置期初余额（创建 Entry）
      manager = Account::OpeningBalanceManager.new(account)
      result = manager.set_opening_balance(balance: row.amount.to_d)
    end
  end
end
```

#### TradeImport (`app/models/trade_import.rb:2-37`)
```ruby
def import!
  transaction do
    # 步骤1：创建 Account 映射
    mappings.each(&:create_mappable!)
    
    # 步骤2：构建所有 Trade + Entry
    trades = rows.map do |row|
      Trade.new(
        security: find_or_create_security(...),
        qty: row.qty,
        price: row.price,
        entry: Entry.new(
          account: mapped_account,
          date: row.date_iso,
          amount: row.signed_amount,
          import: self  # 关键关联！
        )
      )
    end
    
    # 步骤3：批量导入
    Trade.import!(trades, recursive: true)
  end
end
```

### 2.3 关键落盘点

| 数据类型 | 表名 | 关联字段 | 删除策略 |
|---------|------|---------|---------|
| 导入临时行 | import_rows | import_id | dependent: :destroy |
| 字段映射 | import_mappings | import_id | dependent: :destroy |
| 账户 | accounts | import_id | dependent: :destroy |
| 账目条目 | entries | import_id | dependent: :destroy |
| 交易实体 | transactions | (通过 entry 关联) | dependent: :destroy |
| 交易实体 | trades | (通过 entry 关联) | dependent: :destroy |

---

## 三、用户撤销流程（Revert）

### 3.1 调度顺序

```
用户点击撤销
    ↓
1. ImportsController#revert
    ↓
2. Import#revert_later 【非事务】
   ├─ 状态校验：revertable?
   ├─ 状态更新：status → :reverting
   └─ 入队：RevertImportJob.perform_later(self) 【中优先级队列】
    ↓
3. 后台 Job 执行：RevertImportJob#perform
    ↓
4. Import#revert
   ├─ Import.transaction do
   │   ├─ accounts.destroy_all 【级联删除 accounts.import_id = ?】
   │   └─ entries.destroy_all  【级联删除 entries.import_id = ?】
   ├─ family.sync_later        【异步】
   └─ 状态更新：status → :pending
    ↓
5. 异常处理：status → :revert_failed，记录 error 信息
```

### 3.2 事务边界分析

```ruby
def revert
  Import.transaction do
    # 删除该导入创建的所有账户
    accounts.destroy_all
    # 删除该导入创建的所有账目条目（级联删除 transactions/trades）
    entries.destroy_all
  end

  family.sync_later

  update! status: :pending
rescue => error
  update! status: :revert_failed, error: error.message
end
```

**注意**：
- `accounts.destroy_all` 和 `entries.destroy_all` 在同一个事务中
- 删除是通过关联关系：`import.accounts` 和 `import.entries`
- 这意味着只有通过 `import_id` 关联的记录才会被删除

### 3.3 级联删除链

**当 accounts.destroy_all 被调用时：**
```
Account (import_id = ?)
    ↓ dependent: :destroy
    ├─ entries → Entry → [Transaction/Trade/Valuation]
    ├─ holdings
    ├─ balances
    └─ accountable (Deposit/CreditCard 等)
```

**当 entries.destroy_all 被调用时：**
```
Entry (import_id = ?)
    ↓ dependent: :destroy
    └─ entryable (Transaction / Trade / Valuation)
           ↓
           ├─ Transaction → taggings
           └─ Trade
```

---

## 四、幂等性与重试机制

### 4.1 防止重复入账

1. **状态前置校验**
   - `publish_later` 只允许 `pending` 状态（隐式通过 `publishable?`）
   - `revert_later` 只允许 `complete` 或 `revert_failed` 状态

2. **事务原子性**
   - 整个 `import!` 在数据库事务中执行
   - 任何一步失败，全部回滚

3. **关联删除的精确性**
   - 只删除 `import_id = ?` 的记录
   - 不会影响其他导入或手动创建的数据

### 4.2 失败重试场景

#### 场景1：导入失败 (status = failed)
- 可修复配置问题后重新点击确认
- 重新执行 `publish_later`
- `import!` 会从头开始，事务保证不重复

#### 场景2：回滚失败 (status = revert_failed)
- 可直接点击重试撤销
- `revert_later` 允许 `revert_failed` 状态
- 再次执行 `accounts.destroy_all` 和 `entries.destroy_all` 是幂等的

#### 场景3：Job 中途失败（如 Sidekiq 重启）
- **导入中**：`importing` 状态卡死 → 需要人工干预或添加超时机制
  - 建议添加定时任务检测长时间 `importing` 状态
- **回滚中**：`reverting` 状态卡死 → 同上
  - `revert_failed` 状态可重试

### 4.3 潜在风险点

**风险1：Mappings 创建的资源不会被回滚**
```ruby
# TransactionImport#import! 中：
mappings.each(&:create_mappable!)  # 创建 Category、Tag、Account
```
- 这些创建的 Category、Tag 不会在 revert 时被删除
- 因为 `import_mappings.mappable_id` 指向它们，但 revert 只删除 `accounts` 和 `entries`

**风险2：AccountImport 的期初余额 Entry**
- `OpeningBalanceManager` 创建的 Entry 是否设置了 `import_id`？
- 如果没有设置，revert 时无法删除，会产生孤儿数据

**风险3：TradeImport 的 Security 创建**
```ruby
security = Security::Resolver.new(...).resolve
```
- Security 可能是 find_or_create 的
- revert 时不会删除 Security（设计上合理，因为可能被其他交易使用）

---

## 五、关键代码位置索引

| 功能 | 文件 | 行号 |
|-----|------|-----|
| 状态枚举 | `app/models/import.rb` | 24-31 |
| publish_later | `app/models/import.rb` | 56-63 |
| publish | `app/models/import.rb` | 65-75 |
| revert_later | `app/models/import.rb` | 77-83 |
| revert | `app/models/import.rb` | 85-96 |
| TransactionImport#import! | `app/models/transaction_import.rb` | 2-33 |
| AccountImport#import! | `app/models/account_import.rb` | 4-29 |
| TradeImport#import! | `app/models/trade_import.rb` | 2-37 |
| ImportJob | `app/jobs/import_job.rb` | 1-7 |
| RevertImportJob | `app/jobs/revert_import_job.rb` | 1-7 |
| ImportsController | `app/controllers/imports_controller.rb` | 1-69 |

---

## 六、改进建议

### 6.1 增强幂等性
```ruby
# 在 import! 开头添加防重复检查
def import!
  return if complete?  # 幂等保护
  transaction do
    # ...
  end
end
```

### 6.2 清理 Mappings 创建的资源
```ruby
def revert
  Import.transaction do
    # 新增：删除导入创建的 Categories、Tags
    mappings.each do |mapping|
      mapping.mappable.destroy if mapping.create_when_empty && mapping.mappable
    end
    
    accounts.destroy_all
    entries.destroy_all
  end
  # ...
end
```

### 6.3 添加状态超时检测
```ruby
# 建议添加的 scope
scope :stuck_importing, -> { where(status: :importing).where("updated_at < ?", 1.hour.ago) }
scope :stuck_reverting, -> { where(status: :reverting).where("updated_at < ?", 1.hour.ago) }
```

### 6.4 确保期初余额 Entry 关联 import
```ruby
# AccountImport#import! 中
result = manager.set_opening_balance(
  balance: row.amount.to_d,
  import: self  # 传递 import 关联
)
```
