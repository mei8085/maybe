# 交易批量更新一致性分析报告

| 项目 | 内容 |
|------|------|
| 分析主题 | 多条交易记录批量更新的数据一致性保障机制 |
| 分析范围 | 仅限 `Entry.bulk_update!` 链路 |
| 复核状态 | ✅ 已完成严格代码复核 |
| 代码版本 | 当前工作目录版本 |
| 生成日期 | 2026-05-16 |

---

## 1. 执行时序总览

### 1.1 完整执行流程图

```
  用户请求 (HTTP POST)
     │
     ▼
  [控制器层] bulk_updates_controller#create
     │
     ├─ 读：Current.family 获取家庭上下文
     ├─ 读：bulk_update_params 提取字段（见 2.1 节）
     └─ Scope：Current.family.entries.where(id: entry_ids)
          │
          ▼
  [领域层] Entry.bulk_update!(bulk_update_params)
     │
     ├─ 步骤1：构建批量更新属性
     │    └─ 支持字段：date, notes, category_id, merchant_id, tag_ids
     │
     ├─ 步骤2：空值检查（无有效属性时直接 return 0）
     │
     ├─ 步骤3：开启数据库事务
     │    │
     │    ├─ 循环处理每条 Entry
     │    │    ├─ 读：entry.entryable_id
     │    │    ├─ 写：entry.update! bulk_attributes
     │    │    ├─ 读：entry.saved_changes
     │    │    ├─ 写：entry.lock_saved_attributes!
     │    │    └─ 写：entry.entryable.lock_attr!(:tag_ids)
     │    │
     │    └─ 提交事务
     │
     ├─ ⚠️  缺失关键步骤：未触发账户同步 ⚠️
     │
     └─ 返回更新数量（all.size）
```

---

## 2. 严格复核后的字段影响矩阵

### 2.1 批量更新真实支持的输入字段

**代码证据**:
- `app/models/entry.rb:74-82` - `bulk_update!` 方法属性构建
- `app/controllers/transactions/bulk_updates_controller.rb:15-18` - strong parameters 定义

| 字段名称 | 所属层级 | 是否在批量更新输入中 |
|---------|---------|---------------------|
| `date` | Entry 层 | ✅ 是 |
| `notes` | Entry 层 | ✅ 是 |
| `category_id` | Transaction 层 | ✅ 是 |
| `merchant_id` | Transaction 层 | ✅ 是 |
| `tag_ids` | Transaction 层 | ✅ 是 |
| `entry_ids` | 选择条件 | ✅ 是（仅用于筛选） |
| `amount` | Entry 层 | ❌ 否（不属于批量更新输入） |

> 📌 **复核修正点**：此前分析误将 `amount` 列入批量更新字段，实际代码中不存在，已删除。

### 2.2 各字段对账户同步的真实影响

**代码证据**:
- `app/models/entry.rb:46-49` - `sync_account_later` 方法实现
- `app/models/balance/forward_calculator.rb` - 余额计算逻辑

| 字段 | 是否影响余额计算 | 影响机制 | 风险优先级 | 判断依据 |
|------|----------------|---------|-----------|---------|
| `date` | ✅ 是 | 余额计算器**按日期分组**汇总 entries，日期变化会改变交易在余额序列中的位置，影响从变更日期到当前日期的所有余额点 | 🔴 高 | `forward_calculator.rb:10, 24` - 按日期循环并调用 `flows_for_date(date)`；`entry.rb:47` - 同步窗口使用 `date_previously_was` |
| `notes` | ❌ 否 | 纯文本备注字段，不参与任何财务计算 | 🟢 低 | `entry.rb` schema 中仅为 text 字段，无业务逻辑引用 |
| `category_id` | ❌ 否 | 仅用于报表统计和预算分析，不影响账户余额，预算计算不依赖同步机制 | 🟢 低 | 余额计算仅依赖 entries.amount，不引用 transaction.category_id；预算统计为报表层逻辑，无一致性断层风险 |
| `merchant_id` | ❌ 否 | 仅用于商户分析展示，不影响账户余额 | 🟢 低 | 余额计算不引用 transaction.merchant_id |
| `tag_ids` | ❌ 否 | 仅用于筛选标签，不影响账户余额 | 🟢 低 | 余额计算不引用 tags 关联 |

---

## 3. 回滚边界的精确界定

### 3.1 回滚触发条件矩阵

**代码证据**: `app/models/entry.rb:86-94`

| 操作阶段 | 触发异常的场景 | 是否回滚 | 回滚范围 | 数据状态 |
|----------|----------------|----------|----------|----------|
| 参数解析 | 缺少 `bulk_update` 参数、格式错误 | ✅ 自动回滚 | 整个请求 | 无数据变更 |
| 空值检查 | `bulk_attributes.blank?` 为真 | ❌ 不回滚 | 无 | 直接 `return 0`，无数据库操作 |
| 事务 BEGIN 后 | 任何 Ruby 异常（包括 validation 失败） | ✅ 数据库级回滚 | 事务内所有 DML 操作 | 数据库恢复到事务前状态 |
| 单条 entry.update! | 数据验证失败（日期无效、必填字段缺失等） | ✅ 数据库级回滚 | 整个批量操作的所有已更新记录 | ❗ 所有已处理记录全部回滚到初始状态 |
| 属性锁定 | `lock_saved_attributes!` 执行异常 | ✅ 数据库级回滚 | 事务内所有操作 | 包括已执行的 entry.update! 全部回滚 |
| 事务 COMMIT 后 | 任何后续逻辑异常 | ❌ 不回滚 | 无 | 数据库已持久化，无法自动回滚 |

### 3.2 关键结论

> **"全有或全无"语义**：批量更新采用原子事务设计，任何一条记录验证失败，都会导致整个批次的所有成功更新全部回滚。

---

## 4. 同步缺失问题的严格复核

### 4.1 代码证据对比

#### ✅ 单笔交易更新：正确触发同步

**文件**: `app/controllers/transactions_controller.rb:77-89`

```ruby
if @entry.update(entry_params)
  # ...
  @entry.sync_account_later  # ✅ 显式触发同步
  @entry.lock_saved_attributes!
  # ...
end
```

**触发时机**：无论什么字段被更新，单笔更新后无条件触发账户同步。

#### ✅ 批量删除：正确触发同步

**文件**: `app/controllers/transactions/bulk_deletions_controller.rb:3-4`

```ruby
def create
  destroyed = Current.family.entries.destroy_by(id: bulk_delete_params[:entry_ids])
  destroyed.map(&:account).uniq.each(&:sync_later)  # ✅ 删除后触发所有相关账户同步
  # ...
end
```

#### ❌ 批量更新：未触发同步（双重确认）

**证据1**：`app/models/entry.rb:95-97` - `bulk_update!` 方法尾部

```ruby
      end  # transaction 结束
    end

    all.size  # ← 直接返回，无 sync_later 调用
  end
```

**证据2**：`app/controllers/transactions/bulk_updates_controller.rb:5-11` - 控制器侧

```ruby
def create
  updated = Current.family
                   .entries
                   .where(id: bulk_update_params[:entry_ids])
                   .bulk_update!(bulk_update_params)

  # ❗ 控制器也未补充同步调用

  redirect_back_or_to transactions_path, notice: "#{updated} transactions updated"
end
```

### 4.2 影响范围精确判定

| 影响领域 | 严重程度 | 触发条件 | 具体表现 |
|----------|----------|---------|----------|
| 账户余额准确性 | 🔴 高 | 批量更新包含 `date` 字段 | `accounts.balance` 和 `accounts.cash_balance` 字段停留在旧值；`balances` 表数据不一致 |
| 余额图表展示 | 🔴 高 | 批量更新包含 `date` 字段 | 账户余额趋势图、收支曲线图显示错误数据 |
| 所有其他字段更新 | 🟢 低 | 批量更新 notes/category_id/merchant_id/tag_ids | 无财务数据一致性问题，仅影响报表展示，无一致性断层风险 |

---

## 5. 修复建议

### 5.1 立即修复：补充同步触发

**建议修改** `app/models/entry.rb:94-97`，在事务结束后添加同步逻辑：

```ruby
      end  # transaction 结束

      # ✅ 新增：仅当批次中存在日期变更时触发同步
      if all.any? { |entry| entry.saved_change_to_date? }
        affected_accounts = all.map(&:account).uniq
        min_date = all.map { |entry| [ entry.date_previously_was, entry.date ].compact.min }.min

        affected_accounts.each do |account|
          account.sync_later(window_start_date: min_date)
        end
      end

      all.size
    end
```

**设计理由**:
- 仅在实际发生日期变更时触发，避免无效的同步任务
- 使用批量中最小日期作为同步窗口起点，确保完整重算
- 按账户去重，避免同一账户被多次同步

---

## 6. 复核总结

### 6.1 关键发现总览

| 发现项 | 复核状态 | 说明 |
|--------|---------|------|
| 事务原子性 | ✅ 正确 | 采用"全有或全无"策略，数据库事务保障正确 |
| 属性锁定机制 | ✅ 正确 | 用户编辑后立即锁定，防止规则引擎覆盖 |
| 同步触发缺失 | ❌ 问题确认 | 批量更新后未调用 `sync_later`，**仅日期变更**会导致余额数据不一致 |
| 字段影响矩阵 | 🔄 已修正 | 删除了不存在的 `amount` 字段，统一了 category_id 风险口径 |

### 6.2 核心结论

1. **一致性断层仅影响日期变更**：只有当批量更新包含 `date` 字段时，才会产生真实的财务数据一致性问题
2. **其他字段无风险**：notes/category_id/merchant_id/tag_ids 的批量更新**均不影响账户余额**，仅影响报表展示，无一致性断层风险
3. **修复成本较低**：仅需在 `bulk_update!` 方法末尾添加条件同步逻辑

### 6.3 行动建议

| 优先级 | 行动 | 原因 |
|--------|------|------|
| 🔴 高 | 补充批量更新后的同步触发逻辑 | 日期变更会导致余额计算错误 |
| 🟡 中 | 添加集成测试覆盖批量更新日期变更场景 | 防止回归 |
| 🟢 低 | 整理代码注释说明字段影响 | 便于后续维护 |

---

## 附录：代码引用索引

| 逻辑点 | 文件路径 | 关键行号 |
|-------|---------|---------|
| 批量更新控制器入口 | `app/controllers/transactions/bulk_updates_controller.rb` | 5-11 |
| Entry.bulk_update! 方法实现 | `app/models/entry.rb` | 73-97 |
| Entry.sync_account_later 方法 | `app/models/entry.rb` | 46-49 |
| 单笔更新同步触发 | `app/controllers/transactions_controller.rb` | 88 |
| 批量删除同步触发 | `app/controllers/transactions/bulk_deletions_controller.rb` | 3-4 |
| 余额正向计算器 | `app/models/balance/forward_calculator.rb` | 10, 24, 68 |
| Enrichable 属性锁定机制 | `app/models/concerns/enrichable.rb` | 69-73 |
