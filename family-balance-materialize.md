# 家庭账户余额物化流程报告

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

**关键方法**：
- `perform` (第60-80行)：执行同步主流程
- `finalize_if_all_children_finalized` (第83-104行)：处理父子同步关系
- `perform_post_sync` (第142-149行)：同步完成后的后置处理

## 5. 余额计算与物化核心

### 5.1 Account::Syncer

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

### 5.2 Balance::Materializer

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

### 5.3 两种计算策略

#### 5.3.1 正向计算 (Forward Calculator)

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

#### 5.3.2 反向计算 (Reverse Calculator)

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

### 5.4 BaseCalculator 核心方法

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

### 5.5 Balance::SyncCache

`app/models/balance/sync_cache.rb` 提供计算所需的缓存数据：

```ruby
def get_valuation(date)    # 获取当日估值
def get_holdings(date)     # 获取当日持仓
def get_entries(date)      # 获取当日流水
```

**特点**：
- 自动处理货币转换
- 使用内存缓存避免重复查询

## 6. 数据物化到 balances 表

### 6.1 Balance 模型结构

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

### 6.2 唯一索引

持久化时使用 `unique_by: %i[account_id date currency]` 确保：
- 每个账户、每天、每种货币只有一条余额记录

## 7. 同步完成与 UI 广播

### 7.1 同步完成后的事件链

```ruby
# app/models/sync.rb:142-149
def perform_post_sync
  Rails.logger.info("Performing post-sync for #{syncable_type} (#{syncable.id})")
  syncable.perform_post_sync       # 1. 执行后置同步
  syncable.broadcast_sync_complete # 2. 广播同步完成
end
```

### 7.2 Account::SyncCompleteEvent

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

### 7.3 Family::SyncCompleteEvent

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

## 8. 图表数据消费

### 8.1 账户级图表

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

### 8.2 家庭净值图表

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

### 8.3 Balance::ChartSeriesBuilder

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

## 9. 完整时序图

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

## 10. 关键组件速查表

| 组件 | 文件位置 | 职责 |
|------|----------|------|
| Entry | `app/models/entry.rb` | 流水模型，触发同步 |
| Syncable | `app/models/concerns/syncable.rb` | 同步能力混入 |
| Sync | `app/models/sync.rb` | 同步记录与状态管理 |
| SyncJob | `app/jobs/sync_job.rb` | 后台执行同步 |
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

## 11. 设计要点总结

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
