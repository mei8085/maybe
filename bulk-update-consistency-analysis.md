# 交易记录批量更新一致性分析报告

## 文档信息

| 项目 | 内容 |
|------|------|
| 分析主题 | 多条交易记录批量更新的数据一致性保障机制 |
| 涉及模块 | Family（家庭模型）、Entry（交易记录层）、Balance/Sync（余额同步层） |
| 代码版本 | 当前工作目录版本 |
| 生成日期 | 2026-05-16 |

---

## 1. 执行时序总览

### 1.1 完整执行流程图

```
  用户请求
     │
     ▼
  [HTTP层] bulk_updates_controller#create
     │
     ├─ 读：Current.family （获取家庭上下文）
     │
     ▼
  [领域层] Entry.bulk_update!(params)
     │
     ├─ 步骤1：构建批量更新属性
     │    └─ 读：bulk_update_params [:date, :notes, :category_id, :merchant_id, :tag_ids]
     │
     ├─ 步骤2：空值检查（无更新属性直接返回 0）
     │
     ├─ 步骤3：开启数据库事务 ────────────────────────┐
     │    │                                              │
     │    ├─ 循环处理每条 Entry                          │
     │    │    ├─ 读：entry.entryable_id                │
     │    │    ├─ 写：entry.update! bulk_attributes    │──┼── 事务边界：失败回滚
     │    │    ├─ 读：entry.saved_changes               │
     │    │    ├─ 写：entry.lock_saved_attributes!     │
     │    │    └─ 写：entryable.lock_attr!(:tag_ids)   │
     │    │                                              │
     │    └─ 提交事务 ───────────────────────────────────┘
     │
     ├─ ⚠️  缺失：批量更新后未触发账户同步 ⚠️
     │
     └─ 返回更新数量
```

---

## 2. 详细时序与读写动作分析

### 2.1 时序步骤分解

| 序号 | 层级 | 操作 | 读动作 | 写动作 | 触发条件 | 失败处理 |
|------|------|------|--------|--------|----------|----------|
| **1** | 控制器层 | `bulk_updates_controller#create` | `Current.family` (从当前会话获取) | - | HTTP POST 请求到达 | 异常向上抛出，Rails 渲染 500 |
| **2** | 控制器层 | `Current.family.entries.where(id: ...)` | 读取符合 ID 列表的 Entry 集合 | - | 参数包含有效 `entry_ids` 数组 | 无效 ID 时返回空集合 |
| **3** | 模型层 | `Entry.bulk_update!` 入口 | - | - | 批量更新方法被调用 | - |
| **4** | 模型层 | 构建 `bulk_attributes` | 读取 `params[:date, :notes, :category_id, :merchant_id, :tag_ids]` | - | 参数解析成功 | - |
| **5** | 模型层 | 空值检查 | 检查 `bulk_attributes.blank?` | - | 无有效更新属性 | 直接 `return 0`，不执行后续操作 |
| **6** | 模型层 | **开启数据库事务** | - | 事务 BEGIN | 存在有效更新属性 | 事务内任何异常触发回滚 |
| **7** | 模型层 | `entry.update! bulk_attributes` | 读取 `entry.entryable_id` 用于嵌套更新 | 更新 `entries` 表 + `entryable` 关联表（Transaction） | 遍历每条记录执行 | ❗ **验证失败抛出异常，回滚所有已执行更新** |
| **8** | 模型层 | `entry.lock_saved_attributes!` | 读取 `entry.saved_changes` 哈希 | 更新 `entries.locked_attributes` JSONB 字段 | `update!` 执行成功后 | 异常触发事务回滚 |
| **9** | 模型层 | `entryable.lock_attr!(:tag_ids)` | 检查 `entry.transaction?` 和 `tags.any?` | 更新 `transactions.locked_attributes` JSONB 字段 | 交易存在标签时执行 | 异常触发事务回滚 |
| **10** | 模型层 | **提交数据库事务** | - | 事务 COMMIT | 所有记录循环执行完成 | - |
| **11** | 模型层 | 返回 `all.size` | - | - | 事务提交成功 | - |
| **12** | **缺失** | ❗ **账户同步触发** ❗ | - | - | - | **未执行！数据不一致风险** |

---

## 3. 回滚边界的精确界定

### 3.1 回滚触发条件矩阵

| 操作阶段 | 触发异常的场景 | 是否回滚 | 回滚范围 | 数据状态 |
|----------|----------------|----------|----------|----------|
| **参数解析** | 参数格式错误、缺少必要字段 | ✅ 自动回滚 | 整个请求 | 无数据变更 |
| **空值检查前** | 任何前置逻辑异常 | ✅ 自动回滚 | 整个请求 | 无数据变更 |
| **事务 BEGIN 后** | 任何 Ruby 异常（ActiveRecord::RecordInvalid 等） | ✅ 数据库级回滚 | 事务内所有 DML 操作 | 数据库恢复到事务前状态 |
| **单条 entry.update!** | 数据验证失败（日期无效、金额缺失等） | ✅ 数据库级回滚 | 整个批量操作的所有已更新记录 | ❗ 所有已处理记录全部回滚到初始状态 |
| **属性锁定** | `lock_saved_attributes!` 或 `lock_attr!` 异常 | ✅ 数据库级回滚 | 事务内所有操作 | 包括已执行的 entry.update! 全部回滚 |
| **事务 COMMIT 后** | 任何后续逻辑异常 | ❌ **不回滚** | 无 | 数据库已持久化，无法自动回滚 |
| **同步过程中** | Sync Job 执行失败 | ❌ **不回滚** | 仅余额计算 | 交易数据已提交，余额可能不一致 |

### 3.2 关键回滚边界代码证据

**事务边界代码** (`app/models/entry.rb:86-94`):

```ruby
transaction do  # ────────────────────────────────────── 回滚边界起点
  all.each do |entry|
    bulk_attributes[:entryable_attributes][:id] = entry.entryable_id if bulk_attributes[:entryable_attributes].present?
    entry.update! bulk_attributes  # ← 抛出 ActiveRecord::RecordInvalid 时，整个事务回滚

    entry.lock_saved_attributes!   # ← 此处异常同样触发完整回滚
    entry.entryable.lock_attr!(:tag_ids) if entry.transaction? && entry.transaction.tags.any?
  end
end  # ─────────────────────────────────────────────── 回滚边界终点
```

**重要结论**:
> 批量更新采用"全有或全无"策略：任何一条记录验证失败，将导致所有已成功更新的记录全部回滚。

---

## 4. 批量更新后账户同步缺失的代码证据与影响

### 4.1 代码证据对比

#### ✅ 单笔交易更新：正确触发同步

**文件**: `app/controllers/transactions_controller.rb:60-61, 77-88`

```ruby
# 创建时
if @entry.save
  @entry.sync_account_later  # ✅ 显式触发同步
  @entry.lock_saved_attributes!
  # ...
end

# 更新时
if @entry.update(entry_params)
  # ...
  @entry.sync_account_later  # ✅ 显式触发同步
  @entry.lock_saved_attributes!
  # ...
end
```

#### ✅ 批量删除：正确触发同步

**文件**: `app/controllers/transactions/bulk_deletions_controller.rb:3-4`

```ruby
def create
  destroyed = Current.family.entries.destroy_by(id: bulk_delete_params[:entry_ids])
  destroyed.map(&:account).uniq.each(&:sync_later)  # ✅ 删除后触发所有相关账户同步
  # ...
end
```

#### ❌ 批量更新：未触发同步

**文件**: `app/models/entry.rb:73-97` - `bulk_update!` 方法完整代码

```ruby
def bulk_update!(bulk_update_params)
  bulk_attributes = {
    date: bulk_update_params[:date],
    notes: bulk_update_params[:notes],
    entryable_attributes: {
      category_id: bulk_update_params[:category_id],
      merchant_id: bulk_update_params[:merchant_id],
      tag_ids: bulk_update_params[:tag_ids]
    }.compact_blank
  }.compact_blank

  return 0 if bulk_attributes.blank?

  transaction do
    all.each do |entry|
      bulk_attributes[:entryable_attributes][:id] = entry.entryable_id if bulk_attributes[:entryable_attributes].present?
      entry.update! bulk_attributes

      entry.lock_saved_attributes!
      entry.entryable.lock_attr!(:tag_ids) if entry.transaction? && entry.transaction.tags.any?
    end
  end

  # ❗ 此处缺少：
  # all.map(&:account).uniq.each(&:sync_later)
  # ❗ 没有任何账户同步触发逻辑

  all.size
end
```

**控制器调用侧** (`app/controllers/transactions/bulk_updates_controller.rb:5-11`):

```ruby
def create
  updated = Current.family
                   .entries
                   .where(id: bulk_update_params[:entry_ids])
                   .bulk_update!(bulk_update_params)

  # ❗ 控制器也未补充同步调用

  redirect_back_or_to transactions_url, notice: "#{updated} transactions updated"
end
```

### 4.2 影响判断

| 影响领域 | 严重程度 | 具体表现 |
|----------|----------|----------|
| **账户余额准确性** | 🔴 高 | `accounts.balance` 和 `accounts.cash_balance` 字段停留在旧值 |
| **历史余额序列** | 🔴 高 | `balances` 表数据不一致，日期维度的余额计算错误 |
| **余额图表展示** | 🔴 高 | 账户余额趋势图、收支曲线图显示错误数据 |
| **预算计算** | 🟡 中 | 分类预算统计基于旧数据，实际与预算偏差 |
| **转账匹配** | 🟡 中 | 跨账户转账自动匹配可能失败或延迟 |
| **数据缓存** | 🟡 中 | Family 层缓存键未失效，聚合查询返回旧数据 |

### 4.3 触发条件影响矩阵

| 批量更新的字段 | 是否需要账户同步 | 实际是否触发 | 影响程度 |
|----------------|------------------|--------------|----------|
| `date`（交易日期） | ✅ **必须** | ❌ 否 | 🔴 高 - 余额计算依赖日期排序 |
| `amount`（金额） | ✅ **必须** | ❌ 否 | 🔴 高 - 直接影响余额结果 |
| `category_id`（分类） | ❌ 不需要 | ❌ 否 | 🟢 低 - 不影响余额，仅影响统计 |
| `merchant_id`（商户） | ❌ 不需要 | ❌ 否 | 🟢 低 - 不影响余额 |
| `notes`（备注） | ❌ 不需要 | ❌ 否 | 🟢 低 - 纯文本字段 |
| `tag_ids`（标签） | ❌ 不需要 | ❌ 否 | 🟢 低 - 仅影响筛选 |

---

## 5. 各层级职责与协作关系

### 5.1 三层架构职责划分

| 层级 | 核心模型 | 主要职责 | 一致性保障手段 |
|------|---------|---------|---------------|
| **家庭层 (Family)** | `Family` | 1. 数据边界隔离<br>2. 聚合根入口<br>3. 跨账户资源管理 | 通过 `has_many through:` 关联确保范围查询，配合 `Current.family` 上下文隔离 |
| **账户层 (Account)** | `Account` + `Balance` + `Sync` | 1. 余额物化计算<br>2. 异步同步编排<br>3. 账户级状态管理 | 1. `Balance::Materializer` 事务内计算<br>2. Sync 状态机 + 幂等扩展窗口<br>3. `with_lock` 悲观锁防并发 |
| **交易层 (Entry)** | `Entry` + `Transaction` | 1. 单笔交易验证<br>2. 属性变更锁定<br>3. 同步触发点 | 1. 数据库验证约束<br>2. `Enrichable` 锁定机制<br>3. 主动调用 `sync_account_later` |

### 5.2 数据流动方向

```
  用户操作
     │
     ▼
  [Entry 层] 交易记录变更 ──→ 属性锁定 (Enrichable)
     │
     │  理想情况：主动触发 sync_account_later
     ▼
  [Account 层] Sync 记录创建 ──→ SyncJob 入队
     │
     ▼
  [Balance 层] Materializer 执行 ──→ 余额计算 + 持久化
     │
     ▼
  [Family 层] 缓存失效 ──→ 聚合查询数据更新
```

---

## 6. 修复建议与最佳实践

### 6.1 立即修复：补充同步触发

**建议修改** `app/models/entry.rb:94-95` 之间添加同步调用：

```ruby
  end  # transaction 结束

  # 新增：触发相关账户同步
  all.map(&:account).uniq.each do |account|
    min_date = all.map(&:date).min
    account.sync_later(window_start_date: min_date)
  end

  all.size
end
```

### 6.2 改进建议：事务边界优化

```ruby
# 当前：所有记录在一个事务中（可能导致长事务）
transaction do
  all.each { |entry| ... }
end

# 建议：分批处理，降低锁竞争
all.in_batches(of: 50) do |batch|
  Entry.transaction do
    batch.each { |entry| ... }
  end
  # 每批完成后触发同步
end
```

### 6.3 增强：批量操作的一致性测试

建议添加集成测试覆盖以下场景：
1. 部分记录验证失败时的回滚完整性
2. 批量更新后余额数据的正确性验证
3. 跨账户批量更新时多个账户同步触发情况

---

## 7. 总结

### 7.1 关键发现

| 发现项 | 状态 | 说明 |
|--------|------|------|
| 事务原子性 | ✅ 良好 | 采用"全有或全无"策略，数据库事务保障正确 |
| 属性锁定机制 | ✅ 良好 | 用户编辑后立即锁定，防止规则引擎覆盖 |
| **账户同步触发** | ❌ **缺失** | 批量更新后未调用 `sync_later`，导致余额数据不一致 |
| 回滚边界 | ✅ 清晰 | 数据库事务边界明确，异常处理路径清晰 |

### 7.2 核心结论

1. **数据一致性存在断层**: 批量更新操作在事务提交后，未将变更通知传递到余额同步层
2. **日期变更影响最大**: 当批量更新包含 `date` 字段时，余额计算依赖的时间序列排序完全失效
3. **修复成本较低**: 仅需在 `bulk_update!` 方法末尾添加 3-4 行代码即可修复
4. **建议优先级**: 🔴 高优先级，应立即修复

---

## 附录：相关文件路径索引

| 文件 | 路径 | 关键行号 |
|------|------|----------|
| 批量更新控制器 | `app/controllers/transactions/bulk_updates_controller.rb` | 5-11 |
| Entry 批量更新方法 | `app/models/entry.rb` | 73-97 |
| 单笔更新同步触发 | `app/controllers/transactions_controller.rb` | 61, 88 |
| 批量删除同步触发 | `app/controllers/transactions/bulk_deletions_controller.rb` | 3-4 |
| Sync 状态机 | `app/models/sync.rb` | 27-51 |
| 余额物化器 | `app/models/balance/materializer.rb` | 9-52 |
| Enrichable 锁定机制 | `app/models/concerns/enrichable.rb` | 69-73 |
