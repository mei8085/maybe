# 同步并发与失败场景边界分析 v2

## 一、并发场景：窗口扩展机制

### 1.1 核心问题

当同一 syncable 已有未完成同步时，新的 `sync_later` 调用如何处理？

### 1.2 代码路径分析

```ruby
# app/models/concerns/syncable.rb:14-35
def sync_later(parent_sync: nil, window_start_date: nil, window_end_date: nil)
  Sync.transaction do
    with_lock do
      sync = self.syncs.incomplete.first

      if sync
        Rails.logger.info("There is an existing sync, expanding window if needed (#{sync.id})")
        sync.expand_window_if_needed(window_start_date, window_end_date)
      else
        sync = self.syncs.create!(...)
        SyncJob.perform_later(sync)
      end
      sync
    end
  end
end
```

### 1.3 并发安全保障

| 机制 | 位置 | 作用 |
|------|------|------|
| `Sync.transaction` | `sync_later` 外层 | 事务包裹，原子性 |
| `with_lock` | `sync_later` 内层 | 对 syncable 行加排他锁（SELECT FOR UPDATE），防止并发竞态 |
| `syncs.incomplete.first` | 锁内查询 | 在持有锁的情况下查询是否存在 pending/syncing |

**竞态场景避免**：
- 两个并发请求同时调用 `sync_later`
- 第一个请求获得锁，创建新 sync，释放锁
- 第二个请求获得锁时，`incomplete.first` 已返回存在的 sync，不会重复创建

### 1.4 窗口扩展的边界条件

```ruby
# app/models/sync.rb:107-127
def expand_window_if_needed(new_window_start_date, new_window_end_date)
  return unless pending?                              # 条件 1
  return if self.window_start_date.nil? && self.window_end_date.nil?  # 条件 2

  # 取更早的 start_date（min），更晚的 end_date（max）
  update(
    window_start_date: [existing, new].min,  # nil 表示无边界
    window_end_date: [existing, new].max
  )
end
```

**状态约束矩阵**：

| 现有同步状态 | 新 `sync_later` 行为 | 原因 |
|-------------|---------------------|------|
| `pending` | 扩展窗口（如果新窗口更宽），**不创建新同步**，**不重新入队** | `expand_window_if_needed` 有完整逻辑 |
| `syncing` | **不扩展窗口**，**不创建新同步**，**不重新入队** | `return unless pending?` 直接返回 |
| `completed` | 创建新同步 | 不在 `incomplete` scope 中 |
| `failed` | 创建新同步 | 不在 `incomplete` scope 中 |
| `stale` | 创建新同步 | 不在 `incomplete` scope 中 |

### 1.5 窗口扩展策略的设计意图

**"取并集"原则**：
- `window_start_date` 取 **min**（更早 = 覆盖更多历史数据）
- `window_end_date` 取 **max**（更晚 = 覆盖更多未来/最新数据）
- **nil 代表无边界**：如果任一参与方为 nil，则结果为 nil（全范围）

**窗口计算逻辑表**：

| 现有窗口 | 新请求窗口 | 扩展后窗口 | 说明 |
|---------|-----------|-----------|------|
| `[5天前, 今天]` | `[3天前, 今天]` | `[5天前, 今天]` | 新窗口更窄，无变化 |
| `[5天前, 今天]` | `[7天前, 今天]` | `[7天前, 今天]` | 扩展 start |
| `[5天前, 今天]` | `[5天前, 明天]` | `[5天前, 明天]` | 扩展 end |
| `[5天前, 今天]` | `[nil, nil]` | `[nil, nil]` | 新请求无边界 → 全范围 |
| `[nil, nil]` | `[5天前, 今天]` | `[nil, nil]` | 条件 2 命中，直接返回（已最宽） |

### 1.6 关键边界：syncing 状态下的静默忽略

这是一个容易被忽略的重要边界：

```
时序示意：
T0: 触发 sync_later → 创建 Sync(pending) → 入队 Sidekiq
T1: Sidekiq 开始执行 → start! → Sync(syncing)
T2: 用户又点了一次同步 → sync_later 被调用
    ├─ 查到 incomplete.first = 这个 sync(syncing)
    ├─ 调用 expand_window_if_needed
    │  └─ return unless pending? → 直接返回
    └─ 不创建新 sync，不重新入队
T3: 原有 sync 继续执行，使用旧窗口
```

**后果**：
- 在 `syncing` 期间的新同步请求会被**静默忽略**
- 新请求的窗口参数**不会被采纳**
- 如果新请求需要更宽的窗口，必须等当前同步完成后再触发

---

## 二、失败场景：父子同步收敛

### 2.1 状态机概览

```
pending ──start!──► syncing ──complete!──► completed
                      │
                      ├─fail!──► failed
                      │
                      └─mark_stale!──► stale  (24h 超时)
```

### 2.2 子同步失败的上行传播

**触发路径**：子同步完成（无论成功失败）→ 调用 `finalize_if_all_children_finalized`

```ruby
# app/models/sync.rb:83-104
def finalize_if_all_children_finalized
  Sync.transaction do
    lock!

    return unless all_children_finalized?  # 等所有兄弟完成

    if syncing?
      if has_failed_children?
        fail!          # 关键：任一子失败 → 父也失败
      else
        complete!
      end
    end

    perform_post_sync  # 关键点：无论成功失败都会执行
  end

  parent&.finalize_if_all_children_finalized  # 递归向上
end
```

### 2.3 收敛策略详解

#### 策略 1：等待所有兄弟完成

```ruby
return unless all_children_finalized?
# 定义：children.incomplete.empty?
# incomplete = pending OR syncing
```

**含义**：
- 父同步不会在有任何子同步还在运行时就"提前结束"
- 必须等待**所有子同步**都进入终态（completed / failed / stale）

**场景示例**：

```
Family Sync (syncing)
  ├─ Account A Sync (completed) ── 触发 finalize
  │                               └─ all_children_finalized? = false
  │                                  （B 还在 syncing），直接 return
  └─ Account B Sync (syncing)  ── 完成后再次触发 finalize
                                  └─ all_children_finalized? = true
                                     此时才检查 has_failed_children?
```

#### 策略 2：失败一票否决

```ruby
if has_failed_children?
  fail!
```

**含义**：
- 任一子同步 `failed` → 父同步也 `failed`
- 即使其他 99% 的子同步都成功了

**传播链条**：

```
Account C Sync ──fail!──► failed
    │
    ▼
PlaidItem Sync ──finalize──► has_failed_children? = true ──► fail!
    │
    ▼
Family Sync ──finalize──► has_failed_children? = true ──► fail!
```

**最终状态**：整个链路从叶子到根全部标记为 `failed`

#### 策略 3：Stale 也算"完成"

Stale 状态的同步：
- 不在 `incomplete` scope 中
- 但也不在 `failed` scope 中

**关键问题**：stale 的子同步对父同步有什么影响？

```
stale = 不在 incomplete（不会阻塞等待）
      = 不在 failed（不会触发 has_failed_children?）
```

**实际影响**：
- stale 子同步会被当作"已完成且成功"？
- 父同步会正常 `complete!`

### 2.4 父同步失败收敛的完整场景矩阵

| 子同步状态组合 | `all_children_finalized?` | `has_failed_children?` | 父同步结果 |
|---------------|--------------------------|-----------------------|-----------|
| 全部 completed | true | false | complete! |
| 1 failed，其余 completed | true | true | fail! |
| 1 stale，其余 completed | true | false | complete!（stale 不算 failed） |
| 1 syncing，其余 completed | false | N/A | 阻塞等待 |
| 1 pending，其余 completed | false | N/A | 阻塞等待 |

---

## 三、后同步（Post-Sync）在失败链路中的执行

### 3.1 后同步的触发时机

```ruby
# app/models/sync.rb:60-79
def perform
  # ...
  begin
    syncable.perform_sync(self)
  rescue => e
    fail!
    update(error: e.message)
    report_error(e)
  ensure
    finalize_if_all_children_finalized  # 确保块：无论异常与否都会执行
  end
end
```

```ruby
# app/models/sync.rb:83-104
def finalize_if_all_children_finalized
  Sync.transaction do
    lock!
    return unless all_children_finalized?

    if syncing?
      if has_failed_children?
        fail!      # 这里也可能把状态从 syncing 转为 failed
      else
        complete!
      end
    end

    perform_post_sync  # 关键点 1：在检查完状态后执行
  end

  parent&.finalize_if_all_children_finalized  # 关键点 2：递归向上
end
```

### 3.2 后同步的执行条件分析

**`perform_post_sync` 调用位置**：在 `if syncing?` 块**之后**

这意味着：

| 场景 | `syncing?` 检查 | 状态转换 | post-sync 执行 |
|------|----------------|----------|---------------|
| 主流程成功 | true → 执行 complete! | syncing → completed | **执行** |
| perform_sync 抛异常 | 不在 finalize 的这个分支（已 fail!） | syncing → failed | **见下方分析** |
| 子失败导致父失败 | true → 执行 fail! | syncing → failed | **执行**（在 fail! 之后） |
| 所有子都完成但父已非 syncing | false，跳过状态转换 | 无变化 | **执行**（无条件） |
| 还有子在运行 | `all_children_finalized?` = false | 直接 return | **不执行** |

### 3.3 两种失败路径的对比

#### 路径 A：自身 perform_sync 抛异常

```
perform_sync(self)
  └─ 抛出异常
     └─ rescue 块执行
        ├─ fail!               # syncing → failed
        ├─ update(error: ...)
        └─ ensure 块执行
           └─ finalize_if_all_children_finalized
              ├─ 检查 all_children_finalized?
              │  └─ 如果是叶子节点（无子同步）= true
              ├─ syncing? = false（已经 fail! 过了）
              │  └─ 跳过状态转换块
              └─ perform_post_sync  ← **仍然执行！**
```

#### 路径 B：因子同步失败导致父同步失败

```
子同步完成（failed）
  └─ 调用 parent.finalize_if_all_children_finalized
     └─ 父同步进入 finalize
        ├─ all_children_finalized? = true
        ├─ syncing? = true
        ├─ has_failed_children? = true
        ├─ fail!             # syncing → failed
        └─ perform_post_sync ← **执行！**
```

### 3.4 结论：后同步总是在最终收敛时执行

**后同步执行的保障**：

1. **`ensure` 块保障**：`finalize_if_all_children_finalized` 在 `perform` 的 ensure 中调用
2. **无状态检查**：`perform_post_sync` 调用**不在** `if syncing?` 条件内，只要通过了 `all_children_finalized?` 就会执行
3. **失败也不跳过**：无论是自身异常还是子失败传播，post-sync 都会执行

**后同步不执行的唯一情况**：
- 还有子同步在运行（`all_children_finalized?` = false）
- 此时父同步尚未"收敛"，不应该触发 post-sync

### 3.5 实际业务影响

以转账自动匹配为例：

```ruby
# Account::Syncer 和 Family::Syncer 的 perform_post_sync 都会调用
account.family.auto_match_transfers!
```

**场景：某个账户同步失败，其他账户成功**

```
Family Sync (syncing)
  ├─ Account A Sync ──► completed ──► post-sync: auto_match_transfers!
  ├─ Account B Sync ──► failed    ──► post-sync: auto_match_transfers!（仍执行）
  └─ Account C Sync ──► completed ──► post-sync: auto_match_transfers!

最终：Family Sync ──► has_failed_children? = true ──► fail!
                                    └─ post-sync: auto_match_transfers!（仍执行）
```

**结果**：
- 即使整个同步链路失败，转账匹配仍会被多次触发（每个子同步的 post-sync + 父同步的 post-sync）
- 这是有意设计：转账匹配是幂等操作，成功的数据应该被尽可能处理

---

## 四、完整失败链路时序图

```
┌─────────────────────────────────────────────────────────────────────┐
│  场景：Family 同步下有 2 个 PlaidItem，其中 Account C 同步失败      │
└─────────────────────────────────────────────────────────────────────┘

T0: 用户触发 family.sync_later
    └─ 创建 Family Sync (pending)，入队

T1: Family SyncJob 执行
    ├─ start! → syncing
    ├─ Family::Syncer.perform_sync
    │  ├─ sync_trial_status!
    │  ├─ 调度 rules
    │  └─ 调度子同步：
    │     ├─ PlaidItem A.sync_later(parent_sync: family_sync, ...)
    │     └─ PlaidItem B.sync_later(parent_sync: family_sync, ...)
    └─ ensure → finalize_if_all_children_finalized
       ├─ all_children_finalized? = false（子同步刚创建，都是 pending）
       └─ 直接 return，不执行 post-sync

T2: PlaidItem A SyncJob 执行
    ├─ import_latest_plaid_data ✓
    ├─ process_accounts ✓
    └─ schedule_account_syncs
       ├─ Account A.sync_later(parent_sync: plaid_item_a_sync, ...)
       └─ Account B.sync_later(parent_sync: plaid_item_a_sync, ...)
    └─ ensure → finalize_if_all_children_finalized
       └─ all_children_finalized? = false（A/B pending），return

T3: PlaidItem B SyncJob 执行（类似 A）
    └─ 调度 Account C 和 Account D
    └─ finalize 时阻塞等待

T4: Account A SyncJob ──► completed
    └─ finalize → 父 PlaidItem A 仍有 B 在运行，阻塞

T5: Account B SyncJob ──► completed
    └─ finalize
       ├─ PlaidItem A: all_children_finalized? = true
       ├─ has_failed_children? = false
       ├─ complete!
       └─ perform_post_sync（PlaidItem 的是 no-op）
       └─ 向上递归：Family Sync 还有 B 的子同步在运行，阻塞

T6: Account D SyncJob ──► completed
    └─ finalize → PlaidItem B 仍有 C 在运行，阻塞

T7: Account C SyncJob ──► perform_sync 抛异常！
    ├─ rescue 块
    │  ├─ fail! → syncing → failed
    │  └─ update(error: "...")
    └─ ensure → finalize_if_all_children_finalized
       ├─ all_children_finalized? = true（无子同步）
       ├─ syncing? = false（已 fail!）
       ├─ perform_post_sync ← 仍执行！（转账匹配）
       └─ 向上递归：PlaidItem B.finalize

T8: PlaidItem B 在 finalize 中
    ├─ all_children_finalized? = true
    ├─ syncing? = true
    ├─ has_failed_children? = true（Account C failed）
    ├─ fail! → syncing → failed
    ├─ perform_post_sync ← 执行！（no-op）
    └─ 向上递归：Family Sync.finalize

T9: Family Sync 在 finalize 中
    ├─ all_children_finalized? = true（PlaidItem A completed, B failed）
    ├─ syncing? = true
    ├─ has_failed_children? = true（PlaidItem B failed）
    ├─ fail! → syncing → failed
    └─ perform_post_sync ← 执行！（转账匹配 + 广播）

T10: 最终状态
    ├─ Family Sync: failed
    ├─ PlaidItem A Sync: completed
    ├─ PlaidItem B Sync: failed
    ├─ Account A/B/D: completed（各自的 post-sync 已执行）
    └─ Account C: failed（但 post-sync 也执行了）
```

---

## 五、并发与失败场景的边界总结表

### 5.1 并发边界

| 问题 | 答案 | 代码位置 |
|------|------|----------|
| 已有 pending 同步时新请求如何处理？ | 扩展窗口（如果更宽），不新建不同步 | `syncable.rb:19-21`, `sync.rb:107-127` |
| 已有 syncing 同步时新请求如何处理？ | **静默忽略**，不扩展不新建不同步 | `sync.rb:108` `return unless pending?` |
| 如何防止并发创建多个同步？ | 事务 + 行锁（`with_lock`）+ 锁内查询 | `syncable.rb:15-17` |
| 窗口扩展策略？ | start 取 min（更早），end 取 max（更晚），nil 表示无边界 | `sync.rb:111-126` |

### 5.2 失败传播边界

| 问题 | 答案 | 代码位置 |
|------|------|----------|
| 子失败会影响父吗？ | 会，`has_failed_children?` → 父也 `fail!` | `sync.rb:91-92` |
| 父会等待所有子完成吗？ | 会，`all_children_finalized?` 阻塞 | `sync.rb:88` |
| stale 算完成吗？ | 算（不在 incomplete），但不算失败 | `sync.rb:19` scope |
| stale 子会让父失败吗？ | 不会（不在 `failed` scope） | `sync.rb:135` |
| 传播方向？ | 自底向上递归：`parent&.finalize_if_all_children_finalized` | `sync.rb:103` |

### 5.3 后同步执行边界

| 问题 | 答案 | 代码位置 |
|------|------|----------|
| 自身异常时 post-sync 执行吗？ | **执行**，在 ensure → finalize 中 | `sync.rb:77`, `sync.rb:99` |
| 因子失败而失败时执行吗？ | **执行**，在 fail! 之后 | `sync.rb:92-99` |
| 还有子在运行时执行吗？ | **不执行**，被 `all_children_finalized?` 拦截 | `sync.rb:88` |
| post-sync 异常会怎样？ | 被 rescue 捕获，记录错误但不影响同步状态 | `sync.rb:142-149` |
| post-sync 调用几次？ | 每个完成的节点（子+父）各一次，可能重复 | `sync.rb:99` + `sync.rb:103` 递归 |

---

## 六、代码位置索引

| 关注点 | 文件 | 行号 |
|--------|------|------|
| 同步入队与窗口扩展 | `app/models/concerns/syncable.rb` | 14-35 |
| 窗口扩展逻辑 | `app/models/sync.rb` | 107-127 |
| 同步主流程 | `app/models/sync.rb` | 60-80 |
| 父子收敛与状态传播 | `app/models/sync.rb` | 82-104 |
| 后同步执行 | `app/models/sync.rb` | 98-99, 142-149 |
| 状态机定义 | `app/models/sync.rb` | 26-52 |
| incomplete scope | `app/models/sync.rb` | 19 |
| 每日同步次数警告 | `app/models/sync.rb` | 157-166 |
| Stale 清理定时任务 | `app/jobs/sync_cleaner_job.rb` | 全部 |
