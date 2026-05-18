# 资产估值数据流详解

## 概览

用户手工填入资产估值后，系统经历以下完整链路：
**用户提交估值 → 创建 Valuation 记录 → 触发后台同步 → 余额物化计算 → 更新账户余额 → 缓存失效 → 页面自动刷新展示**

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

### 3. Sync 模型 (`app/models/sync.rb`)

**状态机** (第 27-52 行)：
- `pending` → `syncing` → `completed` / `failed`
- `complete` 事件后触发 `handle_completion_transition`

**关键回调** (第 176-178 行)：
```ruby
def handle_completion_transition
  family.touch(:latest_sync_completed_at)  # 触发缓存失效
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

## 6. 视图展示层

### 6.1 资产详情页 (Account Page)

**组件**：`UI::AccountPage` (`app/components/UI/account_page.rb`)

**Turbo Streams 实时更新**：
- 视图订阅账户的 Turbo Stream 频道 (`account_page.html.erb` 第 1 行)：
  ```erb
  <%= turbo_stream_from account %>
  ```
- 同步完成后通过 `broadcast_sync_complete` 推送更新
- 整个页面在 `turbo_frame_tag` 内，可被局部替换

**展示的数据来源**：
- 账户余额：`account.balance`（从 `accounts` 表读取，已在同步时更新）
- 历史图表：`account.sparkline_series`（基于 `balances` 表计算）
- 估值记录：`account.entries.valuations`（从 `entries` + `valuations` 表读取）

### 6.2 净资产页 (Dashboard / Balance Sheet)

**BalanceSheet 聚合** (`app/models/balance_sheet.rb`)：
```ruby
def assets
  @assets ||= ClassificationGroup.new(
    classification: "asset",
    currency: family.currency,
    accounts: account_totals.asset_accounts  # 缓存的账户汇总数据
  )
end

def net_worth
  assets.total - liabilities.total
end

def net_worth_series(period: Period.last_30_days)
  net_worth_series_builder.net_worth_series(period: period)  # 缓存的净资产系列
end
```

**视图**：
- `_net_worth_chart.html.erb`：展示净资产趋势图
- `_balance_sheet.html.erb`：展示资产/负债分类汇总及各账户明细

---

## 完整数据流时序图

```
用户操作
   ↓
[ValuationsController]
   ↓ create_reconciliation / update_reconciliation
[Account::Reconcileable]
   ↓
[Account::ReconciliationManager]
   ├─ 创建/更新 Valuation 记录
   └─ 调用 sync_later
         ↓
[Syncable.sync_later]
   ├─ 创建 Sync 记录
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
   └─ handle_completion_transition
       └─ family.touch(:latest_sync_completed_at)
             ↓
[缓存失效]
   ├─ AccountTotals 缓存键变化 → 重新计算
   └─ NetWorthSeriesBuilder 缓存键变化 → 重新计算
         ↓
[页面展示]
   ├─ 资产详情页：Turbo Stream 推送更新
   └─ 净资产页：下次刷新时读取新缓存
```

---

## 关键设计要点

1.  **异步解耦**：用户提交后立即返回，余额计算在后台异步执行
2.  **幂等设计**：重复的 sync 会被合并（扩展窗口），避免重复计算
3.  **缓存失效**：通过 `latest_sync_completed_at` 时间戳实现优雅的缓存失效
4.  **实时推送**：Turbo Streams 实现资产详情页的实时更新
5.  **余额物化**：每日余额预计算，避免页面加载时的昂贵计算
6.  **多态设计**：Valuation 通过 Entry 包装，与 Transaction、Trade 共享同一入口
