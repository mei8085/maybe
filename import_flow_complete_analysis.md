# 导入流程完整链路重分析（含状态门禁偏差与漏掉分支）

## 一、核心发现（修正上一轮的偏差）

### 1.1 上一轮分析的两个关键偏差

**偏差1：状态门禁检查位置不一致**
- ❌ 上一轮认为：`publish` 有完整的状态门禁
- ✅ 实际情况：`publish_later`（Controller调用）检查 `publishable?`，但 `publish`（Job执行）**不检查** `publishable?`
- 影响：Job排队期间若数据变化，可能导致不可发布的数据被执行

**偏差2：漏掉的导入分支 - MintImport**
- ❌ 上一轮只覆盖了：TransactionImport, TradeImport, AccountImport
- ✅ 实际有：**4类导入**，MintImport 有独特的逐行创建逻辑和签名处理

---

## 二、完整状态机与门禁设计

### 2.1 状态定义（不变）
```ruby
enum :status, {
  pending: "pending",        # 初始/配置中
  importing: "importing",    # 后台执行中
  complete: "complete",      # 成功完成
  reverting: "reverting",    # 回滚执行中
  revert_failed: "revert_failed", # 回滚失败（可重试）
  failed: "failed"           # 导入失败（可重试）
}, validate: true, default: "pending"
```

### 2.2 状态门禁校验链（关键修正）

```
┌─────────────────────────────────────────────────────────────┐
│ 用户点击确认 → ImportsController#publish                    │
└─────────────────────────┬───────────────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────────────┐
│ publish_later 【有完整门禁】                                 │
│   ├─ ✅ 检查：row_count_exceeded? （行数超限）              │
│   ├─ ✅ 检查：publishable?                                   │
│   │      └─ cleaned? (uploaded? && rows.any? && rows.all?&:valid?)
│   │      └─ mappings.all?(&:valid?)                         │
│   ├─ ✅ 状态更新：status → :importing                       │
│   └─ 📥 入队：ImportJob.perform_later(self)                 │
└─────────────────────────┬───────────────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────────────┐
│ 【门禁断档区域】 Sidekiq 排队（可能数秒~数分钟）            │
│  ❗风险：期间数据可能被修改，导致不再 publishable            │
└─────────────────────────┬───────────────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────────────┐
│ ImportJob#perform → publish 【门禁严重缺失】                │
│   ├─ ⚠️  仅检查：row_count_exceeded? （只检查行数）         │
│   ├─ ❌ 未检查：publishable?                                │
│   ├─ ❌ 未检查：status == :importing                        │
│   ├─ 执行：import! (子类事务内)                             │
│   ├─ 成功：status → :complete                               │
│   └─ 失败：status → :failed, 记录 error.message             │
└─────────────────────────────────────────────────────────────┘
```

### 2.3 回滚流程的状态门禁

```
┌─────────────────────────────────────────────────────────────┐
│ 用户点击撤销 → ImportsController#revert                     │
└─────────────────────────┬───────────────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────────────┐
│ revert_later 【有完整门禁】                                 │
│   ├─ ✅ 检查：revertable? (complete? || revert_failed?)     │
│   ├─ ✅ 状态更新：status → :reverting                       │
│   └─ 📥 入队：RevertImportJob.perform_later(self)           │
└─────────────────────────┬───────────────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────────────┐
│ RevertImportJob#perform → revert 【门禁缺失】               │
│   ├─ ❌ 未检查：status == :reverting                        │
│   ├─ 执行：accounts.destroy_all + entries.destroy_all       │
│   ├─ 成功：status → :pending                                │
│   └─ 失败：status → :revert_failed, 记录 error.message      │
└─────────────────────────────────────────────────────────────┘
```

---

## 三、四类导入分支的完整落盘与回滚分析

### 3.1 TransactionImport（CSV交易导入）

#### 导入落盘
```ruby
def import!
  transaction do
    # 步骤1：创建映射资源（Category、Tag、Account）
    mappings.each(&:create_mappable!)
    #   CategoryMapping: find_or_create_by!(name: key)
    #   TagMapping:      find_or_create_by!(name: key)
    #   AccountMapping:  create_or_find_by!(name: key) + 设置 import_id

    # 步骤2：构建所有 Transaction + Entry
    transactions = rows.map do |row|
      Transaction.new(
        entry: Entry.new(import: self, ...)  # ✅ 设置 import_id
      )
    end

    # 步骤3：批量导入
    Transaction.import!(transactions, recursive: true)
  end
end
```

#### 回滚副作用
| 落盘数据 | 回滚删除？ | 备注 |
|---------|-----------|------|
| import_rows | ✅ | 通过 import.rows 关联删除 |
| import_mappings | ✅ | 通过 import.mappings 关联删除 |
| Entry (import_id=?) | ✅ | entries.destroy_all |
| Transaction (通过 entry) | ✅ | entry.destroy 级联删除 |
| Tagging (通过 transaction) | ✅ | transaction.destroy 级联删除 |
| **Category (find_or_create)** | ❌ | 不删除，可能被其他交易使用 |
| **Tag (find_or_create)** | ❌ | 不删除，可能被其他交易使用 |
| **Account (create_or_find)** | ⚠️ 条件删除 | 仅当 account.import_id = self 时通过 import.accounts.destroy_all 删除 |

---

### 3.2 TradeImport（交易导入）

#### 导入落盘
```ruby
def import!
  transaction do
    # 步骤1：创建映射资源（Account）
    mappings.each(&:create_mappable!)

    # 步骤2：构建所有 Trade + Entry + Security
    trades = rows.map do |row|
      Trade.new(
        security: find_or_create_security(...),  # ❗ Security 不关联 import
        entry: Entry.new(import: self, ...)      # ✅ 设置 import_id
      )
    end

    # 步骤3：批量导入
    Trade.import!(trades, recursive: true)
  end
end
```

#### 回滚副作用
| 落盘数据 | 回滚删除？ | 备注 |
|---------|-----------|------|
| import_rows | ✅ | |
| import_mappings | ✅ | |
| Entry (import_id=?) | ✅ | entries.destroy_all |
| Trade (通过 entry) | ✅ | entry.destroy 级联删除 |
| **Security (find_or_create)** | ❌ | 设计上合理，可被其他交易复用 |
| **Account (AccountMapping 创建)** | ⚠️ 条件删除 | 仅 account.import_id = self 时删除 |

---

### 3.3 AccountImport（账户导入）

#### 导入落盘
```ruby
def import!
  transaction do
    rows.each do |row|
      # 步骤1：创建 Account
      account = family.accounts.build(
        import: self,  # ✅ 设置 import_id
        ...
      )
      account.save!

      # 步骤2：设置期初余额
      manager = Account::OpeningBalanceManager.new(account)
      result = manager.set_opening_balance(balance: row.amount.to_d)
      # ❗ 风险：期初余额 Entry 是否设置 import_id？
    end
  end
end
```

#### 回滚副作用
| 落盘数据 | 回滚删除？ | 备注 |
|---------|-----------|------|
| import_rows | ✅ | |
| import_mappings | ✅ | |
| Account (import_id=?) | ✅ | accounts.destroy_all |
| **期初余额 Entry** | ❓ 待确认 | 需要检查 OpeningBalanceManager 是否设置 import_id |
| Accountable (Depository 等) | ✅ | account.destroy 级联删除 |
| Holdings / Balances | ✅ | account.destroy 级联删除 |

---

### 3.4 MintImport 【漏掉的分支！】

#### 独特实现（逐行创建，非批量）
```ruby
def import!
  transaction do
    # 步骤1：创建映射资源（Category、Tag、Account）
    mappings.each(&:create_mappable!)

    # 步骤2：逐行创建 Entry（独特之处：不是批量 import!）
    rows.each do |row|
      account = mappings.accounts.mappable_for(row.account)
      category = mappings.categories.mappable_for(row.category)
      tags = row.tags_list.map { |tag| mappings.tags.mappable_for(tag) }.compact

      # 逐行 build + save
      entry = account.entries.build \
        date: row.date_iso,
        amount: row.signed_amount,  # 独特：根据 Transaction Type 计算签名
        name: row.name,
        entryable: Transaction.new(category: category, tags: tags),
        import: self  # ✅ 设置 import_id

      entry.save!  # ❗ 逐行保存，失败时事务回滚已创建的行
    end
  end
end
```

#### 独特签名处理
```ruby
def signed_csv_amount(csv_row)
  amount = csv_row[amount_col_label]
  type = csv_row["Transaction Type"]

  if type == "credit"
    amount.to_d          # 流入：正数
  else
    amount.to_d * -1     # 流出：转负数
  end
end
```

#### 回滚副作用
| 落盘数据 | 回滚删除？ | 备注 |
|---------|-----------|------|
| import_rows | ✅ | |
| import_mappings | ✅ | |
| Entry (import_id=?) | ✅ | entries.destroy_all |
| Transaction (通过 entry) | ✅ | |
| Tagging (通过 transaction) | ✅ | |
| **Category (find_or_create)** | ❌ | |
| **Tag (find_or_create)** | ❌ | |
| **Account (AccountMapping 创建)** | ⚠️ 条件删除 | 仅 account.import_id = self 时删除 |

---

## 四、失败重试机制的实际作用

### 4.1 状态门禁在重试时的真实行为

#### 场景1：导入失败后重试
```
状态流转：pending → importing → failed
                          ↑
                     用户修复配置后重试

重试时的门禁检查：
  publish_later 检查 publishable?
    ├─ cleaned? → rows.all?(&:valid?) ✅ （用户已修复行数据）
    └─ mappings.all?(&:valid?) ✅ （用户已修复映射）

✅ 结果：可以正常重试
```

#### 场景2：回滚失败后重试
```
状态流转：complete → reverting → revert_failed
                              ↑
                         用户重试撤销

重试时的门禁检查：
  revert_later 检查 revertable?
    └─ complete? || revert_failed? → true ✅

✅ 结果：可以正常重试（revert_failed 状态被允许）
```

#### 场景3：Job中途失败（Sidekiq重试机制）
```
状态：importing （卡死！）

Sidekiq 重试时调用 publish：
  ├─ 检查 row_count_exceeded? → 行数没变化 ✅
  ├─ ❌ 未检查 status == importing → 直接执行 import!
  ├─ ❗ 风险：重复创建条目！

❌ 结果：可能导致数据重复入账！
```

### 4.2 幂等性缺陷矩阵

| 重试场景 | 当前状态 | publish检查 | 实际风险 |
|---------|---------|------------|---------|
| 用户手动重试导入失败 | failed | ✅ publishable? | 安全 |
| 用户手动重试回滚失败 | revert_failed | ✅ revertable? | 安全 |
| Sidekiq自动重试导入 | importing | ❌ 仅检查行数 | 可能重复入账 |
| Sidekiq自动重试回滚 | reverting | ❌ 无检查 | 安全（destroy_all幂等） |
| 恶意调用 publish（complete状态） | complete | ❌ 仅检查行数 | 严重风险！重复执行 |

---

## 五、完整副作用清单与事务边界

### 5.1 导入阶段事务边界

```
┌─────────────────────────────────────────────────────────────────┐
│ publish_later 【非事务】                                         │
│   └─ update!(status: :importing)                                │
└─────────────────────────────────────────────────────────────────┘
                          ↓ 异步
┌─────────────────────────────────────────────────────────────────┐
│ publish → import! 【数据库事务 BEGIN】                          │
│   ├─ mappings.each(&:create_mappable!)                          │
│   │   ├─ Category  (find_or_create)  ← 事务外可见？NO          │
│   │   ├─ Tag       (find_or_create)  ← 事务外可见？NO          │
│   │   └─ Account   (create_or_find) ← 事务外可见？NO           │
│   ├─ 构建 entries / transactions / trades                        │
│   └─ 批量 insert / 逐行 save                                      │
│                        【数据库事务 COMMIT】                      │
└─────────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────────┐
│ 事务外操作                                                        │
│   ├─ family.sync_later （异步计算余额）                           │
│   └─ update!(status: :complete / :failed)                       │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 回滚阶段事务边界

```
┌─────────────────────────────────────────────────────────────────┐
│ revert_later 【非事务】                                          │
│   └─ update!(status: :reverting)                                │
└─────────────────────────────────────────────────────────────────┘
                          ↓ 异步
┌─────────────────────────────────────────────────────────────────┐
│ revert 【数据库事务 BEGIN】                                      │
│   ├─ accounts.destroy_all                                        │
│   │   └─ 删除 Account (import_id=self) + 级联删除其 entries 等  │
│   └─ entries.destroy_all                                         │
│       └─ 删除 Entry (import_id=self) + 级联删除 transaction/trade
│                        【数据库事务 COMMIT】                      │
└─────────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────────┐
│ 事务外操作                                                        │
│   ├─ family.sync_later                                           │
│   └─ update!(status: :pending / :revert_failed)                 │
└─────────────────────────────────────────────────────────────────┘
```

### 5.3 永久残留数据清单（回滚不删除）

| 数据类型 | 创建位置 | 不删除原因 | 影响范围 |
|---------|---------|-----------|---------|
| Category | CategoryMapping#create_mappable! | find_or_create，可能被其他交易使用 | 全局，可被后续导入复用 |
| Tag | TagMapping#create_mappable! | 同上 | 全局，可被后续导入复用 |
| Security | TradeImport#import! | find_or_create，可被多个交易引用 | 全局，可被后续导入复用 |
| Account (AccountMapping 但 import_id=nil) | AccountMapping#create_mappable! | create_or_find_by 找到的已有账户不会设置 import_id | 不会被删除，设计合理 |

---

## 六、关键代码位置索引（精确到行）

| 功能 | 文件 | 行号 | 备注 |
|-----|------|-----|------|
| publish_later 门禁 | `app/models/import.rb` | 56-63 | 有完整检查 |
| publish 门禁缺失 | `app/models/import.rb` | 65-75 | 仅检查行数 |
| revert_later 门禁 | `app/models/import.rb` | 77-83 | 有完整检查 |
| revert 门禁缺失 | `app/models/import.rb` | 85-96 | 无状态检查 |
| TransactionImport#import! | `app/models/transaction_import.rb` | 2-33 | 批量导入 |
| TradeImport#import! | `app/models/trade_import.rb` | 2-37 | 含 Security 创建 |
| AccountImport#import! | `app/models/account_import.rb` | 4-29 | 含期初余额 |
| **MintImport#import!** | `app/models/mint_import.rb` | 23-44 | 逐行创建，漏掉的分支 |
| CategoryMapping#create_mappable! | `app/models/import/category_mapping.rb` | 33-38 | find_or_create |
| TagMapping#create_mappable! | `app/models/import/tag_mapping.rb` | 34-39 | find_or_create |
| AccountMapping#create_mappable! | `app/models/import/account_mapping.rb` | 35-47 | create_or_find_by |

---

## 七、修正建议（针对发现的偏差）

### 7.1 补全 publish 方法的状态门禁
```ruby
def publish
  # 新增：状态幂等性检查
  return if complete?
  # 新增：状态合法性检查
  raise "Import status must be 'importing'" unless importing?

  raise MaxRowCountExceededError if row_count_exceeded?

  import!
  family.sync_later
  update! status: :complete
rescue => error
  update! status: :failed, error: error.message
end
```

### 7.2 补全 revert 方法的状态门禁
```ruby
def revert
  # 新增：状态合法性检查
  raise "Import status must be 'reverting'" unless reverting?

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

### 7.3 可选：清理 Mapping 创建的 Category/Tag
```ruby
def revert
  Import.transaction do
    # 新增：仅删除此导入独有的映射资源（无其他导入/交易使用时）
    mappings.each do |mapping|
      if mapping.create_when_empty && mapping.mappable && safe_to_delete?(mapping.mappable)
        mapping.mappable.destroy
      end
    end

    accounts.destroy_all
    entries.destroy_all
  end
  # ...
end
```

### 7.4 处理 Job 状态卡死
```ruby
# 新增：清理卡死状态的 scope 和方法
scope :stuck_importing, -> { where(status: :importing).where("updated_at < ?", 1.hour.ago) }
scope :stuck_reverting, -> { where(status: :reverting).where("updated_at < ?", 1.hour.ago) }

def reset_stuck_status!
  if stuck_importing? || stuck_reverting?
    update!(status: :failed, error: "Import timed out, please retry")
  end
end
```
