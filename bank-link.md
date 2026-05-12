# 外部银行接入（Plaid）到本地账户挂载链路分析

## 概述

本文档详细分析了 Maybe Finance 应用中外部银行通过 Plaid 接入到本地账户挂载的完整链路，包括：
- 初次授权与 Token 交换流程
- 账户清单拉取
- 数据落库与同步机制
- 错误处理与回滚机制

---

## 一、整体架构概览

```
┌────────────────────┐     ┌────────────────────┐     ┌────────────────────┐
│     前端 (Plaid   │────▶│   后端 (Rails)    │────▶│    Plaid API      │
│    Link 连接器)   │     │                    │     │                    │
└────────────────────┘     └────────────────────┘     └────────────────────┘
         ▲                          ▲                          ▲
         │                          │                          │
         │  Link Token             │  Access Token          │  Webhook
         │  Public Token            │  Item/Account 数据      │  通知
         │                          │                          │
         ▼                          ▼                          │
┌──────────────────────────────────────────────────────────────────┘
                    本地数据库 (PostgreSQL)
```

---

## 二、数据模型关系图

```
Family
  │
  ├── PlaidItem (1:N)
  │       │
  │       ├── access_token (加密存储)
  │       ├── plaid_id (Plaid 侧唯一标识)
  │       ├── next_cursor (交易同步游标)
  │       │
  │       └── PlaidAccount (1:N)
  │               │
  │               ├── plaid_id (账户级唯一标识)
  │               ├── plaid_type / plaid_subtype
  │               ├── current_balance / available_balance
  │               └── raw_payload (原始数据快照)
  │               ├── raw_transactions_payload
  │               ├── raw_investments_payload
  │               └── raw_liabilities_payload
  │
  └── Account (1:N)
          │
          ├── plaid_account_id (关联 PlaidAccount)
          ├── accountable (多态关联: Depository, CreditCard, etc.)
          ├── balance / currency
          └── status (active/draft/disabled/pending_deletion)
```

**关键文件位置：
- 模型定义: `app/models/plaid_item.rb` (L1-L120`
- 账户模型: `app/models/plaid_account.rb` (L1-L54`)
- 主账户模型: `app/models/account.rb` (L1-L163`)

---

## 三、初次授权与 Token 交换流程

### 3.1 流程总览

```
步骤 1: 用户点击"连接银行"
     │
     ▼
步骤 2: 前端请求 Link Token
     │
     ▼
步骤 3: 后端调用 Plaid API 生成 Link Token
     │
     ▼
步骤 4: 前端初始化 Plaid Link 界面
     │
     ▼
步骤 5: 用户完成银行授权
     │
     ▼
步骤 6: Plaid 返回 Public Token
     │
     ▼
步骤 7: 前端提交 Public Token 到后端
     │
     ▼
步骤 8: 后端交换 Public Token → Access Token
     │
     ▼
步骤 9: 创建 PlaidItem 记录并触发同步
```

### 3.2 详细步骤

#### 第一步：获取 Link Token

**入口：`app/controllers/plaid_items_controller.rb` (L4-L14`)

```ruby
def new
  region = params[:region] == "eu" ? :eu : :us
  webhooks_url = region == :eu ? plaid_eu_webhooks_url : plaid_us_webhooks_url

  @link_token = Current.family.get_link_token(
    webhooks_url: webhooks_url,
    redirect_url: accounts_url,
    accountable_type: params[:accountable_type] || "Depository",
    region: region
  )
end
```

**核心实现：`app/models/family/plaid_connectable.rb` (L32-L42`)

```ruby
def get_link_token(webhooks_url:, redirect_url:, accountable_type: nil, region: :us, access_token: nil)
  return nil unless plaid(region)

  plaid(region).get_link_token(
    user_id: self.id,
    webhooks_url: webhooks_url,
    redirect_url: redirect_url,
    accountable_type: accountable_type,
    access_token: access_token
  ).link_token
end
```

**Plaid 提供者实现：`app/models/provider/plaid.rb` (L45-L66`)

```ruby
def get_link_token(user_id:, webhooks_url:, redirect_url:, accountable_type: nil, access_token: nil)
  request_params = {
    user: { client_user_id: user_id },
    client_name: "Maybe Finance",
    country_codes: country_codes,
    language: "en",
    webhook: webhooks_url,
    redirect_uri: redirect_url,
    transactions: { days_requested: MAX_HISTORY_DAYS }
  }

  if access_token.present?
    request_params[:access_token] = access_token
  else
    request_params[:products] = [ get_primary_product(accountable_type) ]
    request_params[:additional_consented_products] = get_additional_consented_products(accountable_type)
  end

  request = Plaid::LinkTokenCreateRequest.new(request_params)
  client.link_token_create(request)
end
```

**产品选择逻辑** (`app/models/provider/plaid.rb` L182-L199`)

| 账户类型 | 主产品 | 附加同意产品 |
|---------|--------|-------------|
| Depository (默认) | transactions | investments, liabilities |
| Investment | investments | transactions, liabilities |
| CreditCard/Loan | liabilities | transactions, investments |

#### 第二步：前端 Plaid Link 初始化

**视图模板：`app/views/plaid_items/new.html.erb` (L1-L8`)

```erb
<%= turbo_frame_tag "modal" do %>
  <%= render "plaid_items/auto_link_opener",
           link_token: @link_token,
           region: params[:region],
           item_id: "",
           is_update: false %>
<% end %>
```

**Stimulus 控制器：`app/javascript/controllers/plaid_controller.js` (L1-L80`)

```javascript
connect() {
  this.open();
}

open() {
  const handler = Plaid.create({
    token: this.linkTokenValue,
    onSuccess: this.handleSuccess,
    onLoad: this.handleLoad,
    onExit: this.handleExit,
    onEvent: this.handleEvent,
  });
  handler.open();
}
```

#### 第三步：Token 交换

**成功回调处理** (`app/javascript/controllers/plaid_controller.js` L28-L64`)

```javascript
handleSuccess = (public_token, metadata) => {
  // 新连接流程
  fetch("/plaid_items", {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "X-CSRF-Token": document.querySelector('[name="csrf-token"]').content,
    },
    body: JSON.stringify({
      plaid_item: {
        public_token: public_token,
        metadata: metadata,
        region: this.regionValue,
      },
    }),
  }).then((response) => {
    if (response.redirected) {
      window.location.href = response.url;
    }
  });
};
```

#### 第四步：后端 Token 交换与 PlaidItem 创建

**控制器** (`app/controllers/plaid_items_controller.rb` L25-L33`)

```ruby
def create
  Current.family.create_plaid_item!(
    public_token: plaid_item_params[:public_token],
    item_name: item_name,
    region: plaid_item_params[:region]
  )
  redirect_to accounts_path, notice: t(".success")
end
```

**核心创建逻辑** (`app/models/family/plaid_connectable.rb` L17-L30`)

```ruby
def create_plaid_item!(public_token:, item_name:, region:)
  # 1. 交换 Public Token → Access Token
  public_token_response = plaid(region).exchange_public_token(public_token)

  # 2. 创建 PlaidItem 记录
  plaid_item = plaid_items.create!(
    name: item_name,
    plaid_id: public_token_response.item_id,
    access_token: public_token_response.access_token,
    plaid_region: region
  )

  # 3. 立即触发第一次同步
  plaid_item.sync_later

  plaid_item
end
```

**Token 交换 API** (`app/models/provider/plaid.rb` L68-L74`)

```ruby
def exchange_public_token(token)
  request = Plaid::ItemPublicTokenExchangeRequest.new(
    public_token: token
  )
  client.item_public_token_exchange(request)
end
```

**Plaid 配置** (`config/initializers/plaid.rb` L1-L18`)

- 支持 US 和 EU 两个区域
- Access Token 采用 Active Record 加密存储
- 环境变量配置：`PLAID_CLIENT_ID`, `PLAID_SECRET`, `PLAID_ENV`

---

## 四、账户清单拉取与落库

### 4.1 同步触发机制

**同步入口** (`app/models/concerns/syncable.rb` L14-L35`)

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

**同步执行** (`app/models/sync.rb` L60-L80`)

```ruby
def perform
  Rails.logger.tagged("Sync", id, syncable_type, syncable_id) do
    unless may_start?
      Rails.logger.warn("Sync #{id} is not in a valid state (#{aasm.from_state}) to start.  Skipping sync.")
      return
    end

    start!

    begin
      syncable.perform_sync(self)  # 调用具体 Syncer
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

### 4.2 PlaidItem 同步流程

**PlaidItem Syncer** (`app/models/plaid_item/syncer.rb` L1-L26`)

```ruby
class PlaidItem::Syncer
  attr_reader :plaid_item

  def perform_sync(sync)
    # 阶段 1: 从 Plaid 拉取原始数据并存储
    plaid_item.import_latest_plaid_data

    # 阶段 2: 处理原始数据，更新内部领域对象
    plaid_item.process_accounts

    # 阶段 3: 调度账户级同步（计算历史余额等）
    plaid_item.schedule_account_syncs(
      parent_sync: sync,
      window_start_date: sync.window_start_date,
      window_end_date: sync.window_end_date
    )
  end
end
```

### 4.3 阶段一：数据导入（Import）

**导入器入口** (`app/models/plaid_item/importer.rb` L1-L57`)

```ruby
class PlaidItem::Importer
  def initialize(plaid_item, plaid_provider:)
    @plaid_item = plaid_item
    @plaid_provider = plaid_provider
  end

  def import
    fetch_and_import_item_data      # 拉取 Item 和 Institution 数据
    fetch_and_import_accounts_data  # 拉取账户和交易数据
  rescue Plaid::ApiError => e
    handle_plaid_error(e)
  end

  private

  def fetch_and_import_item_data
    item_data = plaid_provider.get_item(plaid_item.access_token).item
    institution_data = plaid_provider.get_institution(item_data.institution_id).institution

    plaid_item.upsert_plaid_snapshot!(item_data)
    plaid_item.upsert_plaid_institution_snapshot!(institution_data)
  end

  def fetch_and_import_accounts_data
    snapshot = PlaidItem::AccountsSnapshot.new(plaid_item, plaid_provider: plaid_provider)

    PlaidItem.transaction do
      snapshot.accounts.each do |raw_account|
        plaid_account = plaid_item.plaid_accounts.find_or_initialize_by(
          plaid_id: raw_account.account_id
        )

        PlaidAccount::Importer.new(
          plaid_account,
          account_snapshot: snapshot.get_account_data(raw_account.account_id)
        ).import
      end

      plaid_item.update!(next_cursor: snapshot.transactions_cursor)
    end
  end
end
```

**PlaidItem 快照更新** (`app/models/plaid_item.rb` L73-L94`)

```ruby
def upsert_plaid_snapshot!(item_snapshot)
  assign_attributes(
    available_products: item_snapshot.available_products,
    billed_products: item_snapshot.billed_products,
    raw_payload: item_snapshot,
  )
  save!
end

def upsert_plaid_institution_snapshot!(institution_snapshot)
  assign_attributes(
    institution_id: institution_snapshot.institution_id,
    institution_url: institution_snapshot.url,
    institution_color: institution_snapshot.primary_color,
    raw_institution_payload: institution_snapshot
  )
  save!
end
```

### 4.4 账户数据快照（AccountsSnapshot）

**快照类** (`app/models/plaid_item/accounts_snapshot.rb` L1-L105`)

```ruby
class PlaidItem::AccountsSnapshot
  def initialize(plaid_item, plaid_provider:)
    @plaid_item = plaid_item
    @plaid_provider = plaid_provider
  end

  def accounts
    @accounts ||= plaid_provider.get_item_accounts(plaid_item.access_token).accounts
  end

  def get_account_data(account_id)
    AccountData.new(
      account_data: accounts.find { |a| a.account_id == account_id },
      transactions_data: account_scoped_transactions_data(account_id),
      investments_data: account_scoped_investments_data(account_id),
      liabilities_data: account_scoped_liabilities_data(account_id)
    )
  end

  def transactions_cursor
    return nil unless transactions_data
    transactions_data.cursor
  end

  private

  TransactionsData = Data.define(:added, :modified, :removed)
  LiabilitiesData = Data.define(:credit, :mortgage, :student)
  InvestmentsData = Data.define(:transactions, :holdings, :securities)
  AccountData = Data.define(:account_data, :transactions_data, :investments_data, :liabilities_data)

  def can_fetch_transactions?
    plaid_item.supports_product?("transactions") && accounts.any?
  end

  def transactions_data
    return nil unless can_fetch_transactions?
    @transactions_data ||= plaid_provider.get_transactions(
      plaid_item.access_token,
      next_cursor: plaid_item.next_cursor
    )
  end

  def can_fetch_investments?
    plaid_item.supports_product?("investments") &&
    accounts.any? { |a| a.type == "investment" }
  end

  def investments_data
    return nil unless can_fetch_investments?
    @investments_data ||= plaid_provider.get_item_investments(plaid_item.access_token)
  end

  def can_fetch_liabilities?
    plaid_item.supports_product?("liabilities") &&
    accounts.any? do |a|
      a.type == "credit" && a.subtype == "credit card" ||
      a.type == "loan" && (a.subtype == "mortgage" || a.subtype == "student")
    end
  end

  def liabilities_data
    return nil unless can_fetch_liabilities?
    @liabilities_data ||= plaid_provider.get_item_liabilities(plaid_item.access_token)
  end
end
```

### 4.5 PlaidAccount 导入器

**PlaidAccount::Importer** (`app/models/plaid_account/importer.rb` L1-L32`)

```ruby
class PlaidAccount::Importer
  def initialize(plaid_account, account_snapshot:)
    @plaid_account = plaid_account
    @account_snapshot = account_snapshot
  end

  def import
    import_account_info
    import_transactions if account_snapshot.transactions_data.present?
    import_investments if account_snapshot.investments_data.present?
    import_liabilities if account_snapshot.liabilities_data.present?
  end

  private

  def import_account_info
    plaid_account.upsert_plaid_snapshot!(account_snapshot.account_data)
  end

  def import_transactions
    plaid_account.upsert_plaid_transactions_snapshot!(account_snapshot.transactions_data)
  end

  def import_investments
    plaid_account.upsert_plaid_investments_snapshot!(account_snapshot.investments_data)
  end

  def import_liabilities
    plaid_account.upsert_plaid_liabilities_snapshot!(account_snapshot.liabilities_data)
  end
end
```

**PlaidAccount 快照更新** (`app/models/plaid_account.rb` L9-L46`)

```ruby
def upsert_plaid_snapshot!(account_snapshot)
  assign_attributes(
    current_balance: account_snapshot.balances.current,
    available_balance: account_snapshot.balances.available,
    currency: account_snapshot.balances.iso_currency_code,
    plaid_type: account_snapshot.type,
    plaid_subtype: account_snapshot.subtype,
    name: account_snapshot.name,
    mask: account_snapshot.mask,
    raw_payload: account_snapshot
  )
  save!
end

def upsert_plaid_transactions_snapshot!(transactions_snapshot)
  assign_attributes(
    raw_transactions_payload: transactions_snapshot
  )
  save!
end
```

### 4.6 阶段二：数据处理（Process）

**PlaidItem 处理入口** (`app/models/plaid_item.rb` L56-L60`)

```ruby
def process_accounts
  plaid_accounts.each do |plaid_account|
    PlaidAccount::Processor.new(plaid_account).process
  end
end
```

**PlaidAccount::Processor** (`app/models/plaid_account/processor.rb` L1-L109`)

```ruby
class PlaidAccount::Processor
  include PlaidAccount::TypeMappable

  def process
    process_account!           # 必须成功，失败则整个处理中断
    process_transactions    # 可选，失败不影响其他步骤
    process_investments    # 可选
    process_liabilities      # 可选
  end

  private

  def process_account!
    PlaidAccount.transaction do
      # 查找或创建本地 Account
      account = family.accounts.find_or_initialize_by(
        plaid_account_id: plaid_account.id
      )

      # 允许用户覆盖的属性：name 和 subtype
      account.enrich_attributes(
        {
          name: plaid_account.name,
          subtype: map_subtype(plaid_account.plaid_type, plaid_account.plaid_subtype)
        },
        source: "plaid"
      )

      # 映射 Plaid 类型到本地 Accountable
      account.assign_attributes(
        accountable: map_accountable(plaid_account.plaid_type),
        balance: balance_calculator.balance,
        currency: plaid_account.currency,
        cash_balance: balance_calculator.cash_balance
      )

      account.save!

      # 设置当前余额锚点（事件溯源账本）
      account.set_current_balance(balance_calculator.balance)
    end
  end

  def process_transactions
    PlaidAccount::Transactions::Processor.new(plaid_account).process
  rescue => e
    report_exception(e)
  end

  def process_investments
    PlaidAccount::Investments::TransactionsProcessor.new(plaid_account, security_resolver: security_resolver).process
    PlaidAccount::Investments::HoldingsProcessor.new(plaid_account, security_resolver: security_resolver).process
  rescue => e
    report_exception(e)
  end

  def process_liabilities
    case [ plaid_account.plaid_type, plaid_account.plaid_subtype ]
    when [ "credit", "credit card" ]
      PlaidAccount::Liabilities::CreditProcessor.new(plaid_account).process
    when [ "loan", "mortgage" ]
      PlaidAccount::Liabilities::MortgageProcessor.new(plaid_account).process
    when [ "loan", "student" ]
      PlaidAccount::Liabilities::StudentLoanProcessor.new(plaid_account).process
    end
  rescue => e
    report_exception(e)
  end
end
```

**类型映射** (`app/models/plaid_account/type_mappable.rb` L1-L77`)

| Plaid Type | 本地 Accountable | 示例 Subtypes |
|-----------|----------------|--------------|
| depository | Depository | checking, savings, hsa, cd, money_market |
| credit | CreditCard | credit_card |
| loan | Loan | mortgage, student, auto, business, home_equity, line_of_credit |
| investment | Investment | brokerage, pension, retirement, 401k, roth_401k, 529_plan, hsa, mutual_fund, roth_ira, ira |
| other | OtherAsset | other |

### 4.7 阶段三：账户级同步调度

**调度账户同步** (`app/models/plaid_item.rb` L63-L71`)

```ruby
def schedule_account_syncs(parent_sync: nil, window_start_date: nil, window_end_date: nil)
  accounts.each do |account|
    account.sync_later(
      parent_sync: parent_sync,
      window_start_date: window_start_date,
      window_end_date: window_end_date
    )
  end
end
```

---

## 五、数据库表结构

### 5.1 plaid_items 表

```ruby
create_table "plaid_items", id: :uuid do |t|
  t.uuid "family_id", null: false
  t.string "access_token"           # 加密存储
  t.string "plaid_id", null: false  # Plaid 侧唯一标识
  t.string "name"
  t.string "next_cursor"              # 交易同步游标
  t.boolean "scheduled_for_deletion", default: false
  t.string "available_products", default: [], array: true
  t.string "billed_products", default: [], array: true
  t.string "plaid_region", default: "us", null: false
  t.string "institution_url"
  t.string "institution_id"
  t.string "institution_color"
  t.string "status", default: "good", null: false
  t.jsonb "raw_payload", default: {}
  t.jsonb "raw_institution_payload", default: {}
end
```

### 5.2 plaid_accounts 表

```ruby
create_table "plaid_accounts", id: :uuid do |t|
  t.uuid "plaid_item_id", null: false
  t.string "plaid_id", null: false        # 账户级唯一标识
  t.string "plaid_type", null: false
  t.string "plaid_subtype"
  t.decimal "current_balance", precision: 19, scale: 4
  t.decimal "available_balance", precision: 19, scale: 4
  t.string "currency", null: false
  t.string "name", null: false
  t.string "mask"
  t.jsonb "raw_payload", default: {}
  t.jsonb "raw_transactions_payload", default: {}
  t.jsonb "raw_investments_payload", default: {}
  t.jsonb "raw_liabilities_payload", default: {}
end
```

### 5.3 accounts 表（关联字段

```ruby
create_table "accounts", id: :uuid do |t|
  t.uuid "family_id", null: false
  t.string "name"
  t.string "accountable_type"
  t.uuid "accountable_id"
  t.decimal "balance", precision: 19, scale: 4
  t.string "currency"
  t.virtual "classification", type: :string, as: "...", stored: true
  t.uuid "plaid_account_id"    # 关联 PlaidAccount
  t.decimal "cash_balance", precision: 19, scale: 4, default: "0.0"
  t.jsonb "locked_attributes", default: {}
  t.string "status", default: "active"
end
```

---

## 六、错误处理与回滚机制

### 6.1 Token 级错误处理

**Plaid API 错误分类** (`app/models/plaid_item/importer.rb` L19-L28`)

```ruby
def handle_plaid_error(error)
  error_body = JSON.parse(error.response_body)

  case error_body["error_code"]
  when "ITEM_LOGIN_REQUIRED"
    plaid_item.update!(status: :requires_update)
  else
    raise error
  end
end
```

**错误码处理总结：

| 错误码 | 处理方式 | 说明 |
|--------|---------|------|
| ITEM_LOGIN_REQUIRED | 标记为 requires_update | 需要用户重新授权 |
| ITEM_NOT_FOUND | 标记为 requires_update | Plaid 端已删除 |
| 其他错误 | 重新抛出 | 同步标记为失败 |

### 6.2 同步状态机

**Sync 状态机** (`app/models/sync.rb` L26-L52`)

```
pending ──start──▶ syncing ──complete──▶ completed
                      │
                      └──fail──▶ failed
                      │
                      └──mark_stale──▶ stale
```

**状态说明：
- **pending**: 等待执行
- **syncing**: 正在执行中
- **completed**: 成功完成
- **failed**: 执行失败
- **stale**: 超时未完成（24小时）

**状态转换处理** (`app/models/sync.rb` L82-L104`)

```ruby
def finalize_if_all_children_finalized
  Sync.transaction do
    lock!

    return unless all_children_finalized?

    if syncing?
      if has_failed_children?
        fail!
      else
        complete!
      end
    end

    perform_post_sync
  end

  parent&.finalize_if_all_children_finalized
end
```

### 6.3 事务与原子性

**账户数据导入事务** (`app/models/plaid_item/importer.rb` L38-L56`)

```ruby
def fetch_and_import_accounts_data
  snapshot = PlaidItem::AccountsSnapshot.new(plaid_item, plaid_provider: plaid_provider)

  PlaidItem.transaction do
    snapshot.accounts.each do |raw_account|
      plaid_account = plaid_item.plaid_accounts.find_or_initialize_by(
        plaid_id: raw_account.account_id
      )

      PlaidAccount::Importer.new(
        plaid_account,
        account_snapshot: snapshot.get_account_data(raw_account.account_id)
      ).import
    end

    # 所有数据导入成功后才更新 cursor
    plaid_item.update!(next_cursor: snapshot.transactions_cursor)
  end
end
```

**关键设计要点：
1. 所有账户导入在一个事务内完成
2. cursor 只在所有账户都成功导入后才更新
3. 任何一个账户导入失败，整个事务回滚，cursor 不更新

**账户处理事务** (`app/models/plaid_account/processor.rb` L31-L62`)

```ruby
def process_account!
  PlaidAccount.transaction do
    account = family.accounts.find_or_initialize_by(
      plaid_account_id: plaid_account.id
    )

    account.enrich_attributes(
      {
        name: plaid_account.name,
        subtype: map_subtype(plaid_account.plaid_type, plaid_account.plaid_subtype)
      },
      source: "plaid"
    )

    account.assign_attributes(
      accountable: map_accountable(plaid_account.plaid_type),
      balance: balance_calculator.balance,
      currency: plaid_account.currency,
      cash_balance: balance_calculator.cash_balance
    )

    account.save!

    account.set_current_balance(balance_calculator.balance)
  end
end
```

### 6.4 级联删除与软删除机制

**PlaidItem 销毁流程**

1. **软删除标记** (`app/models/plaid_item.rb` L44-L47`)
```ruby
def destroy_later
  update!(scheduled_for_deletion: true)
  DestroyJob.perform_later(self)
end
```

2. **实际销毁** (`app/models/plaid_item.rb` L101-L112`)
```ruby
def remove_plaid_item
  plaid_provider.remove_item(access_token)
rescue Plaid::ApiError => e
  json_response = JSON.parse(e.response_body)

  unless json_response["error_code"] == "ITEM_NOT_FOUND"
    raise e
  end
end
```

3. **级联关系** (`app/models/plaid_item.rb` L18-L19`)
```ruby
has_many :plaid_accounts, dependent: :destroy
has_many :accounts, through: :plaid_accounts
```

4. **账户级联** (`app/models/plaid_account.rb` L4`)
```ruby
has_one :account, dependent: :destroy
```

5. **DestroyJob** (`app/jobs/destroy_job.rb` L1-L9`)
```ruby
class DestroyJob < ApplicationJob
  queue_as :low_priority

  def perform(model)
    model.destroy
  rescue => e
    model.update!(scheduled_for_deletion: false) # 重置状态，允许用户重试
  end
end
```

**删除链：
```
PlaidItem.destroy_later
  │
  ├── update!(scheduled_for_deletion: true)
  │
  └── DestroyJob.perform_later
           │
           └── PlaidItem.destroy
                    │
                    ├── before_destroy: remove_plaid_item (调用 Plaid API)
                    │
                    └── has_many :plaid_accounts, dependent: :destroy
                             │
                             └── has_one :account, dependent: :destroy
```

### 6.5 分步错误隔离

**PlaidAccount::Processor 错误隔离** (`app/models/plaid_account/processor.rb` L64-L88`)

```ruby
def process
  process_account!           # 必须成功，失败则整个处理中断
  process_transactions    # 失败不影响其他步骤，仅记录到 Sentry
  process_investments    # 同上
  process_liabilities      # 同上
end
```

**设计意图：
- `process_account!` 用 `!` 表示必须成功
- 其他步骤用 `rescue` 捕获异常并上报 Sentry
- 单个账户的交易/投资/负债处理失败不会影响其他账户或主账户创建

### 6.6 Webhook 错误处理

**Webhook 验证** (`app/models/provider/plaid.rb` L14-L43`)

```ruby
def validate_webhook!(verification_header, raw_body)
  jwks_loader = ->(options) do
    key_id = options[:kid]
    jwk_response = client.webhook_verification_key_get(
      Plaid::WebhookVerificationKeyGetRequest.new(key_id: key_id)
    )
    jwks = JWT::JWK::Set.new([ jwk_response.key.to_hash ])
    jwks.filter! { |key| key[:use] == "sig" }
    jwks
  end

  payload, _header = JWT.decode(
    verification_header, nil, true,
    {
      algorithms: [ "ES256" ],
      jwks: jwks_loader,
      verify_expiration: false
    }
  )

  issued_at = Time.at(payload["iat"])
  raise JWT::VerificationError, "Webhook is too old" if Time.now - issued_at > 5.minutes

  expected_hash = payload["request_body_sha256"]
  actual_hash = Digest::SHA256.hexdigest(raw_body)
  raise JWT::VerificationError, "Invalid webhook body hash" unless ActiveSupport::SecurityUtils.secure_compare(expected_hash, actual_hash)
end
```

**Webhook 处理** (`app/models/plaid_item/webhook_processor.rb` L1-L56`)

```ruby
class PlaidItem::WebhookProcessor
  def process
    unless plaid_item
      handle_missing_item
      return
    end

    case [ webhook_type, webhook_code ]
    when [ "TRANSACTIONS", "SYNC_UPDATES_AVAILABLE" ]
      plaid_item.sync_later
    when [ "INVESTMENTS_TRANSACTIONS", "DEFAULT_UPDATE" ]
      plaid_item.sync_later
    when [ "HOLDINGS", "DEFAULT_UPDATE" ]
      plaid_item.sync_later
    when [ "ITEM", "ERROR" ]
      if error["error_code"] == "ITEM_LOGIN_REQUIRED"
        plaid_item.update!(status: :requires_update)
      end
    else
      Rails.logger.warn("Unhandled Plaid webhook type: #{webhook_type}:#{webhook_code}")
    end
  rescue => e
    # 静默捕获并上报所有错误，确保返回 200 给 Plaid
    Sentry.capture_exception(e)
  end
end
```

---

## 七、完整时序图

```
用户                    前端                     后端                    Plaid API                数据库
 │                       │                        │                        │
 │──点击连接银行──▶│                        │                        │
 │                       │──GET /plaid_items/new │                        │
 │                       │                        │──link_token_create──▶│
 │                       │                        │◀──link_token───│
 │                       │◀──link_token───│                        │
 │                       │──Plaid Link 初始化──▶│                        │
 │◀──银行授权界面──│                        │                        │
 │──完成授权───▶│                        │                        │
 │                       │                        │                        │
 │                       │──POST /plaid_items │                        │
 │                       │  (public_token) │                        │
 │                       │                        │──item_public_token_exchange──▶│
 │                       │                        │◀──access_token───│
 │                       │                        │                        │
 │                       │                        │──INSERT plaid_items──▶│
 │                       │                        │                        │
 │                       │                        │──SyncJob.perform_later──▶│
 │                       │                        │                        │
 │◀──redirect /accounts──│                        │                        │
 │                       │                        │                        │
 │                       │                        │                        │
 │                       │  ──SyncJob 执行──▶│                        │
 │                       │                        │──get_item──▶│
 │                       │                        │◀──item_data───│
 │                       │                        │──get_institution──▶│
 │                       │                        │◀──institution_data───│
 │                       │                        │──UPDATE plaid_items──▶│
 │                       │                        │                        │
 │                       │                        │──get_item_accounts──▶│
 │                       │                        │◀──accounts───│
 │                       │                        │                        │
 │                       │                        │──get_transactions──▶│
 │                       │                        │◀──transactions───│
 │                       │                        │                        │
 │                       │                        │──INSERT/UPDATE plaid_accounts──▶│
 │                       │                        │──UPDATE next_cursor──▶│
 │                       │                        │                        │
 │                       │                        │                        │
 │                       │                        │──process_accounts──▶│
 │                       │                        │                        │
 │                       │                        │──find_or_create accounts──▶│
 │                       │                        │──INSERT/UPDATE accounts──▶│
 │                       │                        │                        │
 │                       │                        │──schedule_account_syncs──▶│
 │                       │                        │                        │
```

---

## 八、关键设计亮点

### 8.1 三层数据存储策略

1. **原始数据保留**
   - `raw_payload` 系列字段完整保存 Plaid 原始响应
   - 便于调试和数据重放
   - 支持 "re-sync" 已获取的数据

2. **游标同步**
   - `next_cursor` 实现增量同步
   - 避免重复拉取历史数据

3. **类型映射层**
   - Plaid 类型 → 本地 Accountable 类型
   - 支持多种账户类型抽象

### 8.2 同步机制

1. **父子同步树**
   - PlaidItem 同步作为父同步
   - 每个 Account 同步作为子同步
   - 父同步等待所有子同步完成后才完成

2. **幂等性设计**
   - `find_or_initialize_by(plaid_id: ...)`
   - 重复调用不会创建重复记录

3. **事务边界清晰

### 8.3 安全措施

1. **Token 加密**
   - `encrypts :access_token, deterministic: true

2. **Webhook 验证**
   - JWT 签名验证
   - 时间戳检查（5 分钟内）
   - 请求体哈希验证

3. **错误上报**
   - Sentry 集成
   - 详细的上下文标签

---

## 九、相关文件索引

| 文件路径 | 说明 |
|---------|------|
| `app/controllers/plaid_items_controller.rb` | Plaid 项目控制器 |
| `app/models/family/plaid_connectable.rb` | Family 与 Plaid 连接模块 |
| `app/models/provider/plaid.rb` | Plaid API 提供者 |
| `app/models/plaid_item.rb` | PlaidItem 主模型 |
| `app/models/plaid_item/importer.rb` | PlaidItem 数据导入器 |
| `app/models/plaid_item/syncer.rb` | PlaidItem 同步器 |
| `app/models/plaid_item/accounts_snapshot.rb` | 账户数据快照 |
| `app/models/plaid_item/webhook_processor.rb` | Webhook 处理器 |
| `app/models/plaid_account.rb` | PlaidAccount 模型 |
| `app/models/plaid_account/importer.rb` | PlaidAccount 数据导入器 |
| `app/models/plaid_account/processor.rb` | PlaidAccount 处理器 |
| `app/models/plaid_account/type_mappable.rb` | 类型映射模块 |
| `app/models/concerns/syncable.rb` | 同步通用模块 |
| `app/models/sync.rb` | Sync 记录模型 |
| `app/jobs/sync_job.rb` | 同步作业 |
| `app/jobs/destroy_job.rb` | 销毁作业 |
| `app/javascript/controllers/plaid_controller.js` | 前端 Plaid 控制器 |
| `config/initializers/plaid.rb` | Plaid 配置初始化 |
| `config/routes.rb` | 路由配置 |
| `db/schema.rb` | 数据库结构 |

---

## 十、附录：主要流程总结

### 10.1 新建银行连接

1. 用户点击"连接银行"
2. 后端生成 Link Token
3. 前端打开 Plaid Link 界面
4. 用户完成银行授权
5. Plaid 返回 Public Token
6. 后端交换为 Access Token
7. 创建 PlaidItem 记录
8. 触发第一次同步

### 10.2 同步流程

1. 拉取 Item 和 Institution 数据
2. 拉取账户列表
3. 根据产品支持情况拉取交易/投资/负债数据
4. 事务内更新所有 PlaidAccount
5. 更新 cursor
6. 处理每个账户 → 创建/更新本地 Account
7. 处理交易、投资、负债
8. 调度账户级同步

### 10.3 错误场景

| 场景 | 处理方式 |
|-----|---------|
| ITEM_LOGIN_REQUIRED | 标记 requires_update，等待用户重新授权 |
| 同步失败 | Sync 标记为 failed，记录错误信息 |
| 部分步骤失败 | 上报 Sentry，继续其他步骤 |
| Webhook 验证失败 | 返回 400，上报 Sentry |
| 删除时 Plaid 已删除 | 忽略错误，继续本地删除 |
