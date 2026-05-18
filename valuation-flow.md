# 资产估值数据流详解

## 概览

用户手工填入资产估值后，系统经历以下完整链路：
**用户提交估值 → 创建 Valuation 记录 → 触发后台同步 → 余额物化计算 → 更新账户余额 → Turbo Streams 广播 → 所有页面实时局部刷新**

---

## 1. 用户输入层：表单提交与控制器处理

### 1.1 ValuationsController (`app/controllers/valuations_controller.rb`)

**创建估值 (create)** - 第 32-48 行：
```ruby
def create
  account = Current.family.accounts.find(params.dig(:entry, :account_id))
  result = account.create_reconciliation(
    balance: entry_params[:amount],
    date: entry_params[:date],
  )
  # ... 重定向响应
end
```

**更新估值 (update)** - 第 51-83 行：
```ruby
def update
  result = @entry.account.update_reconciliation(
    @entry,
    balance: entry_params[:amount],
    date: entry_params[:date],
  )
  # ... 响应处理
end
```

**确认页面 (confirm_create/confirm_update)**：
- 先执行 `dry_run: true` 预览变化
- 展示旧余额 vs 新余额的对比

---

## 2. 业务逻辑层：对账管理 (ReconciliationManager)

### 2.1 Reconcileable 模块 (`app/models/account/reconcileable.rb`)

`Account` model 包含 `Reconcileable` concern，提供两个核心方法：
- `create_reconciliation(balance:, date:, dry_run: false)`
- `update_reconciliation(existing_valuation_entry, balance:, date:, dry_run: false)`

成功后调用 `sync_later` 触发后台同步。

### 2.2 Account::ReconciliationManager (`app/models/account/reconciliation_manager.rb`)

**核心方法 `reconcile_balance`** (第 9-30 行)：
1.  获取旧余额组件
2.  准备 Valuation 记录（新建或更新）
3.  非 dry_run 时保存记录
4.  返回 ReconciliationResult（包含新旧余额对比）

**Valuation 记录结构** (`app/models/valuation.rb`)：
- 包含 `Entry` 包装层（多态关联）
- `kind` 枚举：`reconciliation`（用户对账）、`opening_anchor`、`current_anchor`
- 每个账户每天只能有一条 reconciliation 估值（唯一性验证在 `Entry` model）

---

## 3. 后台同步层：Sync + Job

### 3.1 Syncable 模块 (`app/models/concerns/syncable.rb`)

`sync_later` 方法 (第 14-35 行)：
```ruby
def sync_later(parent_sync: nil, window_start_date: nil, window_end_date: nil)
  Sync.transaction do
    with_lock do
      sync = self.syncs.incomplete.first
      if sync
        sync.expand_window_if_needed(window_start_date, window_end_date)
      else
        sync = self.syncs.create!(...)
        SyncJob.perform_later(sync)
      end
    end
  end
end
```

**去重机制**：如果已有未完成的 sync，直接扩展时间窗口，避免重复创建。

### 3.2 SyncJob (`app/jobs/sync_job.rb`)

```ruby
def perform(sync)
  sync.perform
end
```

### 3.3 Sync 模型 (`app/models/sync.rb`)

**状态机** (第 27-52 行)：
- `pending` → `syncing` → `completed` / `failed`
- `complete` 事件后触发 `handle_completion_transition`

**关键回调** (第 176-178 行)：
```ruby
def handle_completion_transition
  family.touch(:latest_sync_completed_at)  # 触发缓存失效
end
```

**Post-Sync 钩子** (第 142-149 行)：
```ruby
def perform_post_sync
  syncable.perform_post_sync
  syncable.broadcast_sync_complete  # 🔴 关键：触发 Turbo Streams 广播
end
```

---

## 4. 余额计算层：Materializer

### 4.1 Account::Syncer (`app/models/account/syncer.rb`)

`perform_sync` 方法 (第 8-12 行)：
```ruby
def perform_sync(sync)
  import_market_data           # 导入市场数据（汇率、证券价格）
  materialize_balances         # 计算并物化余额
end
```

### 4.2 Balance::Materializer (`app/models/balance/materializer.rb`)

`materialize_balances` 方法 (第 9-23 行)：
1.  **物化持仓**：`Holding::Materializer`（投资账户）
2.  **计算余额**：根据策略选择计算器
    - 手动账户：`Balance::ForwardCalculator`（正向计算）
    - 同步账户：`Balance::ReverseCalculator`（反向计算）
3.  **持久化余额**：批量 upsert 到 `balances` 表
4.  **清理陈旧余额**：删除时间窗口外的数据
5.  **更新账户信息** (第 30-52 行)：

```ruby
def update_account_info
  current_balance = account.balances
    .where(currency: account.currency)
    .order(date: :desc)
    .first

  account.update!(
    balance: calculated_balance,           # 更新账户总余额
    cash_balance: calculated_cash_balance  # 更新现金余额
  )
end
```

**Balance 表结构** (`app/models/balance.rb`)：
- 每日一条记录，包含：
  - `start_balance` / `end_balance`：期初/期末余额
  - `cash_inflows` / `cash_outflows`：现金流入/流出
  - `non_cash_inflows` / `non_cash_outflows`：非现金流入/流出
  - `net_market_flows`：市场波动
  - `cash_adjustments` / `non_cash_adjustments`：估值调整（包括用户输入的估值）

---

## 5. 缓存失效层

### 5.1 Family 缓存键 (`app/models/family.rb` 第 96-107 行)

```ruby
def build_cache_key(key, invalidate_on_data_updates: false)
  data_invalidation_key = invalidate_on_data_updates ? latest_sync_completed_at : nil
  [
    id,
    key,
    data_invalidation_key,      # 关键：同步完成时间戳
    accounts.maximum(:updated_at)
  ].compact.join("_")
end
```

**关键机制**：
- `latest_sync_completed_at` 在每次 sync 完成时被 `touch` 更新
- 所有需要随数据变化失效的缓存都包含这个时间戳
- 时间戳一变，缓存键就变，旧缓存自动失效

### 5.2 缓存使用点

**BalanceSheet::AccountTotals** (`app/models/balance_sheet/account_totals.rb` 第 47-62 行)：
```ruby
Rails.cache.fetch(cache_key) do
  visible_accounts.joins(...).select(...).to_a
end
```

**BalanceSheet::NetWorthSeriesBuilder** (`app/models/balance_sheet/net_worth_series_builder.rb` 第 6-17 行)：
```ruby
Rails.cache.fetch(cache_key(period)) do
  Balance::ChartSeriesBuilder.new(...).balance_series
end
```

---

## 6. Turbo Streams 广播层

### 6.1 频道订阅机制

**全局订阅** (`app/views/layouts/shared/_htmldoc.html.erb` 第 29-31 行)：
```erb
<% if Current.family %>
  <%= turbo_stream_from Current.family %>  # 🔴 所有页面订阅家庭级频道
<% end %>
```

**账户级订阅** (`app/components/UI/account_page.html.erb` 第 1 行)：
```erb
<%= turbo_stream_from account %>  # 资产详情页额外订阅账户级频道
```

### 6.2 SyncCompleteEvent 广播器

Syncable 模块通过 `sync_broadcaster` 方法获取对应的广播器：
```ruby
def sync_broadcaster
  self.class::SyncCompleteEvent.new(self)
end
```

### 6.3 账户级广播 (`app/models/account/sync_complete_event.rb`)

```ruby
def broadcast
  # 1. 更新账户列表中的账户行（发送到 family 频道）
  account.broadcast_replace_to(
    account.family,
    target: "account_#{account.id}",
    partial: "accounts/account",
    locals: { account: account }
  )

  # 2. 更新侧边栏分组（发送到 family 频道）
  sidebar_targets.each do |(tab, mobile_flag)|
    account.broadcast_replace_to(...)
  end

  # 🔴 3. 手动账户触发家庭级广播（因为没有 Plaid 同步）
  unless account.linked?
    account.family.broadcast_sync_complete  # 级联触发家庭级广播
  end

  # 🔴 4. 刷新资产详情页（发送到 account 频道）
  # 使用 Turbo 8 Page Refresh 机制，客户端自动 morph 更新页面
  account.broadcast_refresh
end
```

**⚠️ 重要修正**：`account.broadcast_refresh` 是 Turbo Rails 8+ 内置方法，来自 `Turbo::Broadcastable` 模块（自动包含在所有 ActiveRecord 模型中）。它的实现是：
```ruby
def broadcast_refresh
  broadcast_refresh_to self  # 向 account 频道发送 refresh 动作
end
```

这不是调用 `UI::AccountPage#broadcast_refresh!`，而是发送一个 Turbo Page Refresh 事件，客户端使用 morphing 技术智能更新页面。

### 6.4 家庭级广播 (`app/models/family/sync_complete_event.rb`)

```ruby
def broadcast
  # 🔴 直接替换净资产图表（所有打开的页面都会收到）
  family.broadcast_replace(
    target: "net-worth-chart",
    partial: "pages/dashboard/net_worth_chart",
    locals: { balance_sheet: family.balance_sheet, period: Period.last_30_days }
  )

  # 🔴 直接替换资产总览表（所有打开的页面都会收到）
  family.broadcast_replace(
    target: "balance-sheet",
    partial: "pages/dashboard/balance_sheet",
    locals: { balance_sheet: family.balance_sheet }
  )
end
```

---

## 7. 视图展示层

### 7.1 资产详情页 (Account Page)

**组件**：`UI::AccountPage` (`app/components/UI/account_page.rb`)

**实时刷新机制（修正后）**：
- 页面订阅 `account` 频道（独立于家庭频道）
- `account.broadcast_refresh` 发送 Turbo Page Refresh 事件到 `account` 频道
- 客户端收到后，使用 **morphing 技术**智能更新页面（保留滚动位置、表单状态等）
- 这不是替换特定 turbo_frame，而是整页差异更新

**`UI::AccountPage#broadcast_refresh!` 方法说明**：
- 这是一个自定义方法，但**在同步流程中没有被调用**
- 它会替换整个 `#account_123_container` turbo_frame
- 可能用于其他场景（如手动触发刷新）

**展示的数据来源**：
- 账户余额：`account.balance`（从 `accounts` 表读取，已在同步时更新）
- 历史图表：`account.sparkline_series`（基于 `balances` 表计算）
- 估值记录：`account.entries.valuations`（从 `entries` + `valuations` 表读取）

### 7.2 净资产页 (Dashboard / Balance Sheet)

**BalanceSheet 聚合** (`app/models/balance_sheet.rb`)：
```ruby
def assets
  @assets ||= ClassificationGroup.new(
    classification: "asset",
    currency: family.currency,
    accounts: account_totals.asset_accounts
  )
end

def net_worth
  assets.total - liabilities.total
end

def net_worth_series(period: Period.last_30_days)
  net_worth_series_builder.net_worth_series(period: period)
end
```

**实时刷新机制**：
- 所有页面订阅 `Current.family` 频道
- `Family::SyncCompleteEvent` 直接广播替换两个 DOM 元素：
  - `<div id="net-worth-chart">`：净资产趋势图
  - `<div id="balance-sheet">`：资产/负债总览表
- **无需刷新页面**，Turbo 自动处理局部替换

**视图**：
- `_net_worth_chart.html.erb`：展示净资产趋势图
- `_balance_sheet.html.erb`：展示资产/负债分类汇总及各账户明细

---

## 完整数据流时序图

```
用户操作（输入估值）
   ↓
[ValuationsController]
   ↓ create_reconciliation / update_reconciliation
[Account::Reconcileable]
   ↓
[Account::ReconciliationManager]
   ├─ 创建/更新 Valuation 记录（entries + valuations 表）
   └─ 调用 sync_later
         ↓
[Syncable.sync_later]
   ├─ 创建 Sync 记录（syncs 表）
   └─ 入队 SyncJob
         ↓
[后台 Worker]
   ↓ SyncJob.perform
[Sync#perform]
   ├─ 状态变为 syncing
   ├─ 调用 Account::Syncer#perform_sync
   │   ├─ import_market_data (汇率/证券价格)
   │   └─ materialize_balances
   │       ├─ Balance::Materializer
   │       │   ├─ 计算每日余额 (Forward/Reverse Calculator)
   │       │   ├─ 批量 upsert 到 balances 表
   │       │   └─ 更新 account.balance / cash_balance
   │       └─ 账户余额更新 → account.updated_at 变化
   ├─ 状态变为 completed
   ├─ handle_completion_transition
   │   └─ family.touch(:latest_sync_completed_at)  # 缓存失效
   └─ perform_post_sync
       ├─ syncable.perform_post_sync
       └─ syncable.broadcast_sync_complete  # 🔴 触发广播
             ↓
[Account::SyncCompleteEvent#broadcast]
   ├─ 广播替换账户行（到 family 频道）
   ├─ 广播替换侧边栏分组（到 family 频道）
   ├─ 🔴 手动账户额外触发：account.family.broadcast_sync_complete
   │   └─ [Family::SyncCompleteEvent#broadcast]
   │       ├─ 广播替换 <div id="net-worth-chart">（到 family 频道）
   │       └─ 广播替换 <div id="balance-sheet">（到 family 频道）
   └─ 🔴 调用 account.broadcast_refresh（Turbo 8 内置方法）
       └─ 发送 Page Refresh 事件（到 account 频道）
             ↓
[前端 Turbo 自动处理]
   ├─ 资产详情页：接收 account 频道 refresh 事件 → morph 整页更新
   └─ 所有打开的页面：接收 family 频道消息 → 局部替换净资产图和资产总览
```

---

## 关键设计要点

### 广播链路分层设计

1.  **账户级广播**（`Account::SyncCompleteEvent`）：
    - 刷新资产详情页（仅订阅该账户频道的页面）
    - 更新资产列表中的账户行（所有页面）
    - 更新侧边栏分组（所有页面）
    - 使用 **Turbo Page Refresh**（morphing）更新资产详情页

2.  **家庭级广播**（`Family::SyncCompleteEvent`）：
    - 刷新净资产图表（所有页面）
    - 刷新资产总览表（所有页面）
    - 手动账户会从账户级广播级联触发家庭级广播
    - 使用 **broadcast_replace** 替换特定 DOM 元素

3.  **Plaid 链接账户**：
    - 由 `PlaidItem::SyncCompleteEvent` 触发家庭级广播
    - 不需要在账户级广播中重复触发

### Turbo 8 Page Refresh vs broadcast_replace

| 特性 | `broadcast_refresh` | `broadcast_replace` |
|------|---------------------|---------------------|
| 触发方式 | `account.broadcast_refresh` | `family.broadcast_replace(target: ...)` |
| 更新范围 | 整页 morphing 差异更新 | 替换指定 target DOM 元素 |
| 状态保留 | 保留滚动位置、表单状态 | 仅替换目标元素内容 |
| 适用场景 | 复杂页面（如资产详情页） | 特定组件（如净资产图） |
| 实现复杂度 | 低（自动处理） | 高（需指定 target 和 partial） |

### 核心技术设计

1.  **异步解耦**：用户提交后立即返回，余额计算在后台异步执行
2.  **幂等设计**：重复的 sync 会被合并（扩展窗口），避免重复计算
3.  **缓存失效**：通过 `latest_sync_completed_at` 时间戳实现优雅的缓存失效
4.  **实时推送**：Turbo Streams 实现全站无刷新更新
5.  **余额物化**：每日余额预计算，避免页面加载时的昂贵计算
6.  **多态设计**：Valuation 通过 Entry 包装，与 Transaction、Trade 共享同一入口
7.  **频道分层**：家庭频道 + 账户频道的双层订阅，精确控制更新范围
8.  **Turbo 8 Page Refresh**：使用 morphing 技术实现智能页面更新，保留用户状态
