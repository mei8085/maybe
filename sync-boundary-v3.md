# 父子同步关系传递边界分析 v3

## 一、问题核心：`sync_later` 复用时 `parent_sync` 不更新

### 1.1 代码位置与问题

```ruby
# app/models/concerns/syncable.rb:14-35
def sync_later(parent_sync: nil, window_start_date: nil, window_end_date: nil)
  Sync.transaction do
    with_lock do
      sync = self.syncs.incomplete.first

      if sync
        # 问题所在：复用已有 sync 时，只做了窗口扩展
        # 没有更新 parent_sync 字段！
        Rails.logger.info("There is an existing sync, expanding window if needed (#{sync.id})")
        sync.expand_window_if_needed(window_start_date, window_end_date)
      else
        # 新建时才设置 parent
        sync = self.syncs.create!(
          parent: parent_sync,  # ← 只有这里设置 parent
          window_start_date: window_start_date,
          window_end_date: window_end_date
        )
        SyncJob.perform_later(sync)
      end
      sync
    end
  end
end
```

### 1.2 `expand_window_if_needed` 的能力边界

```ruby
# app/models/sync.rb:107-127
def expand_window_if_needed(new_window_start_date, new_window_end_date)
  return unless pending?

  # 只更新窗口日期，不更新 parent！
  update(
    window_start_date: earliest_start_date,
    window_end_date: latest_end_date
  )
end
```

**`expand_window_if_needed` 只做两件事**：
1. 检查 `pending?` 状态
2. 更新 `window_start_date` 和 `window_end_date`

**它不做的事**：
- ❌ 不更新 `parent_id`
- ❌ 不更新任何其他字段

---

## 二、收敛机制的依赖关系

### 2.1 收敛依赖 `parent` 关联

```ruby
# app/models/sync.rb:82-104
def finalize_if_all_children_finalized
  Sync.transaction do
    lock!
    return unless all_children_finalized?

    if syncing?
      has_failed_children? ? fail! : complete!
    end

    perform_post_sync
  end

  # 关键：子同步完成后通过 parent 关联通知父同步
  parent&.finalize_if_all_children_finalized  # ← 依赖 sync.parent 字段
end
```

**父同步如何知道子同步完成？**
- 通过 `parent&.finalize_if_all_children_finalized` 递归调用
- 这个调用依赖于 `sync.parent` 字段的值

**父同步如何判断是否可以收敛？**
```ruby
def all_children_finalized?
  children.incomplete.empty?  # ← 依赖 sync.children 反向关联
end
```

### 2.2 父子关系的双向依赖

```
子同步视角：
  child_sync.parent = parent_sync
  完成时：child_sync.parent&.finalize_if_all_children_finalized
           ↑
           依赖 child_sync.parent_id

父同步视角：
  parent_sync.children = [child_sync_1, child_sync_2, ...]
  检查时：parent_sync.children.incomplete.empty?
           ↑
           依赖 child_syncs.parent_id = parent_sync.id
```

**关键洞察**：`parent_id` 是收敛机制的"脐带"，一旦断裂，父子就失去联系。

---

## 三、场景一：孤立父同步（Orphan Parent Sync）

### 3.1 场景描述

子对象已有未完成 sync（来自父同步 A），此时父同步 B 试图调度该子对象。

### 3.2 时序图

```
前提条件：
  Family 有 1 个 PlaidItem（PlaidItem X）
  PlaidItem X 关联 2 个 Account（A, B）

T0: 用户触发第一次家庭同步（Family Sync A）
    └─ Family A.sync_later()
       └─ 创建 Family Sync #1 (parent: nil, status: pending)
       └─ 入队 Sidekiq

T1: Family Sync #1 Job 开始执行
    ├─ start! → syncing
    ├─ Family::Syncer.perform_sync
    │  └─ 调度子同步：
    │     └─ PlaidItem X.sync_later(parent_sync: Family Sync #1, ...)
    │        └─ 无现有 incomplete，创建 PlaidItem Sync #1
    │           └─ parent_id = Family Sync #1.id ✓
    │           └─ 入队 Sidekiq
    └─ ensure → finalize
       └─ all_children_finalized? = false（PlaidItem Sync #1 pending）
       └─ 直接 return

T2: （关键！）在 PlaidItem Sync #1 开始之前，用户又触发了第二次家庭同步
    └─ Family B.sync_later()
       └─ 检查 incomplete：Family Sync #1 是 syncing 状态，属于 incomplete
       └─ 不会创建新的 Family Sync！（因为 with_lock + incomplete.first）
       
       注意：这里 Family 级别的复用阻止了问题发生，但如果是不同的触发路径...
       
       让我们换一个更能暴露问题的场景：
```

### 3.3 更准确的问题场景：不同层级的并发调度

```
前提条件：
  PlaidItem X 已有一个 Account A（手动账户由 Family 直接调度）
  Account A 可能被单独触发同步，也可能被家庭同步调度

T0: 单独触发 Account A 同步（不通过 Family）
    └─ Account A.sync_later()  ← 无 parent_sync 参数
       └─ 创建 Account Sync #1 (parent_id: nil, status: pending)
       └─ 入队 Sidekiq
       
       Account Sync #1 数据：
         ├─ id: 1
         ├─ syncable_type: Account
         ├─ syncable_id: A.id
         ├─ parent_id: nil  ← 注意这里是 nil！
         └─ status: pending

T1: 在 Account Sync #1 开始执行前，用户触发了家庭同步
    └─ Family Sync #1 执行
       └─ Family::Syncer.perform_sync
          └─ 对 child_syncables 调用 sync_later
             └─ Account A.sync_later(parent_sync: Family Sync #1, ...)
                ├─ 检查 incomplete.first → 返回 Account Sync #1（pending）
                ├─ 复用已有 sync！
                ├─ 调用 expand_window_if_needed(...)
                │  └─ 只更新窗口日期
                │  └─ ❌ parent_id 仍然是 nil！没有更新为 Family Sync #1.id
                └─ 不创建新 sync，不重新入队

T2: Account Sync #1 Job 开始执行
    ├─ start! → syncing
    ├─ perform_sync ✓
    └─ ensure → finalize_if_all_children_finalized
       ├─ all_children_finalized? = true（无子同步）
       ├─ syncing? = true → complete!
       ├─ perform_post_sync
       └─ parent&.finalize_if_all_children_finalized
          └─ parent_id = nil → 什么都不做！
             ↑
             问题：Family Sync #1 永远不会被通知！

T3: Family Sync #1 的状态
    ├─ 它认为自己有一个"孩子"吗？
    │  └─ 没有！因为 Account Sync #1 的 parent_id 是 nil，不是 #1
    │  └─ Family Sync #1.children = []（空集合！）
    │
    ├─ all_children_finalized? = children.incomplete.empty?
    │  └─ [].incomplete.empty? = true（空集合自然"都完成了"）
    │
    └─ 等等，让我们重新审视这个问题...
```

### 3.4 修正后的分析：父同步的视角

让我们重新看 Family::Syncer 是如何调度子同步的：

```ruby
# app/models/family/syncer.rb:18-20
child_syncables.each do |syncable|
  syncable.sync_later(parent_sync: sync, window_start_date: ..., window_end_date: ...)
end
```

**关键发现**：
- 父同步**不维护**一个"应该有哪些子同步"的列表
- 父同步的 `children` 关联是通过 `syncs.parent_id = self.id` 反向查询的
- 如果子同步被复用时 `parent_id` 没有更新，**父同步根本不知道这个子同步的存在**

### 3.5 场景一的真实收敛行为

```
T0-T2 同上：
  Account Sync #1 (parent_id: nil) 被复用
  本应更新为 parent_id: Family Sync #1.id，但没有更新

T3: Family Sync #1 在 perform 后的 ensure 中调用 finalize
    ├─ all_children_finalized?
    │  └─ children.incomplete.empty?
    │     └─ children = Sync.where(parent_id: Family Sync #1.id)
    │        └─ 空集合！因为 Account Sync #1 的 parent_id 是 nil
    │     └─ [].incomplete.empty? = true
    │
    ├─ syncing? = true
    ├─ has_failed_children? = false
    ├─ complete! ← Family Sync #1 立即标记为 completed！
    │
    └─ perform_post_sync ← 转账匹配等操作立即执行
       但此时 Account Sync #1 甚至还没开始执行！
```

**问题性质变化**：
- 不是"父同步永远等待"
- 而是**"父同步过早完成"**
- 父同步的 post-sync 在子同步实际完成之前就执行了

---

## 四、场景二：悬挂子同步（Dangling Child Sync）

### 4.1 场景描述

子同步在执行过程中，一个新的父同步尝试复用它。

### 4.2 时序图

```
前提：PlaidItem X 关联 2 个 Account（A, B）

T0: Family Sync #1 开始执行
    ├─ 调度 PlaidItem X.sync_later(parent_sync: #1)
    │  └─ 创建 PlaidItem Sync #1 (parent_id: #1.id)
    └─ 调度手动账户 Account M.sync_later(parent_sync: #1)
       └─ 创建 Account Sync #M (parent_id: #1.id)
    └─ finalize: children = [#1, #M]，都 incomplete → 阻塞等待

T1: PlaidItem Sync #1 开始执行
    ├─ import_latest_plaid_data ✓
    ├─ process_accounts ✓
    └─ schedule_account_syncs
       ├─ Account A.sync_later(parent_sync: #1, ...)
       │  └─ 创建 Account Sync #A (parent_id: #1.id)
       └─ Account B.sync_later(parent_sync: #1, ...)
          └─ 创建 Account Sync #B (parent_id: #1.id)
    └─ finalize: children = [#A, #B]，都 incomplete → 阻塞等待

T2: 在 Account A/B 执行期间，用户又触发了家庭同步
    └─ Family.sync_later()
       └─ 查到 Family Sync #1 是 syncing 状态（incomplete）
       └─ 不创建新的 Family Sync！
       
       ⚠️ 这里 Family 级别的锁又保护了我们...
       
       让我们构造一个真正会出问题的场景：
```

### 4.3 真正的问题场景：跨层级并发触发

```
系统中存在多种同步触发源：
  1. 用户点击"同步所有账户" → Family.sync_later
  2. Plaid Webhook → PlaidItem.sync_later（无 parent）
  3. 用户修改某个交易 → Entry 回调 → Account.sync_later（无 parent）
  4. 数据缓存清理 → DataCacheClearJob → Family.sync_later
  5. 创建交易转账 → Transfer::Creator → 两个 Account.sync_later

T0: Plaid Webhook 到达（独立触发，无 parent）
    └─ PlaidItem X.sync_later()  ← 无 parent_sync 参数
       └─ 创建 PlaidItem Sync #1 (parent_id: nil, status: pending)
       └─ 入队

T1: PlaidItem Sync #1 Job 开始执行
    ├─ start! → syncing
    ├─ import_latest_plaid_data ✓
    ├─ process_accounts ✓
    └─ schedule_account_syncs
       ├─ Account A.sync_later(parent_sync: #1, ...)
       │  └─ 创建 Account Sync #A (parent_id: #1.id)
       └─ Account B.sync_later(parent_sync: #1, ...)
          └─ 创建 Account Sync #B (parent_id: #1.id)
    └─ finalize: children = [#A, #B] 都 pending → 阻塞等待

T2: 在 Account A/B 执行期间，用户点击"同步所有账户"
    └─ Family Sync #1 创建并执行
       └─ Family::Syncer.perform_sync
          └─ 调度 PlaidItem X.sync_later(parent_sync: Family #1, ...)
             ├─ 查到 incomplete.first = PlaidItem Sync #1（syncing 状态）
             ├─ 复用已有 sync
             ├─ expand_window_if_needed(...)
             │  └─ return unless pending? → 直接返回！
             │  └─ 状态是 syncing，连窗口都不更新
             │  └─ ❌ parent_id 当然也不会更新（仍为 nil）
             └─ 不创建新 sync

T3: Account Sync #A 完成
    ├─ finalize → 通知 parent (PlaidItem Sync #1)
    └─ PlaidItem Sync #1 检查：还有 #B 在运行 → 继续阻塞

T4: Account Sync #B 完成
    ├─ finalize → 通知 parent (PlaidItem Sync #1)
    └─ PlaidItem Sync #1：all_children_finalized? = true
       ├─ complete!
       ├─ perform_post_sync
       └─ parent&.finalize_if_all_children_finalized
          └─ parent_id = nil → 不通知任何人！
             ↑
             问题：Family Sync #1 永远不会知道 PlaidItem 已经完成！

T5: Family Sync #1 的最终状态
    ├─ children = Sync.where(parent_id: Family #1.id)
    │  └─ 可能包括手动账户的 sync，但 PlaidItem Sync #1 不在其中
    │
    ├─ 如果 Family 只有 PlaidItem 没有手动账户：
    │  └─ children = [] → all_children_finalized? = true
    │  └─ 在 T2 之后不久就 complete! 了（过早完成）
    │
    └─ 如果有手动账户：
       └─ 等手动账户完成后就 complete! 了
       └─ 但 PlaidItem 的账户同步可能还在进行中
```

---

## 五、场景三：父同步竞争（Parent Race Condition）

### 5.1 场景描述

两个父同步 A 和 B 先后尝试调度同一个子同步，子同步被第一个父同步"捕获"。

### 5.2 时序图

```
T0: Family Sync #1 执行
    └─ 调度 Account A.sync_later(parent_sync: #1, ...)
       └─ 创建 Account Sync #X (parent_id: #1.id)
       └─ 状态：pending

T1: Account Sync #X 还没开始，Family Sync #2 也开始执行
    （Family Sync #2 能创建成功吗？要看 Family #1 的状态）
    
    情况 A：Family Sync #1 还在 syncing
      └─ Family.sync_later() 查到 incomplete → 复用 #1，不创建 #2
      └─ 问题不会出现（被锁保护了）
    
    情况 B：Family Sync #1 已完成，但 Account Sync #X 还在运行
      这才是问题场景：

T1（修正）: Family Sync #1 因某种原因已完成
    （例如：它调度的子同步被复用时 parent_id 没更新，导致它认为没有子同步）
    └─ Family Sync #1.completed_at = 10:00
    └─ 但 Account Sync #X 仍在运行（status: syncing）

T2: 用户触发新的家庭同步
    └─ Family.sync_later()
       └─ 查到 incomplete = [Family Sync #1 已完成，不算]
       └─ 创建 Family Sync #2
       
T3: Family Sync #2 执行
    └─ 调度 Account A.sync_later(parent_sync: #2, ...)
       ├─ 查到 incomplete.first = Account Sync #X（syncing 状态）
       ├─ 复用已有 sync
       ├─ expand_window_if_needed → syncing 状态直接返回
       ├─ ❌ parent_id 仍为 #1.id（不是 #2.id！）
       └─ 不创建新 sync

T4: Account Sync #X 完成
    └─ parent&.finalize_if_all_children_finalized
       └─ parent = Family Sync #1（已 completed，不会有任何变化）
       └─ ❌ Family Sync #2 完全不知道这件事！

T5: Family Sync #2 的结局
    ├─ 如果它没有其他子同步：
    │  └─ children = [] → 在 T3 之后就立即 complete! 了
    │
    └─ 如果有其他子同步：
       └─ 等那些完成后 complete!
       └─ 但 Account Sync #X 被遗漏了
```

---

## 六、风险结论矩阵

### 6.1 三种异常场景的风险评级

| 场景 | 触发条件 | 概率 | 影响 | 严重程度 |
|------|---------|------|------|---------|
| **孤立父同步** | 子对象被独立触发同步（无 parent）后，又被父同步调度 | 中 | 父同步过早完成，post-sync 在子同步实际完成前执行 | **高** |
| **悬挂子同步** | 子同步正在执行中（syncing），新父同步尝试调度它 | 中 | 子同步完成后不通知新父同步；父同步可能过早完成 | **高** |
| **父同步竞争** | 前一个父同步已完成，但它的子同步仍在运行 | 低 | 新父同步"认领"不到正在运行的子同步，可能过早完成 | **中** |

### 6.2 业务影响分析

#### 影响一：post-sync 执行时机错误

**转账自动匹配 (`auto_match_transfers!`)**：
- 本应在**所有**账户同步完成后执行
- 但父同步过早完成 → 在部分账户同步完成前就执行
- 可能导致：
  - 跨账户转账未被识别（因为另一账户的数据还没同步完）
  - 需要等待下一次同步才能正确匹配

#### 影响二：UI 状态不一致

**同步状态展示**：
- UI 通过 `syncing?` 判断是否在同步中
- `syncing? = syncs.visible.any?`
- `visible = incomplete AND created_at > 5.minutes.ago`

**可能出现的情况**：
- Family Sync 显示为 `completed`（绿色对勾）
- 但某个 Account Sync 实际上还在 `syncing`
- 用户看到"同步完成"，但数据实际上还在处理中

#### 影响三：`latest_sync_completed_at` 过早更新

```ruby
# app/models/sync.rb:176-178
def handle_completion_transition
  family.touch(:latest_sync_completed_at)
end
```

- 这个时间戳用于缓存失效（`build_cache_key`）
- 过早更新 → 缓存认为"数据已最新"
- 但实际上子同步还在写入新数据

---

## 七、现有测试覆盖分析

### 7.1 已覆盖的场景

```ruby
# test/models/sync_test.rb

# ✅ 已覆盖：正常嵌套同步的收敛
test "can run nested syncs that alert the parent when complete"
  └─ 预先创建好 parent 关系，测试正常收敛

# ✅ 已覆盖：失败传播
test "failures propagate up the chain"
  └─ 预先创建好 parent 关系，测试失败传播

# ✅ 已覆盖：父先失败子后成功
test "parent failure should not change status if child succeeds"
  └─ 预先创建好 parent 关系
```

### 7.2 未覆盖的场景（风险点）

| 未覆盖场景 | 对应本文第几章 |
|-----------|---------------|
| `sync_later` 复用时 `parent_id` 是否更新 | 第三、四、五章 |
| 子对象已有无-parent 同步时，父同步调度行为 | 第三章 |
| 子同步在 syncing 状态时，新父同步尝试复用 | 第四章 |
| `expand_window_if_needed` 不更新 parent 的副作用 | 贯穿全文 |

### 7.3 测试缺口验证

现有测试都是这样构造数据的：
```ruby
family_sync = Sync.create!(syncable: family)
plaid_item_sync = Sync.create!(syncable: plaid_item, parent: family_sync)
account_sync = Sync.create!(syncable: account, parent: plaid_item_sync)
```

**问题**：直接用 `Sync.create!` 建立 parent 关系，绕过了 `sync_later` 的复用逻辑！

**真实场景 vs 测试场景**：

| 维度 | 真实场景 | 现有测试场景 |
|------|---------|-------------|
| 父子关系建立时机 | 通过 `sync_later(parent_sync: ...)` 动态建立 | 通过 `Sync.create!(parent: ...)` 静态建立 |
| 是否可能复用已有 sync | 是 | 否（每次都是全新创建） |
| 是否可能出现 parent_id 不更新 | 是 | 否（直接赋值 parent） |

---

## 八、问题根因与修复方向

### 8.1 根因分析

```ruby
# 当前逻辑的问题
def sync_later(parent_sync: nil, ...)
  if sync = incomplete.first
    sync.expand_window_if_needed(...)  # 只扩窗口
    # 缺失：如果传入了新的 parent_sync，是否应该更新？
  else
    create!(parent: parent_sync, ...)  # 只有这里设置 parent
  end
end
```

**设计意图推测**：
- 原设计假设：同一时间一个 syncable 只属于一个"同步上下文"
- 但实际上：存在多种触发源，可能形成多个独立的同步树
- 窗口扩展解决了"数据范围"的合并问题
- 但**没有解决"父子关系"的合并问题**

### 8.2 修复方向探讨

**选项 A：复用时更新 parent_id**
```ruby
if sync
  sync.expand_window_if_needed(...)
  # 新增：如果有新的 parent_sync，更新关联
  if parent_sync && sync.parent != parent_sync
    sync.update!(parent: parent_sync)
  end
end
```
- 问题：如果子同步已属于另一个父同步，"偷过来"是否正确？
- 那个旧的父同步怎么办？

**选项 B：允许一个子同步有多个 parent（多对多）**
- 需要重构数据模型
- 复杂度高，但语义最准确

**选项 C：如果已有 sync 属于其他 parent，创建新的子 sync**
```ruby
if sync && (sync.parent.nil? || sync.parent == parent_sync)
  # 可以复用：无 parent 或 parent 相同
  sync.expand_window_if_needed(...)
  sync.update!(parent: parent_sync) if parent_sync
else
  # 不能复用：已属于其他 parent，创建新的
  create!(parent: parent_sync, ...)
end
```
- 可能导致同一 syncable 有多个 parallel sync
- 需要确保它们不会互相干扰

**选项 D：保持现状，明确文档化限制**
- 接受这种边缘情况的存在
- 在 `sync_later` 注释中说明："复用已有 sync 时不会更新 parent"
- 风险由上层调用者规避

---

## 九、总结

### 9.1 核心发现

1. **`sync_later` 复用已有 sync 时只做窗口扩展，不更新 `parent_id`**
2. **`expand_window_if_needed` 还额外限制了只在 `pending?` 状态工作**
3. **收敛机制完全依赖 `parent_id` 关联，一旦断裂，父子同步失去联系**

### 9.2 可能出现的问题

| 问题 | 表现 |
|------|------|
| 父同步过早完成 | children 为空 → `all_children_finalized?` = true → 立即 complete! |
| post-sync 提前执行 | 转账匹配在所有数据就绪前执行 |
| 新父同步等不到通知 | 子同步完成时通知的是旧 parent（或 nil） |
| 缓存过早失效 | `latest_sync_completed_at` 被过早更新 |

### 9.3 建议行动

1. **短期**：补充测试用例验证当前行为，确认这是"设计如此"还是"bug"
2. **中期**：评估修复方案的成本和风险
3. **长期**：考虑重新设计父子同步关系模型，支持多 parent 或更清晰的归属规则

---

## 十、代码位置索引

| 关注点 | 文件 | 行号 |
|--------|------|------|
| `sync_later` 复用逻辑 | `app/models/concerns/syncable.rb` | 14-35 |
| `expand_window_if_needed` 限制 | `app/models/sync.rb` | 107-127 |
| 收敛时通知 parent | `app/models/sync.rb` | 103 |
| 父同步检查子同步状态 | `app/models/sync.rb` | 138-140 |
| Family 调度子同步 | `app/models/family/syncer.rb` | 17-20 |
| PlaidItem 调度账户同步 | `app/models/plaid_item/syncer.rb` | 16-20 |
| 现有嵌套同步测试 | `test/models/sync_test.rb` | 46-88 |
| 现有失败传播测试 | `test/models/sync_test.rb` | 90-134 |
