# 证券交易录入主流程分析

## 一、整体架构概览

证券交易录入流程涉及从表单提交到持仓刷新的完整链路。核心实体包括：

- **Trade** - 交易记录（买卖）
- **Transaction** - 普通交易（利息、分红、费用等）
- **Entry** - 账务条目（所有交易的容器）
- **Holding** - 持仓快照（按日期的持仓）
- **Balance** - 账户余额快照
- **Account** - 账户（聚合根）

```
表单提交 → 控制器 → CreateForm → Entry 构造 → 异步同步 → 持仓计算 → 余额刷新
```

Entry 支持三种 entryable 类型（通过 `delegated_type` 多态）：
- `Trade` - 证券买卖
- `Transaction` - 利息、分红、费用、转账等
- `Valuation` - 估值锚点

---

## 二、表单校验与领域对象构造

### 2.1 控制器层 (`TradesController`)

**文件位置**: `app/controllers/trades_controller.rb`

```ruby
def create
  @account = Current.family.accounts.find(params[:account_id])
  @model = Trade::CreateForm.new(create_params.merge(account: @account)).create
  # ...
end
```

**关键流程**:
1. 接收表单参数（`:date`, `:amount`, `:currency`, `:qty`, `:price`, `:ticker`, `:type` 等）
2. 委托给 `Trade::CreateForm` 进行统一创建
3. 根据 `persisted?` 判断是否成功
4. 成功后重定向或返回

### 2.2 表单对象层 (`Trade::CreateForm`)

**文件位置**: `app/models/trade/create_form.rb`

支持多种交易类型的分发器模式：

```ruby
def create
  case type
  when "buy", "sell"             → create_trade
  when "interest"                → create_interest_income
  when "deposit", "withdrawal"   → create_transfer
  end
end
```

#### 2.2.1 买入/卖出交易创建 (`create_trade`)

```ruby
def create_trade
  signed_qty = type == "sell" ? -qty.to_d : qty.to_d
  signed_amount = signed_qty * price.to_d

  trade_entry = account.entries.new(
    name: Trade.build_name(type, qty, security.ticker),
    date: date,
    amount: signed_amount,
    currency: currency,
    entryable: Trade.new(
      qty: signed_qty,
      price: price,
      currency: currency,
      security: security
    )
  )

  if trade_entry.save
    trade_entry.lock_saved_attributes!
    account.sync_later
  end
end
```

**数据模型关系**:
- **Entry** (账务条目): 包含 `name`, `date`, `amount`, `currency`
- **Trade** (交易明细): 包含 `qty`, `price`, `security_id`
- 两者通过 `delegated_type` 多态关联

#### 2.2.2 利息收入 (`create_interest_income`)

```ruby
def create_interest_income
  signed_amount = amount.to_d * -1

  entry = account.entries.build(
    name: "Interest payment",
    date: date,
    amount: signed_amount,
    currency: currency,
    entryable: Transaction.new
  )

  if entry.save
    entry.lock_saved_attributes!
    account.sync_later
  end
end
```

**特点**:
- 创建 `Transaction` 类型的 Entry（非 Trade）
- 利息是收入，所以 `amount` 取负（资产账户负金额表示流入）
- 不影响持仓，只影响现金余额

#### 2.2.3 存款/取款 (`create_transfer`)

```ruby
def create_transfer
  if transfer_account_id.present?
    # 关联账户转账
    from_account_id = type == "withdrawal" ? account.id : transfer_account_id
    to_account_id = type == "withdrawal" ? transfer_account_id : account.id

    Transfer::Creator.new(
      family: account.family,
      source_account_id: from_account_id,
      destination_account_id: to_account_id,
      date: date,
      amount: amount
    ).create
  else
    # 无关联账户，作为普通交易处理
    create_unlinked_transfer
  end
end
```

**关联账户转账** (`Transfer::Creator`):
- 创建 `Transfer` 记录，关联两个 `Transaction`
- 源账户：流出（正金额）
- 目标账户：流入（负金额）
- 两个账户都会触发同步

**无关联账户转账** (`create_unlinked_transfer`):
- 创建 `Transaction` 类型的 Entry
- 存款：负金额（流入）
- 取款：正金额（流出）

#### 2.2.4 证券解析 (`Security::Resolver`)

```ruby
def security
  ticker_symbol, exchange_operating_mic = ticker.present? ? ticker.split("|") : [ manual_ticker, nil ]
  Security::Resolver.new(ticker_symbol, exchange_operating_mic: exchange_operating_mic).resolve
end
```

支持两种模式：
- 在线 ticker (从 Synth 提供商查询)
- 手动 ticker (不获取价格)

### 2.3 领域模型校验

**Trade 模型** (`app/models/trade.rb`):

```ruby
validates :qty, presence: true
validates :price, :currency, presence: true
```

**Entry 模型** (`app/models/entry.rb`):

```ruby
validates :date, :name, :amount, :currency, presence: true
validates :date, comparison: { greater_than: -> { min_supported_date } }
```

---

## 三、持仓与余额刷新流程

### 3.1 同步触发机制

**Entry 保存后触发同步**：

```ruby
# trade/create_form.rb
trade_entry.lock_saved_attributes!
account.sync_later
```

**Syncable Concern** (`app/models/concerns/syncable.rb`):

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

**去重机制**:
- 如果已有未完成的同步，则扩展时间窗口
- 避免重复排队

### 3.2 账户同步器 (`Account::Syncer`)

**文件位置**: `app/models/account/syncer.rb`

```ruby
def perform_sync(sync)
  import_market_data          # 1. 导入市场数据（汇率、证券价格）
  materialize_balances          # 2. 物化余额（含持仓）
end
```

### 3.3 余额物化器 (`Balance::Materializer`)

**文件位置**: `app/models/balance/materializer.rb`

```ruby
def materialize_balances
  Balance.transaction do
    materialize_holdings    # 1. 物化持仓
    calculate_balances     # 2. 计算余额
    persist_balances       # 3. 持久化余额
    purge_stale_balances      # 4. 清理过期余额

    # ⚠️  关键：只有正向策略才更新账户总额
    if strategy == :forward
      update_account_info
    end
  end
end
```

#### 3.3.1 账户总额更新条件（重要修正）

```ruby
def update_account_info
  current_balance = account.balances
    .where(currency: account.currency)
    .order(date: :desc)
    .first

  if current_balance
    calculated_balance = current_balance.end_balance
    calculated_cash_balance = current_balance.end_cash_balance
  end

  account.update!(
    balance: calculated_balance,
    cash_balance: calculated_cash_balance
  )
end
```

**更新条件**:
- ✅ **正向策略（forward）**: 会更新 `Account.balance` 和 `Account.cash_balance`
- ❌ **反向策略（reverse）**: 不会更新账户总额

**设计原因**:
- 正向策略用于手动账户，所有交易已知，计算结果可信
- 反向策略用于同步账户（如 Plaid），以提供商的当前余额为准，不从历史反推总额

---

## 四、持仓计算核心逻辑

### 4.1 持仓物化器 (`Holding::Materializer`)

**文件位置**: `app/models/holding/materializer.rb`

```ruby
def materialize_holdings
  calculate_holdings      # 计算持仓
  persist_holdings    # 批量 upsert
  purge_stale_holdings # 清理过期持仓
end
```

**持久化策略**:
- 使用 `upsert_all` 批量写入
- 唯一键: `[account_id, security_id, date, currency]`

### 4.2 两种计算策略

#### 4.2.1 正向计算 (`ForwardCalculator`) - 手动账户

**文件位置**: `app/models/holding/forward_calculator.rb`

```ruby
def calculate
  current_portfolio = generate_starting_portfolio  # 初始空持仓
  holdings = []

  account.start_date.upto(Date.current).each do |date|
    trades = portfolio_cache.get_trades(date: date)
    next_portfolio = transform_portfolio(current_portfolio, trades, direction: :forward)
    holdings += build_holdings(next_portfolio, date)
    current_portfolio = next_portfolio
  end

  Holding.gapfill(holdings)
end
```

**核心算法**:
1. 从账户开始日期到今天逐日遍历
2. 每天应用当日交易调整持仓数量
3. 为每只证券生成当日持仓记录
4. 最后进行缺口填充（LOCF - Last Observation Carried Forward）

#### 4.2.2 反向计算 (`ReverseCalculator`) - 同步账户

**文件位置**: `app/models/holding/reverse_calculator.rb`

```ruby
def calculate
  current_portfolio = portfolio_snapshot.to_h  # 从当前持仓快照开始
  holdings = []

  Date.current.downto(account.start_date).each do |date|
    today_trades = portfolio_cache.get_trades(date: date)
    previous_portfolio = transform_portfolio(current_portfolio, today_trades, direction: :reverse)
    holdings += build_holdings(current_portfolio, date)
    current_portfolio = previous_portfolio
  end

  Holding.gapfill(holdings)
end
```

**适用场景**:
- Plaid 等同步账户，只提供当前持仓但不提供完整历史交易
- 从当前持仓反推历史持仓

### 4.3 投资组合变换 (`transform_portfolio`)

两种计算器共享的核心逻辑：

```ruby
def transform_portfolio(previous_portfolio, trade_entries, direction: :forward)
  new_quantities = previous_portfolio.dup
  trade_entries.each do |trade_entry|
    trade = trade_entry.entryable
    security_id = trade.security_id
    qty_change = trade.qty
    qty_change = qty_change * -1 if direction == :reverse  # 反向时取反
    new_quantities[security_id] = (new_quantities[security_id] || 0) + qty_change
  end
  new_quantities
end
```

### 4.4 持仓缓存 (`Holding::PortfolioCache`)

**文件位置**: `app/models/holding/portfolio_cache.rb`

**价格优先级**:
```
1. DB 价格（从提供商同步）→ 优先级 1
2. 交易价格 → 优先级 2
3. 持仓价格 → 优先级 3
```

```ruby
def get_price(security_id, date, source: nil)
  # 按优先级选择价格，最低优先级的价格优先使用
  price = security[:prices].select { |p| p.price.date == date }.min_by(&:priority)&.price
  # 自动转换为账户货币
end
```

### 4.5 缺口填充 (`Gapfillable`)

**文件位置**: `app/models/holding/gapfillable.rb`

```ruby
def gapfill(holdings)
  # 对每只证券，从第一个有记录的日期到今天
  # 如果某天没有持仓记录，则使用前一天的数据填充（LOCF）
end
```

**作用**: 确保净值历史曲线连续，无断档

---

## 五、非买卖交易接入链路详解

### 5.1 交易类型与 Entry 类型映射

| 表单类型 | Entry 类型 | 影响现金 | 影响持仓 |
|---------|-----------|---------|---------|
| buy/sell | Trade | ✅ | ✅ |
| interest | Transaction | ✅ | ❌ |
| deposit/withdrawal (无对方账户) | Transaction | ✅ | ❌ |
| deposit/withdrawal (有对方账户) | Transfer (两个 Transaction) | ✅ (双方) | ❌ |

### 5.2 余额计算中的现金流分类

**文件位置**: `app/models/balance/base_calculator.rb`

```ruby
def flows_for_date(date)
  entries = sync_cache.get_entries(date)  # 只取 transaction? 和 trade? 类型

  txn_inflow_sum = entries.select { |e| e.amount < 0 && e.transaction? }.sum(&:amount)
  txn_outflow_sum = entries.select { |e| e.amount >= 0 && e.transaction? }.sum(&:amount)

  trade_cash_inflow_sum = entries.select { |e| e.amount < 0 && e.trade? }.sum(&:amount)
  trade_cash_outflow_sum = entries.select { |e| e.amount >= 0 && e.trade? }.sum(&:amount)

  if account.balance_type != :non_cash
    cash_inflows = txn_inflow_sum.abs + trade_cash_inflow_sum.abs
    cash_outflows = txn_outflow_sum + trade_cash_outflow_sum

    # 交易对非现金的影响是反向的
    # 买入（现金流出）= 持仓增加（非现金流入）
    # 卖出（现金流入）= 持仓减少（非现金流出）
    non_cash_outflows = trade_cash_inflow_sum.abs  # 买入对应持仓增加
    non_cash_inflows = trade_cash_outflow_sum      # 卖出对应持仓减少
  end
  # ...
end
```

**关键逻辑**:
- `Transaction` 类型只影响现金流
- `Trade` 类型同时影响现金流和非现金流（持仓），且方向相反
- 利息、分红等非买卖交易作为 `Transaction` 处理，只影响现金

### 5.3 利息/分红接入路径

```
用户选择 "interest" 类型
    ↓
Trade::CreateForm#create_interest_income
    ↓
创建 Entry (entryable: Transaction.new)
    ↓
entry.save → account.sync_later
    ↓
Balance::ForwardCalculator#calculate
    ↓
flows_for_date(date) 识别为 transaction?
    ↓
计入 cash_inflows / cash_outflows
    ↓
derive_cash_balance 更新现金余额
    ↓
（不影响持仓，non_cash 无变化）
```

### 5.4 转账接入路径（关联账户）

```
用户选择 "deposit"/"withdrawal" 并指定对方账户
    ↓
Trade::CreateForm#create_transfer
    ↓
Transfer::Creator#create
    ├─ 创建 Transaction (源账户，流出，正金额)
    ├─ 创建 Transaction (目标账户，流入，负金额)
    └─ 创建 Transfer 关联两个 Transaction
    ↓
source_account.sync_later
destination_account.sync_later
    ↓
两个账户各自独立计算余额
```

### 5.5 存款/取款接入路径（无关联账户）

```
用户选择 "deposit"/"withdrawal" 不指定对方账户
    ↓
Trade::CreateForm#create_unlinked_transfer
    ↓
创建 Entry (entryable: Transaction.new)
    ↓
entry.save → account.sync_later
    ↓
Balance 计算时识别为 transaction?
    ↓
计入 cash_inflows / cash_outflows
```

---

## 六、持仓快照与历史交易的承接关系

### 6.1 完整数据流向图

```
表单提交
    ↓
Trade::CreateForm (根据 type 分发)
    ├─ buy/sell → Entry + Trade
    ├─ interest → Entry + Transaction
    └─ transfer → Transfer (2x Transaction)
    ↓
entry.save → account.sync_later
    ↓
SyncJob 异步执行
    ↓
Account::Syncer#perform_sync
    ├─ import_market_data (汇率、证券价格)
    └─ materialize_balances
        ├─ materialize_holdings (持仓计算)
        │   ├─ ForwardCalculator / ReverseCalculator
        │   │   ├─ PortfolioCache (预加载所有 trades + prices)
        │   │   ├─ 逐日遍历，应用当日 trades
        │   │   ├─ build_holdings (生成持仓快照)
        │   │   └─ Holding.gapfill (缺口填充)
        │   └─ persist_holdings (批量 upsert)
        ├─ calculate_balances (余额计算)
        │   └─ flows_for_date (区分 transaction/trade 现金流)
        ├─ persist_balances
        └─ update_account_info (仅 forward 策略)
            ↓
Account.balance / cash_balance 更新
```

### 6.2 持仓快照 (Holding) 结构

| 字段 | 说明 |
|------|------|
| `date` | 快照日期 |
| `security_id` | 证券ID |
| `qty` | 持仓数量 |
| `price` | 当日价格 |
| `amount` | 持仓市值 (qty * price) |
| `currency` | 货币 |

### 6.3 历史交易 → 持仓快照 的承接

#### 承接点 1: PortfolioCache 预加载

```ruby
def trades
  @trades ||= account.entries.includes(entryable: :security).trades.chronological.to_a
end
```

- 一次性加载账户所有交易，按日期排序
- 后续计算直接从缓存读取，避免 N+1 查询

#### 承接点 2: 逐日交易应用

```ruby
account.start_date.upto(Date.current).each do |date|
  trades = portfolio_cache.get_trades(date: date)  # 取出当日交易
  next_portfolio = transform_portfolio(current_portfolio, trades, direction: :forward)
  # ...
end
```

- 每个日期的交易增量应用到持仓
- 交易的 `qty` 正负直接影响持仓数量

#### 承接点 3: 价格继承

```ruby
# 交易价格作为价格源之一
trade_prices = trades.map do |trade|
  PriceWithPriority.new(
    price: Security::Price.new(...),
    priority: 2,  # 中等优先级
    source: "trade"
  )
end
```

- 交易本身的价格会被用作价格回退源
- 确保即使没有市场数据，也能计算持仓市值

### 6.4 持仓快照 → 历史交易 的反向查询

Holding 提供回溯能力：

```ruby
# holding.rb:50-52
def trades
  account.entries.where(entryable: account.trades.where(security: security)).reverse_chronological
end
```

可以从任一持仓快照反查影响该证券的所有历史交易。

### 6.5 持仓快照的时间连续性

通过 `Gapfillable` 模块确保：
- 即使某天没有交易，也会生成持仓记录
- 使用前一天的数量和价格填充
- 保证历史曲线完整

---

## 七、完整调用链总结

### 7.1 买入交易示例

```
1. 用户提交买入表单
   ↓
2. TradesController#create
   ↓
3. Trade::CreateForm#create_trade
   ├─ Security::Resolver#resolve (解析证券)
   ├─ Entry.create (创建账务条目，amount 为负)
   ├─ Trade.create (创建交易明细，qty 为正)
   └─ account.sync_later (触发同步)
   ↓
4. SyncJob 异步执行
   ↓
5. Account::Syncer#perform_sync
   ├─ import_market_data (导入市场数据)
   └─ materialize_balances (物化余额)
      ↓
6. Balance::Materializer#materialize_balances
   ├─ materialize_holdings (物化持仓)
   │  ├─ ForwardCalculator#calculate (逐日计算持仓)
   │  │  ├─ PortfolioCache (预加载数据)
   │  │  ├─ transform_portfolio (应用当日交易，qty 增加)
   │  │  ├─ build_holdings (生成当日持仓)
   │  │  └─ Holding.gapfill (缺口填充)
   │  └─ persist_holdings (批量写入)
   ├─ calculate_balances (计算余额)
   │  └─ flows_for_date (trade 类型：现金流出，非现金流入)
   ├─ persist_balances (写入余额)
   └─ update_account_info (仅 forward 策略，更新 Account.balance)
```

### 7.2 利息收入示例

```
1. 用户提交利息表单
   ↓
2. Trade::CreateForm#create_interest_income
   ├─ Entry.create (创建账务条目，amount 为负)
   ├─ Transaction.create (entryable 类型)
   └─ account.sync_later
   ↓
3. SyncJob 异步执行
   ↓
4. Balance 计算
   ├─ materialize_holdings (持仓无变化，因为没有 Trade)
   └─ calculate_balances
      └─ flows_for_date (transaction 类型：现金流入，非现金无变化)
```

---

## 八、关键设计模式

1. **CQRS 风格**: 写入（交易录入）与读取（持仓查询）分离，通过异步同步连接

2. **事件溯源思想**: 交易是不可变事件，持仓/余额是事件投影

3. **策略模式**: Forward/Reverse 两种计算策略，适配不同账户类型

4. **物化视图**: Holding 和 Balance 是 Trade 逻辑物化到数据库

5. **缓存前置**: PortfolioCache 和 Balance::SyncCache 预加载所有依赖数据

6. **幂等设计**: 重复同步不会重复创建，使用 upsert

7. **多态设计**: Entry 通过 delegated_type 支持多种 entryable 类型

---

## 九、代码溯源

| 模块 | 文件位置 |
|------|----------|
| 交易表单 | `app/models/trade/create_form.rb` |
| 交易模型 | `app/models/trade.rb` |
| 普通交易 | `app/models/transaction.rb` |
| 转账创建器 | `app/models/transfer/creator.rb` |
| 转账模型 | `app/models/transfer.rb` |
| 账务条目 | `app/models/entry.rb` |
| 持仓模型 | `app/models/holding.rb` |
| 正向计算（持仓） | `app/models/holding/forward_calculator.rb` |
| 反向计算（持仓） | `app/models/holding/reverse_calculator.rb` |
| 投资组合缓存 | `app/models/holding/portfolio_cache.rb` |
| 持仓物化器 | `app/models/holding/materializer.rb` |
| 缺口填充 | `app/models/holding/gapfillable.rb` |
| 持仓快照 | `app/models/holding/portfolio_snapshot.rb` |
| 余额基础计算器 | `app/models/balance/base_calculator.rb` |
| 正向计算（余额） | `app/models/balance/forward_calculator.rb` |
| 余额物化器 | `app/models/balance/materializer.rb` |
| 余额同步缓存 | `app/models/balance/sync_cache.rb` |
| 账户同步器 | `app/models/account/syncer.rb` |
| 同步机制 | `app/models/concerns/syncable.rb` |
| 交易控制器 | `app/controllers/trades_controller.rb` |
