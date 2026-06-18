# 账户余额快照修正路径说明

## 一、核心概念

### 1.1 余额快照（Balance）

余额快照是每日账户余额的"切片"，存储在 `balances` 表中。每一天一条记录，包含该日的起始余额、各类流入流出、调整项、最终余额等完整信息。

核心字段：

| 字段 | 含义 |
|------|------|
| `start_balance` / `end_balance` | 当日期初 / 期末总余额 |
| `start_cash_balance` / `end_cash_balance` | 当日期初 / 期末现金部分余额 |
| `start_non_cash_balance` / `end_non_cash_balance` | 当日期初 / 期末非现金部分余额（如持仓、本金等） |
| `cash_inflows` / `cash_outflows` | 当日现金流入 / 流出 |
| `non_cash_inflows` / `non_cash_outflows` | 当日非现金流入 / 流出（如买入证券、还本金） |
| `net_market_flows` | 当日市值变动（投资类账户） |
| `cash_adjustments` / `non_cash_adjustments` | 当日现金 / 非现金调整项（手工修正的体现） |
| `flows_factor` | 流向因子（资产=1，负债=-1） |

相关文件：
- [balance.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/balance.rb)

### 1.2 估值锚点（Valuation）

估值记录（`valuations` 表，通过 `entries` 关联）是余额计算的"锚点"，有三种类型：

| 类型 | 作用 | 相关代码 |
|------|------|----------|
| `opening_anchor` | 期初余额锚点 — 账户起始日的余额基准 | [opening_balance_manager.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/account/opening_balance_manager.rb) |
| `current_anchor` | 当前余额锚点 — 关联账户（如Plaid）的最新余额 | [current_balance_manager.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/account/current_balance_manager.rb) |
| `reconciliation` | 对账调整 — 用户手动设置的某一天余额（用于修正差异） | [reconciliation_manager.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/account/reconciliation_manager.rb) |

相关文件：
- [valuation.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/valuation.rb)

---

## 二、重算（Recalculation）

### 2.1 重算的触发时机 — sync_later 的调用入口全景

当任何可能影响余额的操作发生时，系统不会立即重算，而是通过 `sync_later` 调度一个异步同步任务。

`sync_later` 是 **Syncable** 模块提供的统一入口，它的核心行为是：
1. 如果该 syncable 已有 pending/syncing 状态的同步任务，则复用它并扩展窗口
2. 否则创建新的 Sync 记录，交给 `SyncJob` 异步执行

#### 2.1.1 所有触发 sync_later 的入口

**A. 手工修正类（直接触发）**

| 操作 | 触发位置 | 代码位置 |
|------|----------|----------|
| 设置期初余额 | `Anchorable#set_opening_anchor_balance` | [anchorable.rb#L12-L16](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/account/anchorable.rb#L12-L16) |
| 设置当前余额 | `Anchorable#set_current_balance` | [anchorable.rb#L30-L34](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/account/anchorable.rb#L30-L34) |
| 创建对账 | `Reconcileable#create_reconciliation` | [reconcileable.rb#L4-L8](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/account/reconcileable.rb#L4-L8) |
| 更新对账 | `Reconcileable#update_reconciliation` | [reconcileable.rb#L10-L14](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/account/reconcileable.rb#L10-L14) |

**B. 控制器 HTTP 请求（用户操作触发）**

| 控制器动作 | 触发方式 | 代码位置 |
|-----------|----------|----------|
| 账户创建 | `create_and_sync` 内部 | [accountable_resource.rb#L37](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/controllers/concerns/accountable_resource.rb#L37) |
| 账户更新（改余额） | 显式调用 `@account.sync_later` | [accountable_resource.rb#L52](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/controllers/concerns/accountable_resource.rb#L52) |
| 房产余额更新 | 通过 `set_current_balance` 间接触发 | [properties_controller.rb#L40](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/controllers/properties_controller.rb#L40) |
| 对账创建 | 通过 `create_reconciliation` 间接触发 | [valuations_controller.rb#L35-L38](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/controllers/valuations_controller.rb#L35-L38) |
| 对账更新 | 通过 `update_reconciliation` 间接触发 | [valuations_controller.rb#L56-L60](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/controllers/valuations_controller.rb#L56-L60) |
| 对账删除 | `EntryableResource#destroy` → `sync_account_later` | [entryable_resource.rb#L31-L37](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/controllers/concerns/entryable_resource.rb#L31-L37) |
| 交易批量删除 | 对每个受影响账户调用 `sync_later` | [bulk_deletions_controller.rb#L4](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/controllers/transactions/bulk_deletions_controller.rb#L4) |
| 手动触发单账户同步 | `AccountsController#sync` | [accounts_controller.rb#L30](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/controllers/accounts_controller.rb#L30) |
| 手动触发全家同步 | `AccountsController#sync_all` | [accounts_controller.rb#L13](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/controllers/accounts_controller.rb#L13) |
| Plaid 手动同步 | `PlaidItemsController#sync` | [plaid_items_controller.rb#L42](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/controllers/plaid_items_controller.rb#L42) |

**C. 模型层业务操作（间接触发）**

| 操作 | 触发位置 | 代码位置 |
|------|----------|----------|
| 交易/对账/交易删除 | `Entry#sync_account_later` | [entry.rb#L46-L49](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/entry.rb#L46-L49) |
| 转账创建 | 源账户 + 目标账户各触发一次 | [transfer/creator.rb#L18-L19](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/transfer/creator.rb#L18-L19) |
| 交易创建/更新 | 通过 Entry callback 触发 | (同上 Entry) |
| 持仓更新 | `Holding` callback | [holding.rb#L60](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/holding.rb#L60) |
| 交易表单创建 | 买卖/分红/拆分等 | [trade/create_form.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/trade/create_form.rb) |
| Plaid Webhook | 交易/投资/持仓更新 | [plaid_item/webhook_processor.rb#L20-L24](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/plaid_item/webhook_processor.rb#L20-L24) |
| 数据导入完成 | 全家级触发 | [import.rb#L70](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/import.rb#L70) |

**D. 同步器级联调度（Family → PlaidItem → Account）**

Family 同步器不直接重算，而是调度子同步任务：

```ruby
# Family::Syncer
def perform_sync(sync)
  child_syncables.each do |syncable|
    syncable.sync_later(parent_sync: sync, ...)  # 调度每个 PlaidItem 和 手工Account
  end
end

# PlaidItem 同步完成后，也会调度其下的每个 Account
def schedule_account_syncs(parent_sync:, ...)
  accounts.each { |acct| acct.sync_later(parent_sync:, ...) }
end
```

相关文件：
- [syncable.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/concerns/syncable.rb)
- [family/syncer.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/family/syncer.rb)
- [plaid_item.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/plaid_item.rb)

### 2.2 同步调度的完整链路 — 从 sync_later 到 Materializer

这是最容易让人困惑的部分：**用户操作只改锚点不直接重算，重算一定发生在 SyncJob 异步执行阶段**。

#### 完整调度链路图

```
用户操作（HTTP请求）
  │
  ├─ 修改 Valuation 锚点 / Transaction / Trade
  │
  └─ 调用 sync_later
       │
       ▼
  Syncable#sync_later（同步模块入口）
    ├─ 检查是否有 pending/syncing 的 Sync？
    │   ├─ 有 → expand_window_if_needed（扩展窗口，不复用新任务）
    │   └─ 无 → 创建新 Sync 记录
    │
    └─ SyncJob.perform_later(sync)  ← 交给 Sidekiq 异步队列
         │
         ▼  （以下在后台 Worker 中执行）
  SyncJob#perform(sync)
    └─ sync.perform
         │
         ▼
  Sync#perform
    ├─ start! （状态：pending → syncing）
    │
    ├─ syncable.perform_sync(self)  ← 分发到各层级 Syncer
    │    │
    │    ├─ Family::Syncer  →  只调度子同步，不做重算
    │    │    └─ 对每个 PlaidItem + 手工 Account 调用 sync_later
    │    │
    │    ├─ PlaidItem::Syncer  →  拉数据后调度 Account
    │    │    └─ 对每个关联 Account 调用 schedule_account_syncs
    │    │
    │    └─ Account::Syncer  →  **真正的重算在这里**
    │         ├─ import_market_data（汇率/股价）
    │         └─ Balance::Materializer#materialize_balances
    │              ├─ materialize_holdings  （持仓物化）
    │              ├─ calculate_balances    （调用计算器）
    │              ├─ persist_balances      （upsert 到 balances 表）
    │              ├─ purge_stale_balances  （清理过期快照）
    │              └─ update_account_info   （更新 account.balance 缓存）
    │
    └─ finalize_if_all_children_finalized
         ├─ 等所有子 Sync 完成
         ├─ complete! / fail!（状态：syncing → completed/failed）
         ├─ perform_post_sync（自动匹配转账等）
         └─ broadcast_sync_complete（Turbo 广播刷新 UI）
```

#### 关键代码位置

| 阶段 | 代码位置 |
|------|----------|
| sync_later 入口 | [syncable.rb#L14-L35](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/concerns/syncable.rb#L14-L35) |
| SyncJob 异步入口 | [sync_job.rb#L1-L7](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/jobs/sync_job.rb#L1-L7) |
| Sync 状态机执行 | [sync.rb#L60-L80](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/sync.rb#L60-L80) |
| Account 重算入口 | [account/syncer.rb#L8-L12](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/account/syncer.rb#L8-L12) |
| Family 级联调度 | [family/syncer.rb#L8-L21](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/family/syncer.rb#L8-L21) |
| 同步完成后处理 | [sync.rb#L82-L104](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/sync.rb#L82-L104) |

### 2.3 重算的执行流程

重算由 `Balance::Materializer` 统一调度：

```
Account::Syncer#perform_sync
  ↓
Balance::Materializer#materialize_balances
  ├─ materialize_holdings (持仓物化，投资类账户)
  ├─ calculate_balances (调用计算器计算)
  ├─ persist_balances (批量 upsert 到 balances 表)
  ├─ purge_stale_balances (删除过期快照)
  └─ update_account_info (更新 account 缓存字段)
```

**重要细节 — update_account_info 与 account.balance 的关系**：

重算完成后，Materializer 会用最新快照的期末余额覆盖 `account.balance` 缓存字段：

```ruby
# materializer.rb L30-L52
def update_account_info
  current_balance = account.balances.order(date: :desc).first

  account.update!(
    balance: current_balance.end_balance,        # 覆盖缓存
    cash_balance: current_balance.end_cash_balance
  )
end
```

这意味着：如果用户在 UI 修改了余额（set_current_balance 会先写 account.balance），但随后异步重算完成时，account.balance 会被 **重算结果再次覆盖**。两者应该是一致的（因为锚点已写入），但时序上存在一个短暂的窗口。

相关文件：
- [materializer.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/balance/materializer.rb)
- [syncer.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/account/syncer.rb)

### 2.4 两种计算策略

根据账户类型不同，使用不同的计算方向：

| 策略 | 适用账户 | 计算方向 | 起点 | 终点 |
|------|----------|----------|------|------|
| `:forward`（正向） | 手工账户 | 从旧到新 | opening_anchor | 最后一条记录日期 |
| `:reverse`（反向） | 关联账户（Plaid） | 从新到旧 | current_anchor | opening_anchor |

#### 正向计算（ForwardCalculator）

从期初余额开始，逐日累加交易和调整，推算出每天的期末余额：

```ruby
# 伪代码逻辑
start_balance = opening_anchor_balance
for date in opening_date..latest_date:
  if date has valuation:
    end_balance = valuation.amount  # 锚点日直接使用估值
  else:
    end_balance = start_balance + net_flows  # 非锚点日根据交易推算
  
  adjustments = end_balance - start_balance - net_flows  # 计算调整项
  save_balance_record(date, start_balance, end_balance, adjustments, ...)
  
  start_balance = end_balance  # 下一天的期初 = 当天的期末
```

关键细节：
- 遇到 `reconciliation` 类型的估值时，直接使用该估值作为当日期末余额
- `adjustments` = 期末余额 - 期初余额 - 净流量（即"无法用交易解释的差额"）
- 投资类账户还会计算 `net_market_flows`（市值变动）

相关文件：
- [forward_calculator.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/balance/forward_calculator.rb)
- [base_calculator.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/balance/base_calculator.rb)

#### 反向计算（ReverseCalculator）

从当前余额（current_anchor）开始，逐日往回推算：

```ruby
# 伪代码逻辑
end_balance = current_anchor_balance
for date in current_date.downto(opening_date):
  if date == opening_anchor_date:
    end_balance = opening_anchor_balance  # 到期初锚点时，使用期初估值
    start_balance = end_balance
  else:
    start_balance = end_balance - net_flows  # 往回推算期初余额
  
  save_balance_record(date, start_balance, end_balance, ...)
  end_balance = start_balance  # 前一天的期末 = 当天的期初
```

注意：反向计算不使用 `reconciliation` 估值，仅使用 `opening_anchor` 和 `current_anchor` 两个锚点。

相关文件：
- [reverse_calculator.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/balance/reverse_calculator.rb)

---

## 三、手工修正（Manual Adjustment）— 何时追加、何时更新

手工修正有三种入口，分别对应不同的场景和锚点类型。**同日更新 vs 追加新记录** 的决策逻辑是本节的重点。

### 3.1 期初余额修正（Opening Balance）

**场景**：设置账户的起始余额（如房产的购入价、账户初始余额）。

**入口**：
- 创建账户时自动设置（`Account.create_and_sync`）
- 导入账户时设置（`AccountImport`）
- 通过 `set_opening_anchor_balance` 方法

**流程**：

```
用户修改期初余额
  ↓
OpeningBalanceManager#set_opening_balance
  ├─ 存在 opening_anchor → 更新 amount / date（UPDATE，不新建）
  └─ 不存在 → 创建新的 opening_anchor 估值记录（INSERT）
  ↓
sync_later（触发异步重算）
  ↓
重算时，ForwardCalculator 以 opening_anchor 为起点逐日计算
```

**追加 vs 更新 的判定逻辑**（[opening_balance_manager.rb#L26-L44](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/account/opening_balance_manager.rb#L26-L44)）：

```ruby
def set_opening_balance(balance:, date: nil)
  if opening_anchor_valuation.nil?
    create_opening_anchor(...)                    # 不存在 → 新建（追加）
    Result.new(changes_made?: true, ...)
  else
    changes_made = update_opening_anchor(...)     # 已存在 → 原地更新（同一条记录）
    Result.new(changes_made?: changes_made, ...)  # 如果值/日期没变，changes_made=false
  end
end
```

结论：
- **opening_anchor 有且仅有一条**（整个账户生命周期）
- 首次设置 → 追加 1 条
- 后续修改 → 更新同一条，不追加

**代码要点**：
- 期初余额日期必须早于最早的交易记录日期
- 期初余额会影响后续所有日期的余额计算（正向传播）

相关文件：
- [opening_balance_manager.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/account/opening_balance_manager.rb)

### 3.2 当前余额修正（Current Balance）— 最复杂的决策分支

**场景**：用户在账户编辑页直接修改"当前余额"。

**入口**：
- 账户更新接口（`AccountableResource#update`）
- 资产余额更新（`PropertiesController#update_balances`）

**完整流程 + 追加/更新判定**：

```
用户提交 balance=xxxx
  ↓
Anchorable#set_current_balance
  └─ CurrentBalanceManager#set_current_balance(balance)
       ├─ account.linked?（关联账户走A分支，手工走B分支）
       │
       │  【A分支】关联账户（Plaid）
       │    └─ current_anchor 存在？
       │         ├─ 存在 → update_current_anchor（UPDATE）
       │         │    · 金额变了 → 更新 amount
       │         │    · 日期变了（跨天）→ 更新 date
       │         │    · 都没变 → changes_made=false
       │         └─ 不存在 → create_current_anchor（INSERT，追加1条）
       │
       │  【B分支】手工账户
       │    └─ balance_type == :cash 且 无 reconciliation 估值？
       │         ├─ 是（交易调整策略）→ adjust_opening_balance_with_delta
       │         │    · 计算 delta = new - old
       │         │    · 调用 opening_balance_manager.set_opening_balance(
       │         │          opening_balance + delta)
       │         │    · 最终效果：UPDATE opening_anchor，不新增记录
       │         │
       │         └─ 否（价值追踪策略）→ reconciliation_manager.reconcile_balance
       │              └─ prepare_reconciliation 内部判定：
       │                 (1) existing_valuation_entry 作为参数传进来了？
       │                     → 直接用它（UPDATE）
       │                 (2) 否则，查数据库有没有 date == Date.current 的估值？
       │                     · 有 → 直接用它（UPDATE，同日更新）
       │                     · 无 → build 新记录（INSERT，追加）
       │
       ├─ account.update!(balance: balance)  ← 立即更新缓存（HTTP响应立即可见）
       │
       └─ 返回 Result，上层再触发 sync_later
```

#### 同日更新 vs 追加 的精确判定（以手工账户"价值追踪策略"为例）

核心代码在 [reconciliation_manager.rb#L44-L59](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/account/reconciliation_manager.rb#L44-L59)：

```ruby
def prepare_reconciliation(balance, date, existing_valuation)
  valuation_record =
    existing_valuation ||                              # 优先级1：调用方传入的现有记录
    account.entries.valuations.find_by(date: date) ||  # 优先级2：查同日是否已有
    account.entries.build(...)                         # 优先级3：都没有才新建

  valuation_record.assign_attributes(date:, amount:, currency:)
  valuation_record
end
```

这意味着：
| 情况 | 行为 |
|------|------|
| 同日第一次设置 | 追加 1 条 reconciliation |
| 同日第二次（及以后）设置 | 原地 UPDATE，不追加 |
| 跨天设置 | 新的日期追加 1 条 |
| 传了 existing_valuation_entry（编辑页场景） | 无论如何都 UPDATE 那条记录，即使日期改了 |

**非现金手工账户的特殊行为**：

Property、Vehicle、Loan 等 `non_cash` 类型账户，**即使是第一次设置当前余额，也一定走"价值追踪策略"**，直接追加一条 reconciliation（见测试 [current_balance_manager_test.rb#L48-L71](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/test/models/account/current_balance_manager_test.rb#L48-L71)）。这是因为这类账户没有交易，只能靠对账来追踪价值变化。

**两种自动策略总结（手工账户）**：

1. **交易调整策略**（现金账户 + 无对账记录时）：
   - 调整 opening_balance 的数值（"回推"到期初）
   - 适用于用户主要通过交易来跟踪余额的场景（如现金账户）
   - 修改余额不会产生额外的对账记录，只 UPDATE opening_anchor

2. **价值追踪策略**（有对账记录 / 非现金账户时）：
   - 同日 → UPDATE 同一条 reconciliation
   - 跨日 → INSERT 新的 reconciliation
   - 适用于用户主要通过对账来跟踪账户价值的场景（如房产、投资）

相关文件：
- [current_balance_manager.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/account/current_balance_manager.rb)
- [reconciliation_manager.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/account/reconciliation_manager.rb)

### 3.3 对账调整（Reconciliation）

**场景**：用户在特定日期设置余额，用于修正与实际值的差异。这是从 ValuationsController 直接调用的独立流程，与 CurrentBalanceManager 无关。

**入口**：`create_reconciliation` / `update_reconciliation`

**流程**：

```
用户设置某日对账余额
  ↓
ValuationsController#create / #update
  ├─ confirm_create / confirm_update（先 dry_run 展示差异预览）
  │    └─ create_reconciliation(..., dry_run: true)
  │         └─ reconciliation_manager.reconcile_balance(..., dry_run: true)
  │              · 不 save，只构建对象用于展示 old→new 差异
  │
  └─ 正式提交（create / update action）
       └─ create_reconciliation / update_reconciliation (dry_run: false)
            └─ reconciliation_manager.reconcile_balance
                 ├─ prepare_reconciliation（同日判定逻辑同上）
                 │    ├─ existing 传了 → UPDATE 那条
                 │    ├─ 同日有记录 → UPDATE 同日那条
                 │    └─ 否则 → INSERT 新记录
                 ├─ prepared_valuation.save!（dry_run=false 才持久化）
                 └─ 返回 ReconciliationResult（包含old/new余额组件）
  ↓
Reconcileable 中捕获到 success? && !dry_run → sync_later（触发异步重算）
  ↓
重算时，ForwardCalculator 遇到该日 valuation，直接使用其金额作为期末余额
  ↓
当日的 adjustments = 期末余额 - 期初余额 - 净流量
（即：对账值与"按交易推算值"的差额，记入 adjustments）
```

**同日更新 vs 追加 在 Controller 层的差异**：

- `ValuationsController#create`：调用 `create_reconciliation`，不传 `existing_valuation_entry`。是否同日追加，完全由 `prepare_reconciliation` 内部的 `find_by(date:)` 决定。
- `ValuationsController#update`：调用 `update_reconciliation(@entry, ...)`，明确传入了 `@entry`（用户正在编辑的那条记录），**一定是 UPDATE 那条**，即使改了日期也不会创建新记录，而是改原记录的 date。

相关文件：
- [reconciliation_manager.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/account/reconciliation_manager.rb)
- [valuations_controller.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/controllers/valuations_controller.rb)
- [reconcileable.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/account/reconcileable.rb)

### 3.4 汇总：所有手工修正的追加/更新决策表

| 修正入口 | 锚点类型 | 首次调用 | 同日再调 | 跨日再调 |
|----------|----------|----------|----------|----------|
| set_opening_balance | opening_anchor | 追加1条 | 更新同一条 | 同一条（改date） |
| set_current_balance（关联账户） | current_anchor | 追加1条 | 更新同一条 | 同一条（改date） |
| set_current_balance（手工·交易调整策略） | 改 opening_anchor | 更新同一条 | 更新同一条 | 更新同一条 |
| set_current_balance（手工·价值追踪策略） | reconciliation | 追加1条 | 更新同一条 | 追加新1条 |
| create_reconciliation（新对账） | reconciliation | 追加1条 | 更新同日那条 | 追加新1条 |
| update_reconciliation（编辑已有对账） | reconciliation | — | UPDATE 传入那条 | UPDATE 传入那条 |

---

## 四、展示值更新（Display Value）— 从快照到 UI

### 4.1 展示组件与数据流

余额快照的展示由 `UI::Account::BalanceReconciliation` 组件负责，它根据账户类型组织不同的展示项目。

**展示数据的直接来源是 Balance 表，不是 account.balance 缓存**。

展示结构（以默认类型为例）：

```
Start balance         $1,000.00   ← balance.start_balance（Balance 表字段）
Net cash flow           +$50.00   ← cash_inflows - cash_outflows（Balance 表字段）
End balance          $1,050.00   ← end_balance_before_adjustments（仅 adjustments≠0 时显示）
Adjustments            -$10.00   ← cash_adjustments + non_cash_adjustments
Final balance        $1,040.00   ← balance.end_balance（Balance 表字段，始终显示）
```

其中 `end_balance_before_adjustments` 不是数据库字段，而是展示层计算出来的：
```ruby
def end_balance_before_adjustments
  balance.end_balance_money - total_adjustments
end
```

它的含义是：**如果当天没有手工对账修正，仅靠交易推算出来的余额应该是多少**。展示这个值能让用户清晰地看到"交易推算是X，手工修正了Y，最终是Z"。

### 4.2 不同账户类型的展示差异

| 账户类型 | 展示重点 |
|----------|----------|
| 储蓄/其他资产/负债 | 起始余额 + 净现金流 + 调整项 + 最终余额 |
| 信用卡 | 起始余额 + 消费 + 还款 + 调整项 + 最终余额 |
| 投资 | 起始余额 + 现金变动 + 持仓买卖 + 市价变动 + 调整项 + 最终余额 |
| 贷款 | 起始本金 + 本金变动 + 调整项 + 最终本金 |
| 房产/车辆 | 起始价值 + 价值变动 + 调整项 + 最终价值 |
| 加密货币 | 起始余额 + 买入 + 卖出 + 市价变动 + 调整项 + 最终余额 |

### 4.3 展示值的时序一致性问题

由于快照是异步重算的，UI 展示可能存在不一致窗口：

```
  T0  用户在编辑页提交新余额
        │
        ├─ set_current_balance 执行
        │    ├─ UPDATE / INSERT Valuation 锚点
        │    └─ account.update!(balance: xxx)   ← 缓存立即更新
        │
        ├─ 返回 HTTP 响应（account.balance 已显示新值）
        │
        └─ sync_later → SyncJob 入队（异步）
              │
  T1          │  Sidekiq 取出执行
              │    Materializer 重算...
              │    upsert_all balances 表...
              │    update_account_info（再次写 account.balance，值应相同）
              │    broadcast_sync_complete（Turbo 广播）
              ▼
  T2        前端收到 Turbo Stream，刷新图表/活动流
            BalanceReconciliation 组件读取最新 Balance 记录
```

**T0 ~ T2 之间**，用户看到的 `account.balance`（账户卡片上的余额）是新值，但 `BalanceReconciliation` 组件展示的每日明细可能还是旧快照（因为 balances 表还没被重算更新）。这是预期行为，不是 Bug。同步完成后 Turbo 广播会自动刷新。

相关文件：
- [balance_reconciliation.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/components/UI/account/balance_reconciliation.rb)
- [balance_reconciliation.html.erb](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/components/UI/account/balance_reconciliation.html.erb)

---

## 五、三者衔接的完整链路 — 端到端视角

### 5.1 完整流程图（带时序和决策分支）

```
┌──────────────────────────────────────────────────────────────────────────────────────┐
│                           ① 用户操作层（HTTP 请求线程）                                │
│                                                                                      │
│  ┌────────────────────┐  ┌─────────────────────┐  ┌──────────────────────────┐      │
│  │ 修改期初余额        │  │ 编辑账户改当前余额   │  │ 对账页面创建/编辑对账     │      │
│  │ set_opening_anchor │  │ set_current_balance │  │ create/update_reconciliation│    │
│  └─────────┬──────────┘  └──────────┬──────────┘  └─────────────┬────────────┘      │
│            │                        │                              │                   │
│            ▼                        ▼                              ▼                   │
│  ┌──────────────────────────────────────────────────────────────────────────────┐    │
│  │ ② 锚点写入层（同步执行，当前 HTTP 线程内）                                    │    │
│  │                                                                              │    │
│  │  OpeningBalanceManager / CurrentBalanceManager / ReconciliationManager       │    │
│  │    · 决定是 UPDATE 同日期记录 还是 INSERT 新记录（见3.4决策表）              │    │
│  │    · 保存 Valuation 锚点到 DB                                               │    │
│  │    · set_current_balance 还会立刻写 account.balance 缓存                    │    │
│  └────────────────────────────────────┬─────────────────────────────────────────┘    │
│                                       │                                              │
│                                       ▼                                              │
│  ┌──────────────────────────────────────────────────────────────────────────────┐    │
│  │ ③ 同步调度层（当前线程只入队，不执行）                                       │    │
│  │                                                                              │    │
│  │  sync_later                                                                  │    │
│  │    · 有进行中的 Sync？→ 扩展窗口复用                                         │    │
│  │    · 无 → 创建 Sync 记录 + SyncJob.perform_later（入Sidekiq队列）            │    │
│  └────────────────────────────────────┬─────────────────────────────────────────┘    │
│                                       │                                              │
│                                       │ 返回 HTTP 响应给用户                         │
│                                       │ （UI 上 account.balance 已更新）             │
└───────────────────────────────────────┼──────────────────────────────────────────────┘
                                        │
         ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─┼─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─
         " 异步边界（Sidekiq Worker 线程）              "
                                        │
                                        ▼
┌──────────────────────────────────────────────────────────────────────────────────────┐
│                        ④ Worker 执行层（Sidekiq 异步线程）                           │
│                                                                                      │
│  SyncJob#perform → Sync#perform                                                      │
│    ├─ start!（pending → syncing）                                                    │
│    │                                                                                  │
│    ├─ perform_sync 按层级分发                                                         │
│    │    ├─ Family    → 只调度子同步（PlaidItem、手工Account）                         │
│    │    ├─ PlaidItem → 拉取数据后调度每个 Account                                     │
│    │    └─ Account   → **真正的重算**                                                │
│    │         └─ Account::Syncer#perform_sync                                         │
│    │              ├─ import_market_data（汇率/股价）                                 │
│    │              └─ Balance::Materializer#materialize_balances                      │
│    │                   ├─ materialize_holdings                                       │
│    │                   ├─ calculate_balances（Forward/Reverse 计算器）                │
│    │                   │    ┌───────────────────────────────────────────────────┐   │
│    │                   │    │ 锚点日：直接用 Valuation.amount 作期末余额        │   │
│    │                   │    │ 非锚点日：期初 + 净流量 = 期末（推算）            │   │
│    │                   │    │ adjustments = 期末 - 期初 - 净流量               │   │
│    │                   │    └───────────────────────────────────────────────────┘   │
│    │                   ├─ persist_balances（upsert_all 批量写 balances 表）          │
│    │                   ├─ purge_stale_balances（清理超范围日期）                     │
│    │                   └─ update_account_info（覆盖 account.balance 缓存）           │
│    │                                                                                  │
│    └─ finalize_if_all_children_finalized                                             │
│         ├─ complete!（syncing → completed）                                          │
│         ├─ perform_post_sync（自动匹配转账）                                          │
│         └─ broadcast_sync_complete ←── Turbo 广播，通知前端刷新                      │
│                                                                                      │
└──────────────────────────────────────────────┬───────────────────────────────────────┘
                                               │
                                               ▼
┌──────────────────────────────────────────────────────────────────────────────────────┐
│                         ⑤ 展示层（收到 Turbo 广播后刷新）                              │
│                                                                                      │
│  BalanceReconciliation 组件读取 balances 表最新数据                                   │
│    · 根据账户类型选择展示模板                                                         │
│    · 有 adjustments 时多展示"修正前余额 + Adjustments"两行                           │
│    · 最终展示：Start → Flows → (Subtotal) → Adjustments → Final                     │
│                                                                                      │
└──────────────────────────────────────────────────────────────────────────────────────┘
```

### 5.2 关键衔接点深度解读

#### 衔接点 1：手工修正 → 重算（从 HTTP 线程到 Worker）

- **手工修正操作本身只修改 Valuation（估值锚点），不碰 Balance 表**
- 修正操作写锚点成功后，调用 `sync_later` 把重算任务交给 Sidekiq
- HTTP 请求线程在 `sync_later` 入队后就返回了，**不会等重算完成**
- 重算一定发生在异步 Worker 中，通过 `Sync#perform → Account::Syncer → Balance::Materializer` 这条链到达

#### 衔接点 2：锚点 → 快照（重算如何"消费"手工修正）

- 重算开始时，ForwardCalculator 读取账户所有 Valuation 记录
- 按日期遍历，遇到估值锚点日：**强制用估值作为期末余额**
- 非锚点日：用前一天的期末余额 + 当天交易净流量推算
- 锚点日与推算法不一致的差额 → 自动记入 `cash_adjustments` / `non_cash_adjustments`
- adjustments 是**重算的副产品**，不是用户直接写入的

#### 衔接点 3：快照 → 展示（从 DB 到 UI）

- BalanceReconciliation 组件直接读 `balances` 表的某一天记录
- 展示哪些项目、如何分组由 `account.accountable_type` 决定
- Adjustments 不为 0 时，会额外显示"修正前小计 + Adjustments"，让用户看到手工修正的效果
- 重算完成后，通过 `broadcast_sync_complete` 发 Turbo 广播触发前端刷新

#### 衔接点 4：两种缓存 — account.balance vs balances.end_balance

| 缓存 | 写入时机 | 用途 |
|------|----------|------|
| `account.balance` | ① set_current_balance 立即写一次；② Materializer#update_account_info 重算后再覆盖一次 | 账户列表、卡片等快速显示"当前余额"，不依赖重算完成 |
| `balances` 表（每日快照） | 只在 Materializer#persist_balances 批量写 | 历史图表、对账明细展示，是"最终真相" |

两者应该最终一致，但在 T0~T2 窗口内可能短暂不同步。这是设计上的权衡（UI 响应速度 vs 数据计算一致性）。

### 5.3 用一个具体例子串起来

**场景**：用户有一个手工 Depository 账户（现金类），已有若干交易记录，**从未对过账**。

1. **T0**：用户在编辑页将"当前余额"从 900 改为 1000。
2. **T0 + 0ms**：`CurrentBalanceManager` 判定：现金类 + 无对账记录 → **交易调整策略**。
3. **T0 + 1ms**：计算 delta=100，调用 `set_opening_balance(opening_balance + 100)`，UPDATE opening_anchor（从 1000 → 1100）。
4. **T0 + 2ms**：写 `account.balance = 1000`，返回 HTTP 响应。用户立刻在账户卡片上看到 1000。
5. **T0 + 3ms**：`sync_later` 创建 Sync 记录，SyncJob 入 Sidekiq 队列。
6. **T0 + 200ms**（Worker 中）：SyncJob 启动 → Account::Syncer#perform_sync → Materializer。
7. **T0 + 250ms**：ForwardCalculator 从 opening_anchor（=1100）开始，+ 交易 -100 = 期末余额 1000，**adjustments 全为 0**（因为差额已经"吸收"到期初余额里了）。
8. **T0 + 260ms**：upsert_all 写入 balances，Turbo 广播。
9. **T0 + 270ms**：用户页面上 BalanceReconciliation 组件刷新，展示 Start 1100 → Flow -100 → Final 1000，**没有 Adjustments 行**。

对比：如果同一账户**已有对账记录**（即走价值追踪策略），则步骤 3 会在今日追加一条 reconciliation=1000，重算时 adjustments = 1000 - 900 - 0 = +100（假设当日无交易），展示时会出现 Adjustments +100 行。

---

## 六、关键代码索引

| 模块 | 文件 |
|------|------|
| 余额模型 | [balance.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/balance.rb) |
| 余额物化器 | [materializer.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/balance/materializer.rb) |
| 正向计算器 | [forward_calculator.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/balance/forward_calculator.rb) |
| 反向计算器 | [reverse_calculator.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/balance/reverse_calculator.rb) |
| 基础计算器 | [base_calculator.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/balance/base_calculator.rb) |
| 期初余额管理 | [opening_balance_manager.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/account/opening_balance_manager.rb) |
| 当前余额管理 | [current_balance_manager.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/account/current_balance_manager.rb) |
| 对账管理 | [reconciliation_manager.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/account/reconciliation_manager.rb) |
| 账户同步器 | [account/syncer.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/account/syncer.rb) |
| 全家同步器 | [family/syncer.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/family/syncer.rb) |
| 同步模块 | [syncable.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/concerns/syncable.rb) |
| 同步状态机 | [sync.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/sync.rb) |
| 同步 Job | [sync_job.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/jobs/sync_job.rb) |
| 锚点模块 | [anchorable.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/account/anchorable.rb) |
| 对账模块 | [reconcileable.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/account/reconcileable.rb) |
| 对账控制器 | [valuations_controller.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/controllers/valuations_controller.rb) |
| 通用账户控制器 | [accountable_resource.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/controllers/concerns/accountable_resource.rb) |
| 通用条目控制器 | [entryable_resource.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/controllers/concerns/entryable_resource.rb) |
| 展示组件 | [balance_reconciliation.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/components/UI/account/balance_reconciliation.rb) |
| 估值模型 | [valuation.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/valuation.rb) |
| 账户模型 | [account.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/account.rb) |
| Entry 模型 | [entry.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/42-maybe/app/models/entry.rb) |
