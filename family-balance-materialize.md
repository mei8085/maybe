# 家庭账户余额物化流程报告

## 目录

1. [概述](#1-概述)
2. [核心流程总览](#2-核心流程总览)
3. [流水触发同步机制](#3-流水触发同步机制)
4. [后台 Job 调度系统](#4-后台-job-调度系统)
5. [同步失败与子同步失败状态收敛及广播行为](#5-同步失败与子同步失败状态收敛及广播行为)
6. [并发提交时同步窗口扩展机制](#6-并发提交时同步窗口扩展机制)
7. [stale 标记后的重跑路径](#7-stale-标记后的重跑路径)
8. [余额计算与物化核心](#8-余额计算与物化核心)
9. [数据物化到 balances 表](#9-数据物化到-balances-表)
10. [同步完成与 UI 广播](#10-同步完成与-ui-广播)
11. [图表数据消费](#11-图表数据消费)
12. [完整时序图](#12-完整时序图)
13. [关键组件速查表](#13-关键组件速查表)
14. [设计要点总结](#14-设计要点总结)

---

## 1. 概述

本报告详细说明家庭账户在新流水（Entry）进来之后，余额是如何重新计算并物化到列表与图表的完整流程。这个过程涉及多个后台 Job 与计算器的协作，最终实现数据下沉到 UI 层。

## 2. 核心流程总览

### 2.1 流程概览

```
新流水创建/更新
      ↓
Entry 触发 sync_account_later
      ↓
SyncJob（后台高优先级队列）
      ↓
Account::Syncer.perform_sync
      ↓
Balance::Materializer.materialize_balances
      ↓
Balance::Calculator（正向/反向）计算
      ↓
持久化到 balances 表
      ↓
Sync 完成事件广播
      ↓
UI 列表与图表更新
```

## 3. 流水触发同步机制

### 3.1 流水创建与更新入口

新流水的创建和更新主要通过以下几个入口触发：

1. **手动创建交易**：`TransactionsController#create` (app/controllers/transactions_controller.rb:56-74)
2. **手动创建交易**：`TradesController#create`
3. **API 创建交易**：`Api::V1::TransactionsController`
4. **Plaid 同步导入**：通过 ImportJob 和相关处理流程

### 3.2 触发同步的关键代码

在 `app/models/entry.rb:46-49` 中定义了同步触发方法：

```ruby
def sync_account_later
  sync_start_date = [ date_previously_was, date ].compact.min unless destroyed?
  account.sync_later(window_start_date: sync_start_date)
end
```

**关键点**：
- 如果流水被修改，同步窗口从最早的日期开始
- 这确保了即使流水日期被修改，所有受影响日期的余额都会重新计算

### 3.3 控制器中的调用

在 `app/controllers/transactions_controller.rb:60-63`：

```ruby
if @entry.save
  @entry.sync_account_later  # 触发后台同步
  @entry.lock_saved_attributes!
  # ...
end
```

同样在 `update` 操作中（第88行）也会调用 `sync_account_later`。

## 4. 后台 Job 调度系统

### 4.1 Syncable 模块

`app/models/concerns/syncable.rb` 是所有可同步对象的核心模块：

```ruby
def sync_later(parent_sync: nil, window_start_date: nil, window_end_date: nil)
  Sync.transaction do
    with_lock do
      sync = self.syncs.incomplete.first

      if sync
        # 如果已有待处理的同步，扩展同步窗口
        sync.expand_window_if_needed(window_start_date, window_end_date)
      else
        # 创建新的同步记录并排入队列
        sync = self.syncs.create!(...)
        SyncJob.perform_later(sync)
      end

      sync
    end
  end
end
```

**设计亮点**：
- **窗口扩展**：如果已有同步在等待执行，不会重复创建 Job，而是扩展现有同步的时间窗口
- **原子性**：使用事务和锁确保同步操作的原子性
- **高优先级队列**：SyncJob 使用 `:high_priority` 队列

### 4.2 SyncJob

`app/jobs/sync_job.rb` 是执行同步的主 Job：

```ruby
class SyncJob < ApplicationJob
  queue_as :high_priority

  def perform(sync)
    sync.perform
  end
end
```

### 4.3 Sync 状态机

`app/models/sync.rb` 实现了完整的同步状态管理：

**状态流转**：
```
pending → syncing → completed
              ↓
            failed
              ↓
            stale (超时24小时)
```

**关键状态定义** (`app/models/sync.rb:18-20`)：
```ruby
scope :ordered, -> { order(created_at: :desc) }
scope :incomplete, -> { where("syncs.status IN (?)", %w[pending syncing]) }
scope :visible, -> { incomplete.where("syncs.created_at > ?", VISIBLE_FOR.ago) }
```

**重要**：`incomplete` 作用域**仅包含** `pending` 和 `syncing` 状态，**不包含** `failed` 和 `stale` 状态。这意味着：
- `failed` 和 `stale` 的同步记录不会被 `syncs.incomplete.first` 找到
- 后续的 `sync_later` 调用会创建**全新**的同步记录

**关键方法**：
- `perform` (第60-80行)：执行同步主流程
- `finalize_if_all_children_finalized` (第83-104行)：处理父子同步关系
- `perform_post_sync` (第142-149行)：同步完成后的后置处理

## 5. 同步失败与子同步失败状态收敛及广播行为

### 5.1 父子同步层级结构

家庭账户同步采用**混合树形结构**，Family 同时并行调度两条路径：

```
                    Family（家庭级同步）
                          │
           ┌──────────────┼──────────────┐
           ▼                             ▼
   PlaidItem（银行连接级）       Manual Account（手动账户级）
           │
           ▼
   Linked Account（链接账户级）
```

**代码证据** (`app/models/family/syncer.rb:17-30`)：

```ruby
def perform_sync(sync)
  # Schedule child syncs
  child_syncables.each do |syncable|
    syncable.sync_later(parent_sync: sync, 
                        window_start_date: sync.window_start_date, 
                        window_end_date: sync.window_end_date)
  end
end

private
  def child_syncables
    family.plaid_items + family.accounts.manual  # ← 两条路径并行
  end
```

**PlaidItem 再调度其下的链接账户** (`app/models/plaid_item/syncer.rb:8-21`)：

```ruby
def perform_sync(sync)
  # Loads item metadata, accounts, transactions, and other data to our DB
  plaid_item.import_latest_plaid_data

  # Processes the raw Plaid data and updates internal domain objects
  plaid_item.process_accounts

  # All data is synced, so we can now run an account sync to calculate historical balances and more
  plaid_item.schedule_account_syncs(
    parent_sync: sync,
    window_start_date: sync.window_start_date,
    window_end_date: sync.window_end_date
  )
end
```

**层级说明**：
1. **第一级（并行）**：Family 同步同时调度 `plaid_items` 和 `accounts.manual`
2. **第二级**：PlaidItem 同步再调度其下的链接账户（Linked Account）
3. **状态收敛**：所有子同步完成后才收敛到 Family 同步状态

### 5.2 同步主流程的异常处理

`app/models/sync.rb:60-80`：

```ruby
def perform
  Rails.logger.tagged("Sync", id, syncable_type, syncable_id) do
    # This can happen on server restarts or if Sidekiq enqueues a duplicate job
    unless may_start?
      Rails.logger.warn("Sync #{id} is not in a valid state (#{aasm.from_state}) to start.  Skipping sync.")
      return
    end

    start!

    begin
      syncable.perform_sync(self)
    rescue => e
      fail!
      update(error: e.message)
      report_error(e)
    ensure
      finalize_if_all_children_finalized
    end
  end
end
```

**异常处理要点**：
1. **状态检查**：`may_start?` 确保只有 `pending` 状态的同步能开始执行
2. **异常捕获**：`begin-rescue-ensure` 结构确保即使同步失败也会执行收尾
3. **失败标记**：`fail!` 立即标记同步为 `failed` 状态
4. **错误记录**：`update(error: e.message)` 保存错误信息
5. **保证收尾**：`ensure` 块确保 `finalize_if_all_children_finalized` 总是执行

### 5.3 父子同步状态收敛机制

`app/models/sync.rb:82-104`：

```ruby
def finalize_if_all_children_finalized
  Sync.transaction do
    lock!

    # If this is the "parent" and there are still children running, don't finalize.
    return unless all_children_finalized?

    if syncing?
      if has_failed_children?
        fail!
      else
        complete!
      end
    end

    # If we make it here, the sync is finalized.  Run post-sync, regardless of failure/success.
    perform_post_sync
  end

  # If this sync has a parent, try to finalize it so the child status propagates up the chain.
  parent&.finalize_if_all_children_finalized
end
```

**关键辅助方法** (`app/models/sync.rb:134-140`)：

```ruby
def has_failed_children?
  children.failed.any?
end

def all_children_finalized?
  children.incomplete.empty?
end
```

**状态收敛规则**：

| 条件 | 父同步状态 | 说明 |
|------|-----------|------|
| `all_children_finalized?` 为 false | 保持 `syncing` | 仍有子同步在执行，等待 |
| `all_children_finalized?` 为 true 且 `has_failed_children?` 为 true | `fail!` | 任一子同步失败，父同步也失败 |
| `all_children_finalized?` 为 true 且 `has_failed_children?` 为 false | `complete!` | 所有子同步成功，父同步成功 |

**递归传播**：`parent&.finalize_if_all_children_finalized` 确保状态向上传播整个层级。

### 5.4 测试场景验证

根据 `test/models/sync_test.rb` 的三个关键测试：

**场景1：全链路成功** (`test "can run nested syncs that alert the parent when complete"`)：
```
Family:syncing ──→ PlaidItem:syncing ──→ Account:syncing
     ↓                    ↓                    ↓
Family:completed ←─ PlaidItem:completed ←─ Account:completed
```

**场景2：子同步失败向上传播** (`test "failures propagate up the chain"`)：
```
Family:syncing ──→ PlaidItem:syncing ──→ Account:fail!
     ↓                    ↓                    ↓
Family:failed  ←── PlaidItem:failed  ←── Account:failed
```

**场景3：父已失败，子成功不改变父状态** (`test "parent failure should not change status if child succeeds"`)：
```
Family:fail! ──→ PlaidItem:fail! ──→ Account:complete!
     ↓                 ↓                    ↓
Family:failed    PlaidItem:failed    Account:completed
(状态不变)      (状态不变)
```

### 5.5 失败后的广播行为

`app/models/sync.rb:142-149`：

```ruby
def perform_post_sync
  Rails.logger.info("Performing post-sync for #{syncable_type} (#{syncable.id})")
  syncable.perform_post_sync
  syncable.broadcast_sync_complete
rescue => e
  Rails.logger.error("Error performing post-sync for #{syncable_type} (#{syncable.id}): #{e.message}")
  report_error(e)
end
```

**关键设计**：
1. **无论成功或失败都执行**：`perform_post_sync` 在 `finalize_if_all_children_finalized` 中无条件调用
2. **广播总是触发**：`broadcast_sync_complete` 在失败后仍然执行
3. **异常隔离**：post-sync 内部异常被捕获，不影响状态机流程

**测试验证** (`test/models/sync_test.rb:121-127`)：
```ruby
# 即使 Account 同步失败，仍然会执行广播：
Account.any_instance.expects(:perform_post_sync).once
PlaidItem.any_instance.expects(:perform_post_sync).once
Family.any_instance.expects(:perform_post_sync).once

Account.any_instance.expects(:broadcast_sync_complete).once
PlaidItem.any_instance.expects(:broadcast_sync_complete).once
Family.any_instance.expects(:broadcast_sync_complete).once
```

**广播层级**：

| 层级 | 广播组件 | 广播内容 |
|------|---------|---------|
| Account | `Account::SyncCompleteEvent` | 账户列表行、侧边栏分组、账户详情页 |
| PlaidItem | `PlaidItem::SyncCompleteEvent` | 触发各账户广播 → 最终触发家庭广播 |
| Family | `Family::SyncCompleteEvent` | 资产负债表、净值图表 |

**失败场景下的广播策略**：
- 即使同步失败，UI 仍然会被刷新，让用户看到最新的可用数据
- 失败信息通过 `sync.error` 字段存储，可在 UI 中展示
- `sync_error` 方法 (`app/models/concerns/syncable.rb:49-51`) 用于获取同步错误：
  ```ruby
  def sync_error
    latest_sync&.error || latest_sync&.children&.map(&:error)&.compact&.first
  end
  ```

## 6. 并发提交时同步窗口扩展机制

### 6.1 并发控制的原子性保障

`app/models/concerns/syncable.rb:14-35`：

```ruby
def sync_later(parent_sync: nil, window_start_date: nil, window_end_date: nil)
  Sync.transaction do
    with_lock do
      sync = self.syncs.incomplete.first

      if sync
        Rails.logger.info("There is an existing sync, expanding window if needed (#{sync.id})")
        sync.expand_window_if_needed(window_start_date, window_end_date)
      else
        sync = self.syncs.create!(
          parent: parent_sync,
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

**并发控制三重保障**：
1. **数据库事务** (`Sync.transaction do`)：整个操作在一个事务中
2. **行级锁** (`with_lock do`)：对 syncable 记录加锁，防止并发创建
3. **状态检查** (`self.syncs.incomplete.first`)：仅查找 `pending` 或 `syncing` 状态

**并发场景示意**：

```
请求A: 创建流水1 (日期: 2026-05-01)
请求B: 创建流水2 (日期: 2026-05-10)
请求C: 创建流水3 (日期: 2026-05-05)

时间线：
─────────────────────────────────────────────────→
  │         │         │
  ▼         ▼         ▼
请求A     请求B     请求C
开始      开始      开始
  │         │         │
  │        with_lock  │
  │         │         │
  │      查找incomplete
  │      (nil)        │
  │      创建Sync1    │
  │      window: [2026-05-10]
  │         │         │
  │      释放锁       │
  ▼         │         ▼
with_lock              with_lock
  │                    查找incomplete
  │                    (找到Sync1: pending)
  │                    expand_window_if_needed
  │                    window: [2026-05-05, 2026-05-10]
  │                    释放锁
查找incomplete
(找到Sync1: pending)
expand_window_if_needed
window: [2026-05-01, 2026-05-10]
释放锁
  │
  ▼
最终: 只有一个SyncJob被调度
      window覆盖所有三个日期 [2026-05-01 ~ 2026-05-10]
```

### 6.2 窗口扩展算法

`app/models/sync.rb:106-127`：

```ruby
def expand_window_if_needed(new_window_start_date, new_window_end_date)
  return unless pending?
  return if self.window_start_date.nil? && self.window_end_date.nil? # already as wide as possible

  earliest_start_date = if self.window_start_date && new_window_start_date
    [ self.window_start_date, new_window_start_date ].min
  else
    nil
  end

  latest_end_date = if self.window_end_date && new_window_end_date
    [ self.window_end_date, new_window_end_date ].max
  else
    nil
  end

  update(
    window_start_date: earliest_start_date,
    window_end_date: latest_end_date
  )
end
```

**扩展规则**：

| 条件 | 行为 |
|------|------|
| 同步状态不是 `pending` | **直接返回**，不扩展（正在执行的同步不修改窗口） |
| `window_start_date` 和 `window_end_date` 都为 `nil` | **直接返回**（`nil` 表示全量同步，已最宽） |
| 新窗口开始日期更早 | `window_start_date = min(old_start, new_start)` |
| 新窗口结束日期更晚 | `window_end_date = max(old_end, new_end)` |

**窗口含义**：
- `window_start_date = nil, window_end_date = nil`：**全量同步**（重新计算所有历史）
- 有具体日期：**增量同步**（仅计算该窗口内的变化）

### 6.3 避免漏算的关键设计

**1. 窗口扩展仅在 pending 状态**：
```ruby
return unless pending?
```
- 如果同步已经 `syncing`，不再扩展窗口
- 避免正在执行的同步中途改变计算范围

**2. 完整时间覆盖**：
- `Entry#sync_account_later` 传递 `window_start_date` 为最早受影响日期
- 扩展算法取 `min`/`max` 确保所有日期都被覆盖

**3. 多次触发只调度一次**：
```ruby
sync = self.syncs.incomplete.first

if sync
  # 已有incomplete同步，仅扩展窗口，不创建新Job
  sync.expand_window_if_needed(window_start_date, window_end_date)
else
  # 没有incomplete同步，创建新记录并调度
  sync = self.syncs.create!(...)
  SyncJob.perform_later(sync)
end
```

**测试验证** (`test/models/sync_test.rb:203-221`)：
```ruby
test "expand_window_if_needed widens start and end dates on a pending sync" do
  initial_start = 1.day.ago.to_date
  initial_end   = 1.day.ago.to_date

  sync = Sync.create!(
    syncable: accounts(:depository),
    window_start_date: initial_start,
    window_end_date: initial_end
  )

  new_start = 5.days.ago.to_date
  new_end   = Date.current

  sync.expand_window_if_needed(new_start, new_end)

  assert_equal new_start, sync.window_start_date  # 取更早的
  assert_equal new_end,   sync.window_end_date    # 取更晚的
end
```

### 6.4 特殊场景：流水日期修改

`app/models/entry.rb:46-49`：

```ruby
def sync_account_later
  sync_start_date = [ date_previously_was, date ].compact.min unless destroyed?
  account.sync_later(window_start_date: sync_start_date)
end
```

**设计意图**：
- 如果流水日期从 `2026-05-10` 改为 `2026-05-01`
- `date_previously_was = 2026-05-10`，`date = 2026-05-01`
- `min = 2026-05-01`，确保从最早的日期开始重算

**测试验证** (`test/models/account/entry_test.rb:29-43`)：
```ruby
test "triggers sync with correct start date when transaction date changed to prior" do
  prior_date = @entry.date - 1
  @entry.update! date: prior_date
  
  @entry.account.expects(:sync_later).with(window_start_date: prior_date)
  @entry.sync_account_later
end

test "triggers sync with correct start date when transaction date changed to later" do
  prior_date = @entry.date
  @entry.update! date: @entry.date + 1
  
  @entry.account.expects(:sync_later).with(window_start_date: prior_date)
  @entry.sync_account_later
end
```

**关键点**：无论日期改早还是改晚，`sync_start_date` 都取 `min(旧日期, 新日期)`，确保覆盖所有受影响的日期。

## 7. stale 标记后的重跑路径

### 7.1 stale 状态的产生

`app/models/sync.rb:2-4`：

```ruby
# We run a cron that marks any syncs that have not been resolved in 24 hours as "stale"
# Syncs often become stale when new code is deployed and the worker restarts
STALE_AFTER = 24.hours
```

**stale 标记条件**：
- 同步状态为 `pending` 或 `syncing`
- `created_at` 超过 24 小时

**标记执行** (`app/models/sync.rb:54-58`)：

```ruby
class << self
  def clean
    incomplete.where("syncs.created_at < ?", STALE_AFTER.ago).find_each(&:mark_stale!)
  end
end
```

**定时调度** (`config/schedule.yml:10-14`)：

```yaml
clean_syncs:
  cron: "0 * * * *" # every hour
  class: "SyncCleanerJob"
  queue: "scheduled"
  description: "Cleans up stale syncs"
```

**触发场景**：
1. Sidekiq worker 重启（部署时常见）
2. 同步执行过程中进程崩溃
3. 同步执行时间异常长（超过24小时）

### 7.2 stale 状态的关键特性

**1. incomplete 作用域不包含 stale** (`app/models/sync.rb:19`)：
```ruby
scope :incomplete, -> { where("syncs.status IN (?)", %w[pending syncing]) }
```

**影响**：
- `syncs.incomplete.first` 不会返回 stale 同步
- 后续 `sync_later` 调用会创建**全新**的同步记录

**2. 状态机转换** (`app/models/sync.rb:48-51`)：
```ruby
# Marks a sync that never completed within the expected time window
event :mark_stale do
  transitions from: %i[pending syncing], to: :stale
end
```

**3. 无法从 stale 重新开始**：
- 没有定义从 `stale` 到其他状态的转换事件
- stale 同步记录仅作为历史记录保留

### 7.3 重跑路径：新流水触发

**重跑机制完全依赖新的 `sync_later` 调用**：

```
现有场景：
─────────────────────────────────────────────────
SyncA: pending → syncing → (24小时后) → stale
                        (worker重启中断)

用户创建新流水：
─────────────────────────────────────────────────
entry.save! → entry.sync_account_later
                    ↓
              account.sync_later
                    ↓
              syncs.incomplete.first 
              (返回 nil，因为 stale 不在 incomplete)
                    ↓
              创建 SyncB: pending
              SyncJob.perform_later(SyncB)
                    ↓
              SyncB 正常执行 → completed
```

**关键代码路径**：

```ruby
# app/models/concerns/syncable.rb:14-35
def sync_later(parent_sync: nil, window_start_date: nil, window_end_date: nil)
  Sync.transaction do
    with_lock do
      sync = self.syncs.incomplete.first  # ← 不会找到 stale

      if sync
        # 只有 pending/syncing 才会走这里
        sync.expand_window_if_needed(...)
      else
        # stale 存在时会走这里，创建新同步
        sync = self.syncs.create!(...)
        SyncJob.perform_later(sync)
      end
    end
  end
end
```

### 7.4 测试验证

`test/models/sync_test.rb:182-201`：

```ruby
test "clean marks stale incomplete rows" do
  stale_pending = Sync.create!(
    syncable: accounts(:depository),
    status: :pending,
    created_at: 25.hours.ago  # 超过24小时
  )

  stale_syncing = Sync.create!(
    syncable: accounts(:depository),
    status: :syncing,
    created_at: 25.hours.ago,
    pending_at: 24.hours.ago,
    syncing_at: 23.hours.ago
  )

  Sync.clean

  assert_equal "stale", stale_pending.reload.status
  assert_equal "stale", stale_syncing.reload.status
end
```

### 7.5 设计权衡分析

**优点**：
1. **自动恢复**：不需要人工干预，新的操作自然触发重跑
2. **避免无限重试**：stale 记录不会被自动重试，防止死循环
3. **历史可追溯**：stale 记录保留，可用于问题排查

**潜在风险**：
1. **数据延迟**：如果没有新流水触发，余额可能长时间不更新
2. **窗口丢失**：stale 同步的 window 信息不会被新同步继承
3. **需要外部触发**：完全依赖新的 `sync_later` 调用

**缓解措施**：
- 定时同步（Plaid 自动刷新）
- 用户手动触发同步（UI 中的刷新按钮）
- 下次任何流水操作都会自然触发

### 7.6 完整状态流转图

```
                    ┌─────────────────────────────────┐
                    │                                 │
                    ▼                                 │
    entry.save → sync_later → 查找incomplete → 创建pending
                    │                                 │
                    │  找到incomplete?                 │
                    │    │                             │
                    │    ├─ Yes → 扩展窗口             │
                    │    │                             │
                    │    └─ No  → 创建新 Sync          │
                    │                                 │
                    ▼                                 │
               SyncJob执行                            │
                    │                                 │
                    ├─ perform_sync 成功               │
                    │      │                          │
                    │      └─→ completed              │
                    │                                 │
                    ├─ perform_sync 异常               │
                    │      │                          │
                    │      └─→ failed                 │
                    │                                 │
                    └─ 24小时未完成                    │
                           │                          │
                           └─ Sync.clean → stale ────┘
                                              (不会被incomplete找到)
                                              (新sync_later创建新记录)
```

## 8. 余额计算与物化核心

### 8.1 Account::Syncer

`app/models/account/syncer.rb` 是账户级别的同步器：

```ruby
def perform_sync(sync)
  Rails.logger.info("Processing balances (#{account.linked? ? 'reverse' : 'forward'})")
  import_market_data      # 导入市场数据（汇率、证券价格）
  materialize_balances    # 核心：余额物化
end

def perform_post_sync
  account.family.auto_match_transfers!  # 自动匹配转账
end
```

### 8.2 Balance::Materializer

`app/models/balance/materializer.rb` 是余额物化的核心组件：

```ruby
def materialize_balances
  Balance.transaction do
    materialize_holdings   # 第一步：物化持仓
    calculate_balances     # 第二步：计算余额
    persist_balances       # 第三步：持久化余额
    purge_stale_balances   # 第四步：清理过期余额

    if strategy == :forward
      update_account_info  # 第五步：更新账户信息（仅正向计算）
    end
  end
end
```

**关键步骤详解**：

#### 步骤1：物化持仓 (`materialize_holdings`)

```ruby
@holdings = Holding::Materializer.new(account, strategy: strategy).materialize_holdings
```

持仓物化过程与余额类似：
- 先计算持仓数据
- 持久化到 `holdings` 表
- 清理过期持仓

#### 步骤2：计算余额 (`calculate_balances`)

```ruby
@balances = calculator.calculate
```

根据账户类型选择计算器：
- **手动账户**：`Balance::ForwardCalculator`（正向计算）
- **Plaid 连接账户**：`Balance::ReverseCalculator`（反向计算）

#### 步骤3：持久化余额 (`persist_balances`)

```ruby
account.balances.upsert_all(
  @balances.map { |b| b.attributes.slice(...).merge("updated_at" => current_time) },
  unique_by: %i[account_id date currency]
)
```

使用 `upsert_all` 批量更新，确保高效。

#### 步骤4：清理过期余额 (`purge_stale_balances`)

删除计算日期范围之外的余额记录。

#### 步骤5：更新账户信息 (`update_account_info`)

仅在正向计算时更新 `account.balance` 和 `account.cash_balance` 字段。

### 8.3 两种计算策略

#### 8.3.1 正向计算 (Forward Calculator)

适用于**手动账户**，从期初余额开始向前计算：

```ruby
# app/models/balance/forward_calculator.rb:2-52
def calculate
  start_cash_balance = derive_cash_balance_on_date_from_total(
    total_balance: account.opening_anchor_balance,
    date: account.opening_anchor_date
  )
  start_non_cash_balance = account.opening_anchor_balance - start_cash_balance

  calc_start_date.upto(calc_end_date).map do |date|
    # 逐天计算...
  end
end
```

**计算日期范围**：
- 开始日期：`opening_anchor_date`（期初锚点日期）
- 结束日期：`[ last_entry_date, last_holding_date ].compact.max || Date.current`

**每日计算逻辑**：
1. 获取当日估值（valuation）或根据流水推导
2. 计算当日现金流入/流出
3. 计算市场价值变化（投资账户）
4. 计算调整项（差异）
5. 构建余额对象

#### 8.3.2 反向计算 (Reverse Calculator)

适用于**Plaid 连接账户**，从最新余额倒推历史余额：

```ruby
# app/models/balance/reverse_calculator.rb:2-82
def calculate
  end_cash_balance = derive_cash_balance_on_date_from_total(
    total_balance: account.current_anchor_balance,
    date: account.current_anchor_date
  )
  end_non_cash_balance = account.current_anchor_balance - end_cash_balance

  account.current_anchor_date.downto(account.opening_anchor_date).map do |date|
    # 倒序计算...
  end
end
```

**适用场景**：Plaid 提供的是当前账户余额，但历史流水完整，需要从当前倒推。

### 8.4 BaseCalculator 核心方法

`app/models/balance/base_calculator.rb` 提供了基础计算能力：

#### 流水分类 (`flows_for_date`)

```ruby
def flows_for_date(date)
  entries = sync_cache.get_entries(date)
  
  # 分类为：
  # - 现金流入/流出
  # - 非现金流入/流出
end
```

#### 余额构建 (`build_balance`)

构建包含以下字段的余额对象：
- `date`：日期
- `balance` / `cash_balance`：总余额/现金余额
- `start_cash_balance` / `start_non_cash_balance`：期初余额
- `cash_inflows` / `cash_outflows`：现金流量
- `non_cash_inflows` / `non_cash_outflows`：非现金流量
- `cash_adjustments` / `non_cash_adjustments`：调整项
- `net_market_flows`：市场价值变化
- `flows_factor`：资产(+1) / 负债(-1) 方向

### 8.5 Balance::SyncCache

`app/models/balance/sync_cache.rb` 提供计算所需的缓存数据：

```ruby
def get_valuation(date)    # 获取当日估值
def get_holdings(date)     # 获取当日持仓
def get_entries(date)      # 获取当日流水
```

**特点**：
- 自动处理货币转换
- 使用内存缓存避免重复查询

## 9. 数据物化到 balances 表

### 9.1 Balance 模型结构

`app/models/balance.rb`：

```ruby
class Balance < ApplicationRecord
  belongs_to :account
  
  # 核心字段
  monetize :balance, :cash_balance,
           :start_cash_balance, :start_non_cash_balance, :start_balance,
           :cash_inflows, :cash_outflows, :non_cash_inflows, :non_cash_outflows, :net_market_flows,
           :cash_adjustments, :non_cash_adjustments,
           :end_cash_balance, :end_non_cash_balance, :end_balance
end
```

**注意**：`end_balance`、`start_balance` 等字段可能是数据库生成列。

### 9.2 唯一索引

持久化时使用 `unique_by: %i[account_id date currency]` 确保：
- 每个账户、每天、每种货币只有一条余额记录

## 10. 同步完成与 UI 广播

### 10.1 同步完成后的事件链

```ruby
# app/models/sync.rb:142-149
def perform_post_sync
  Rails.logger.info("Performing post-sync for #{syncable_type} (#{syncable.id})")
  syncable.perform_post_sync       # 1. 执行后置同步
  syncable.broadcast_sync_complete # 2. 广播同步完成
end
```

### 10.2 Account::SyncCompleteEvent

`app/models/account/sync_complete_event.rb` 处理账户级别的广播：

```ruby
def broadcast
  # 1. 替换账户列表中的账户行
  account.broadcast_replace_to(
    account.family,
    target: "account_#{account.id}",
    partial: "accounts/account",
    locals: { account: account }
  )

  # 2. 替换侧边栏的账户分组
  sidebar_targets.each do |(tab, mobile_flag)|
    account.broadcast_replace_to(...)
  end

  # 3. 非链接账户触发家庭级广播
  unless account.linked?
    account.family.broadcast_sync_complete
  end

  # 4. 刷新账户详情页
  account.broadcast_refresh
end
```

### 10.3 Family::SyncCompleteEvent

`app/models/family/sync_complete_event.rb` 处理家庭级别的广播：

```ruby
def broadcast
  # 1. 更新资产负债表
  family.broadcast_replace(
    target: "balance-sheet",
    partial: "pages/dashboard/balance_sheet",
    locals: { balance_sheet: family.balance_sheet }
  )

  # 2. 更新净值图表
  family.broadcast_replace(
    target: "net-worth-chart",
    partial: "pages/dashboard/net_worth_chart",
    locals: { balance_sheet: family.balance_sheet, period: Period.last_30_days }
  )
end
```

## 11. 图表数据消费

### 11.1 账户级图表

`app/models/account/chartable.rb`：

```ruby
def balance_series(period: Period.last_30_days, view: :balance, interval: nil)
  builder = Balance::ChartSeriesBuilder.new(
    account_ids: [ id ],
    currency: self.currency,
    period: period,
    favorable_direction: favorable_direction,
    interval: interval
  )
  builder.send("#{view}_series")
end
```

### 11.2 家庭净值图表

`app/models/balance_sheet/net_worth_series_builder.rb`：

```ruby
def net_worth_series(period: Period.last_30_days)
  Rails.cache.fetch(cache_key(period)) do
    builder = Balance::ChartSeriesBuilder.new(
      account_ids: visible_account_ids,
      currency: family.currency,
      period: period,
      favorable_direction: "up"
    )
    builder.balance_series
  end
end
```

### 11.3 Balance::ChartSeriesBuilder

`app/models/balance/chart_series_builder.rb` 是连接余额数据和图表的桥梁：

```ruby
def build_series_for(column)
  values = query_data.map do |datum|
    Series::Value.new(
      date: datum.date,
      date_formatted: I18n.l(datum.date, format: :long),
      value: Money.new(datum.send(column), currency),
      trend: Trend.new(...)
    )
  end

  Series.new(
    start_date: period.start_date,
    end_date: period.end_date,
    interval: interval,
    values: values,
    favorable_direction: favorable_direction
  )
end
```

**核心查询逻辑**：
- 使用 `generate_series` 生成连续日期
- 通过 `LATERAL JOIN` 获取每个日期的最新余额
- 处理货币转换
- 聚合多个账户的余额

## 12. 完整时序图

```
用户操作
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│ TransactionsController#create/update                       │
│   └── @entry.save                                           │
│       └── @entry.sync_account_later                         │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│ Entry#sync_account_later (entry.rb:46)                      │
│   └── account.sync_later(window_start_date: ...)            │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│ Syncable#sync_later (concerns/syncable.rb:14)               │
│   ├── 创建或扩展 Sync 记录                                   │
│   └── SyncJob.perform_later(sync)                          │
└────────────────────────┬────────────────────────────────────┘
                         │  Sidekiq 异步执行
                         ▼
┌─────────────────────────────────────────────────────────────┐
│ SyncJob#perform (jobs/sync_job.rb)                          │
│   └── sync.perform                                          │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│ Sync#perform (sync.rb:60)                                   │
│   ├── syncable.perform_sync(self)                          │
│   │   └── Account::Syncer#perform_sync                      │
│   │       ├── import_market_data                            │
│   │       └── Balance::Materializer#materialize_balances    │
│   │           ├── materialize_holdings                      │
│   │           ├── calculator.calculate (Forward/Reverse)    │
│   │           ├── persist_balances (upsert_all)             │
│   │           ├── purge_stale_balances                      │
│   │           └── update_account_info (forward only)        │
│   └── finalize_if_all_children_finalized                    │
│       └── perform_post_sync                                 │
│           ├── syncable.perform_post_sync                    │
│           └── syncable.broadcast_sync_complete              │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│ Account::SyncCompleteEvent#broadcast                        │
│   ├── broadcast_replace_to (账户列表行)                     │
│   ├── broadcast_replace_to (侧边栏分组)                     │
│   ├── family.broadcast_sync_complete (非链接账户)           │
│   └── broadcast_refresh (账户详情页)                        │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│ Family::SyncCompleteEvent#broadcast                         │
│   ├── broadcast_replace (资产负债表)                        │
│   └── broadcast_replace (净值图表)                          │
└────────────────────────────────────┬────────────────────────┘
                                     │
                                     ▼
                          ┌──────────────────┐
                          │  UI 实时更新      │
                          │  - 账户列表      │
                          │  - 侧边栏        │
                          │  - 资产负债表    │
                          │  - 净值图表      │
                          └──────────────────┘
```

## 13. 关键组件速查表

| 组件 | 文件位置 | 职责 |
|------|----------|------|
| Entry | `app/models/entry.rb` | 流水模型，触发同步 |
| Syncable | `app/models/concerns/syncable.rb` | 同步能力混入 |
| Sync | `app/models/sync.rb` | 同步记录与状态管理 |
| SyncJob | `app/jobs/sync_job.rb` | 后台执行同步 |
| SyncCleanerJob | `app/jobs/sync_cleaner_job.rb` | 清理stale同步记录 |
| Account::Syncer | `app/models/account/syncer.rb` | 账户级同步执行器 |
| Family::Syncer | `app/models/family/syncer.rb` | 家庭级同步执行器 |
| Balance::Materializer | `app/models/balance/materializer.rb` | 余额物化核心 |
| Balance::ForwardCalculator | `app/models/balance/forward_calculator.rb` | 正向余额计算器 |
| Balance::ReverseCalculator | `app/models/balance/reverse_calculator.rb` | 反向余额计算器 |
| Balance::BaseCalculator | `app/models/balance/base_calculator.rb` | 计算器基类 |
| Balance::SyncCache | `app/models/balance/sync_cache.rb` | 计算数据缓存 |
| Balance::ChartSeriesBuilder | `app/models/balance/chart_series_builder.rb` | 图表数据构建器 |
| Account::SyncCompleteEvent | `app/models/account/sync_complete_event.rb` | 账户同步完成广播 |
| Family::SyncCompleteEvent | `app/models/family/sync_complete_event.rb` | 家庭同步完成广播 |

## 14. 设计要点总结

1. **异步化**：所有余额计算都通过 Sidekiq 后台 Job 异步执行，不阻塞用户操作
2. **窗口扩展**：避免重复 Job，优化频繁流水场景
3. **双策略计算**：根据账户类型（手动 vs 链接）选择正向/反向计算策略
4. **批量操作**：使用 `upsert_all` 批量持久化，性能优化
5. **事件驱动**：同步完成通过 Turbo Stream 实时更新 UI
6. **缓存机制**：
   - `SyncCache`：计算过程中的数据缓存
   - `Rails.cache`：图表数据缓存（依赖家庭缓存键失效）
7. **分层广播**：账户级广播 → 家庭级广播，精确控制更新范围
8. **事务保护**：整个物化过程在数据库事务中执行，确保数据一致性
9. **失败收敛**：父子同步状态自动传播，失败后仍广播更新
10. **Stale恢复**：通过新流水触发自动创建新同步，无需人工干预
