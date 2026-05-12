# 外部银行接入（Plaid）到本地账户挂载链路分析

## 概述

本文档详细分析了 Maybe Finance 应用中外部银行通过 Plaid 接入到本地账户挂载的完整链路，特别关注：
- 初次授权与 Token 交换流程
- 账户清单拉取
- 数据落库与同步机制
- **ITEM_NOT_FOUND 在各阶段的处理链路**
- **事务边界与回滚机制**

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
  │       ├── status (good / requires_update)
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

**关键文件位置：**
- PlaidItem 模型: `app/models/plaid_item.rb` (L1-L120)
- PlaidAccount 模型: `app/models/plaid_account.rb` (L1-L54)
- Account 模型: `app/models/account.rb` (L1-L163)

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

**控制器入口** `app/controllers/plaid_items_controller.rb` (L4-L14)

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

**核心实现** `app/models/family/plaid_connectable.rb` (L32-L42)

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

**Plaid 提供者实现** `app/models/provider/plaid.rb` (L45-L66)

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

**产品选择逻辑** `app/models/provider/plaid.rb` (L182-L199)

| 账户类型 | 主产品 | 附加同意产品 |
|---------|--------|-------------|
| Depository (默认) | transactions | investments, liabilities |
| Investment | investments | transactions, liabilities |
| CreditCard/Loan | liabilities | transactions, investments |

#### 第二步：前端 Plaid Link 初始化

**视图模板** `app/views/plaid_items/new.html.erb` (L1-L8)

```erb
<%= turbo_frame_tag "modal" do %>
  <%= render "plaid_items/auto_link_opener",
           link_token: @link_token,
           region: params[:region],
           item_id: "",
           is_update: false %>
<% end %>
```

**Stimulus 控制器** `app/javascript/controllers/plaid_controller.js` (L1-L80)

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

**成功回调处理** `app/javascript/controllers/plaid_controller.js` (L28-L64)

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

**控制器** `app/controllers/plaid_items_controller.rb` (L25-L33)

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

**核心创建逻辑** `app/models/family/plaid_connectable.rb` (L17-L30)

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

**Token 交换 API** `app/models/provider/plaid.rb` (L68-L74)

```ruby
def exchange_public_token(token)
  request = Plaid::ItemPublicTokenExchangeRequest.new(
    public_token: token
  )
  client.item_public_token_exchange(request)
end
```

**Plaid 配置** `config/initializers/plaid.rb` (L1-L18)

- 支持 US 和 EU 两个区域
- Access Token 采用 Active Record 加密存储
- 环境变量配置：`PLAID_CLIENT_ID`, `PLAID_SECRET`, `PLAID_ENV`

---

## 四、账户清单拉取与落库

### 4.1 同步触发机制

**同步入口** `app/models/concerns/syncable.rb` (L14-L35)

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

**同步执行** `app/models/sync.rb` (L60-L80)

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

**PlaidItem Syncer** `app/models/plaid_item/syncer.rb` (L1-L26)

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

**导入器入口** `app/models/plaid_item/importer.rb` (L1-L57)

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

      # 所有数据导入成功后才更新 cursor
      plaid_item.update!(next_cursor: snapshot.transactions_cursor)
    end
  end
end
```

**PlaidItem 快照更新** `app/models/plaid_item.rb` (L73-L94)

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

**快照类** `app/models/plaid_item/accounts_snapshot.rb` (L1-L105)

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

**PlaidAccount::Importer** `app/models/plaid_account/importer.rb` (L1-L32)

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

**PlaidAccount 快照更新** `app/models/plaid_account.rb` (L9-L46)

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

**PlaidItem 处理入口** `app/models/plaid_item.rb` (L56-L60)

```ruby
def process_accounts
  plaid_accounts.each do |plaid_account|
    PlaidAccount::Processor.new(plaid_account).process
  end
end
```

**PlaidAccount::Processor** `app/models/plaid_account/processor.rb` (L1-L109)

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

**类型映射** `app/models/plaid_account/type_mappable.rb` (L1-L77)

| Plaid Type | 本地 Accountable | 示例 Subtypes |
|-----------|----------------|--------------|
| depository | Depository | checking, savings, hsa, cd, money_market |
| credit | CreditCard | credit_card |
| loan | Loan | mortgage, student, auto, business, home_equity, line_of_credit |
| investment | Investment | brokerage, pension, retirement, 401k, roth_401k, 529_plan, hsa, mutual_fund, roth_ira, ira |
| other | OtherAsset | other |

### 4.7 阶段三：账户级同步调度

**调度账户同步** `app/models/plaid_item.rb` (L63-L71)

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
  t.string "status", default: "good", null: false  # good / requires_update
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

### 5.3 accounts 表（关联字段）

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

## 六、ITEM_NOT_FOUND 处理链路详解

### 6.1 背景说明

`ITEM_NOT_FOUND` 是 Plaid API 返回的错误码，表示该 item 在 Plaid 侧已不存在。

**触发场景：**
- 用户在 Plaid 门户手动删除了该银行连接
- Plaid 管理员主动删除了该 item
- access_token 对应的 item 已过期/失效

### 6.2 阶段一：导入阶段（数据拉取）

**触发场景：** 定时同步或手动触发同步时，`SyncJob` 执行导入流程

**调用链路：**
```
SyncJob#perform
  └── Sync#perform
        └── PlaidItem#perform_sync
              └── PlaidItem::Syncer#perform_sync
                    └── PlaidItem#import_latest_plaid_data
                          └── PlaidItem::Importer#import
                                ├── fetch_and_import_item_data
                                │     └── plaid_provider.get_item(access_token)
                                │           └── Plaid::ApiError (ITEM_NOT_FOUND)
                                └── handle_plaid_error(e)
```

**代码实现** `app/models/plaid_item/importer.rb` (L17-L28)

```ruby
def handle_plaid_error(error)
  error_body = JSON.parse(error.response_body)

  case error_body["error_code"]
  when "ITEM_LOGIN_REQUIRED"
    plaid_item.update!(status: :requires_update)
  else
    # ITEM_NOT_FOUND 会走到这里，被重新抛出
    raise error
  end
end
```

**处理方式：异常上抛**

| 行为 | 说明 |
|-----|------|
| ITEM_NOT_FOUND 是否被特殊处理 | ❌ 否 |
| 处理方式 | `raise error` 重新抛出异常 |
| 上层捕获位置 | `Sync#perform` 的 `rescue => e` |
| Sync 状态变化 | `syncing` → `failed` |
| 数据库事务回滚 | ❌ 此阶段无事务（导入阶段在事务边界之外抛错） |
| PlaidItem.status 变化 | ❌ 无变化（仍为 `good`） |
| 错误信息记录 | `sync.update(error: e.message)` |

**注意：** 导入阶段的 `fetch_and_import_item_data` 和 `fetch_and_import_accounts_data` 中调用的 Plaid API（`get_item`, `get_item_accounts`, `get_transactions` 等）都在事务外执行。只有 `fetch_and_import_accounts_data` 内部有一个 `PlaidItem.transaction` 包裹数据库写入操作。

### 6.3 阶段二：更新 Token 阶段（重新授权）

**触发场景：** 用户点击"更新银行连接"，系统尝试用旧的 access_token 生成 update mode 的 link_token

**调用链路：**
```
PlaidItemsController#edit
  └── PlaidItem#get_update_link_token
        └── family.get_link_token(access_token: access_token)
              └── plaid_provider.get_link_token(..., access_token: access_token)
                    └── Plaid::ApiError (ITEM_NOT_FOUND)
                          └── rescue 捕获
```

**代码实现** `app/models/plaid_item.rb` (L25-L42)

```ruby
def get_update_link_token(webhooks_url:, redirect_url:)
  family.get_link_token(
    webhooks_url: webhooks_url,
    redirect_url: redirect_url,
    region: plaid_region,
    access_token: access_token
  )
rescue Plaid::ApiError => e
  error_body = JSON.parse(e.response_body)

  if error_body["error_code"] == "ITEM_NOT_FOUND"
    # Mark the connection as invalid but don't auto-delete
    update!(status: :requires_update)
  end

  Sentry.capture_exception(e)
  nil
end
```

**处理方式：状态标记 + 返回 nil**

| 行为 | 说明 |
|-----|------|
| ITEM_NOT_FOUND 是否被特殊处理 | ✅ 是 |
| 处理方式 | `update!(status: :requires_update)` + 上报 Sentry + `return nil` |
| 异常是否上抛 | ❌ 否（已在 rescue 中处理） |
| PlaidItem.status 变化 | `good` → `requires_update` |
| 数据库事务回滚 | ❌ 无数据库写入事务（只有 `update!` 独立执行） |
| 返回值 | `nil` |
| 前端表现 | 视图使用 `@link_token`（可能为 nil），Plaid Link 无法初始化 |

### 6.4 阶段三：删除阶段

**触发场景：** 用户点击"删除银行连接"

**调用链路：**
```
PlaidItemsController#destroy
  └── PlaidItem#destroy_later
        ├── update!(scheduled_for_deletion: true)
        └── DestroyJob.perform_later
              └── DestroyJob#perform
                    └── PlaidItem#destroy
                          ├── before_destroy :remove_plaid_item
                          │     └── plaid_provider.remove_item(access_token)
                          │           └── Plaid::ApiError (ITEM_NOT_FOUND)
                          │                 └── rescue 静默吞掉
                          └── 继续执行本地删除（级联删除）
```

**代码实现** `app/models/plaid_item.rb` (L100-L112)

```ruby
def remove_plaid_item
  plaid_provider.remove_item(access_token)
rescue Plaid::ApiError => e
  json_response = JSON.parse(e.response_body)

  # If the item is not found, that means it was already deleted by the user on their
  # Plaid portal OR by Plaid support.  Either way, we're not being billed, so continue
  # with the deletion of our internal record.
  unless json_response["error_code"] == "ITEM_NOT_FOUND"
    raise e
  end
end
```

**处理方式：静默吞掉 + 继续本地删除**

| 行为 | 说明 |
|-----|------|
| ITEM_NOT_FOUND 是否被特殊处理 | ✅ 是 |
| 处理方式 | 静默忽略，不抛异常 |
| 其他错误码处理 | `raise e` 继续抛出 |
| 级联删除是否执行 | ✅ 是 |
| 数据库事务回滚 | ❌ 无回滚（删除成功） |
| PlaidItem 及关联数据 | 被永久删除 |
| DestroyJob 异常处理 | 如果删除过程中发生其他错误：`model.update!(scheduled_for_deletion: false)` 重置状态 |

### 6.5 ITEM_NOT_FOUND 三阶段对比总结表

| 维度 | 导入阶段 (Import) | 更新 Token 阶段 (Edit) | 删除阶段 (Destroy) |
|-----|------------------|---------------------|-------------------|
| **触发时机** | 定时/手动同步 | 用户点击"更新连接" | 用户点击"删除" |
| **调用入口** | `PlaidItem::Importer#import` | `PlaidItem#get_update_link_token` | `PlaidItem#remove_plaid_item` |
| **ITEM_NOT_FOUND 处理** | ❌ 无特殊处理 | ✅ 标记 `requires_update` | ✅ 静默忽略 |
| **异常上抛** | ✅ `raise error` | ❌ 不抛，返回 nil | ❌ 不抛，继续删除 |
| **Sync 状态变化** | `syncing` → `failed` | 无 Sync 参与 | 无 Sync 参与 |
| **PlaidItem.status** | ❌ 不变（仍为 good） | `good` → `requires_update` | N/A（已删除） |
| **错误信息记录** | `sync.error = e.message` | Sentry 上报 | 不记录 |
| **数据库事务回滚** | ❌ 无（API 调用在事务外） | ❌ 无 | ❌ 无 |
| **用户可见反馈** | 同步失败，显示错误 | UI 提示需要更新 | 静默成功删除 |

---

## 七、事务边界与回滚机制详解

### 7.1 同步流程中的事务分布图

```
SyncJob#perform (无事务)
│
├── Sync#start! (单独事务)
│
├── syncable.perform_sync(self)
│     │
│     ├── 阶段一: import_latest_plaid_data
│     │     │
│     │     ├── fetch_and_import_item_data (无事务)
│     │     │     ├── get_item API 调用
│     │     │     ├── get_institution API 调用
│     │     │     └── 2 次独立的 .save! (各有自己的隐式事务)
│     │     │
│     │     └── fetch_and_import_accounts_data
│     │           │
│     │           ├── get_item_accounts API 调用 (无事务)
│     │           ├── get_transactions API 调用 (无事务)
│     │           ├── get_investments API 调用 (无事务)
│     │           ├── get_liabilities API 调用 (无事务)
│     │           │
│     │           └── PlaidItem.transaction do ──────────────┐
│     │                 ├── N 次 PlaidAccount::Importer#import  │ 事务 T1
│     │                 │     └── .save! / .update!            │
│     │                 └── plaid_item.update!(next_cursor)    │
│     │           └────────────────────────────────────────────┘
│     │
│     └── 阶段二: process_accounts (遍历每个 PlaidAccount)
│           │
│           └── 每个 PlaidAccount:
│                 │
│                 ├── process_account!
│                 │     └── PlaidAccount.transaction do ─────┐
│                 │           ├── find_or_initialize_by     │ 事务 T2n
│                 │           ├── enrich_attributes          │
│                 │           ├── assign_attributes          │
│                 │           ├── account.save!              │
│                 │           └── set_current_balance        │
│                 └──────────────────────────────────────────┘
│                 │
│                 ├── process_transactions (无事务, rescue 捕获)
│                 ├── process_investments (无事务, rescue 捕获)
│                 └── process_liabilities (无事务, rescue 捕获)
│
├── rescue => e (捕获阶段一/二抛出的异常)
│     ├── Sync#fail! (单独事务)
│     ├── sync.update(error: e.message) (单独事务)
│     └── report_error (Sentry)
│
└── ensure: finalize_if_all_children_finalized (Sync.transaction)
```

### 7.2 各事务边界详解

#### 事务 T1：账户数据导入事务

**位置** `app/models/plaid_item/importer.rb` (L38-L56)

```ruby
def fetch_and_import_accounts_data
  snapshot = PlaidItem::AccountsSnapshot.new(plaid_item, plaid_provider: plaid_provider)

  PlaidItem.transaction do  # ← 事务 T1 开始
    snapshot.accounts.each do |raw_account|
      plaid_account = plaid_item.plaid_accounts.find_or_initialize_by(
        plaid_id: raw_account.account_id
      )

      PlaidAccount::Importer.new(
        plaid_account,
        account_snapshot: snapshot.get_account_data(raw_account.account_id)
      ).import  # 内部调用 .save!
    end

    plaid_item.update!(next_cursor: snapshot.transactions_cursor)
  end  # ← 事务 T1 结束（commit 或 rollback）
end
```

**事务 T1 包含的操作：**
| 操作 | 说明 |
|-----|------|
| `find_or_initialize_by` | 查找或初始化 PlaidAccount（纯读） |
| `PlaidAccount::Importer#import` | 内部调用 `.save!` / `.update!`（写） |
| `plaid_item.update!(next_cursor)` | 更新游标（写） |

**事务 T1 回滚条件：**
- 任何一次 `.save!` 失败（验证失败等）
- `plaid_item.update!` 失败
- 事务内任何其他异常

**事务 T1 不包含的操作（在事务外执行）：**
| 操作 | 说明 |
|-----|------|
| `get_item_accounts` | Plaid API 调用 |
| `get_transactions` | Plaid API 调用 |
| `get_investments` | Plaid API 调用 |
| `get_liabilities` | Plaid API 调用 |

**⚠️ 重要：** Plaid API 调用在事务外执行。如果 API 调用抛 `ITEM_NOT_FOUND`，**不会触发事务回滚**（因为事务还没开始，或者已经在异常抛出点之前）。

#### 事务 T2n：单个账户处理事务（每个 PlaidAccount 一个独立事务）

**位置** `app/models/plaid_account/processor.rb` (L31-L62)

```ruby
def process_account!
  PlaidAccount.transaction do  # ← 事务 T2n 开始
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
  end  # ← 事务 T2n 结束
end
```

**事务 T2n 包含的操作：**
| 操作 | 说明 |
|-----|------|
| `find_or_initialize_by` | 查找或创建 Account（读/可能写） |
| `enrich_attributes` | 设置属性（内存操作） |
| `assign_attributes` | 设置属性（内存操作） |
| `account.save!` | 保存 Account（写） |
| `set_current_balance` | 创建/更新估值锚点（可能写） |

**事务 T2n 回滚条件：**
- `account.save!` 失败
- `set_current_balance` 内部抛异常
- 映射类型时抛 `UnknownAccountTypeError`

**事务 T2n 失败后的影响：**
| 行为 | 说明 |
|-----|------|
| 当前 Account | 回滚（不会创建/更新） |
| 其他 PlaidAccount 的事务 | ❌ 不受影响（各自独立） |
| 同步整体状态 | ❌ 会失败（异常上抛到 Sync#perform） |

**⚠️ 注意：** `process_account!` 没有 `rescue`，异常会直接向上传播，导致整个同步失败。但由于每个账户的事务是独立的，**已成功提交的其他账户事务不会回滚**。

#### 事务 T3：Sync 状态变更事务

**位置** `app/models/sync.rb` (L68-L79)

```ruby
start!  # 单独事务

begin
  syncable.perform_sync(self)
rescue => e
  fail!  # 单独事务
  update(error: e.message)  # 单独事务
  report_error(e)
ensure
  finalize_if_all_children_finalized  # Sync.transaction
end
```

### 7.3 异常传播与回滚矩阵

#### 阶段一（Import）异常传播

| 异常来源 | 捕获位置 | 处理方式 | 回滚范围 |
|---------|---------|---------|---------|
| `fetch_and_import_item_data` 中 API 报错 | `handle_plaid_error` | `ITEM_LOGIN_REQUIRED` → 状态标记<br>其他 → `raise` | ❌ 无（在事务外） |
| 事务 T1 内 `.save!` 失败 | 无（直接上抛） | 异常上抛 | ✅ 事务 T1 内所有操作回滚 |
| 事务 T1 外 API 报错（如 `ITEM_NOT_FOUND`） | `handle_plaid_error` → `raise` | 异常上抛 | ❌ 无（API 调用在事务外） |

#### 阶段二（Process）异常传播

| 异常来源 | 捕获位置 | 处理方式 | 回滚范围 |
|---------|---------|---------|---------|
| `process_account!` 内事务 T2n 失败 | 无（直接上抛） | 异常上抛 | ✅ 当前事务 T2n 回滚<br>❌ 其他已提交的 T2n 不回滚 |
| `process_transactions` 内部异常 | `rescue => e` | Sentry 上报，继续 | ❌ 无回滚 |
| `process_investments` 内部异常 | `rescue => e` | Sentry 上报，继续 | ❌ 无回滚 |
| `process_liabilities` 内部异常 | `rescue => e` | Sentry 上报，继续 | ❌ 无回滚 |

### 7.4 错误处理策略分类

#### 分类 A：必须成功 → 异常上抛 → Sync 失败

**特征：** 没有 `rescue` 或 `rescue` 后重新 `raise`

**包含：**
| 位置 | 行为 |
|-----|------|
| `PlaidItem::Importer#fetch_and_import_item_data` | 无 rescue，异常直接上抛 |
| `PlaidItem::Importer#fetch_and_import_accounts_data` | 事务内 `.save!` 失败上抛 |
| `PlaidItem::Importer#handle_plaid_error` | 非 `ITEM_LOGIN_REQUIRED` 的错误 `raise` |
| `PlaidAccount::Processor#process_account!` | 无 rescue，事务内失败上抛 |
| `Sync#perform` | `rescue` 后 `fail!` 并记录错误 |

**结果：**
- Sync 状态：`syncing` → `failed`
- 错误信息：`sync.error = e.message`
- 已提交的独立事务：不会回滚

#### 分类 B：状态标记 → 不上抛 → 同步继续

**特征：** `rescue` 后更新数据库状态，不抛异常

**包含：**
| 位置 | 行为 |
|-----|------|
| `PlaidItem::Importer#handle_plaid_error` | `ITEM_LOGIN_REQUIRED` → `update!(status: :requires_update)` |
| `PlaidItem#get_update_link_token` | `ITEM_NOT_FOUND` → `update!(status: :requires_update)` + Sentry + return nil |

**结果：**
- PlaidItem.status: `good` → `requires_update`
- 同步流程：可能继续或终止（取决于具体位置）
- 无事务回滚

#### 分类 C：静默吞掉 → 仅 Sentry 上报 → 继续执行

**特征：** `rescue => e` + `report_exception` 或无操作

**包含：**
| 位置 | 行为 |
|-----|------|
| `PlaidAccount::Processor#process_transactions` | `rescue => e` + `report_exception(e)` |
| `PlaidAccount::Processor#process_investments` | `rescue => e` + `report_exception(e)` |
| `PlaidAccount::Processor#process_liabilities` | `rescue => e` + `report_exception(e)` |
| `PlaidItem#remove_plaid_item` | `ITEM_NOT_FOUND` 时不抛异常，继续删除 |
| `PlaidItem::WebhookProcessor#process` | `rescue => e` + `Sentry.capture_exception` |

**结果：**
- 单个步骤失败不影响整体流程
- 错误仅记录到 Sentry
- 无事务回滚

#### 分类 D：删除失败 → 状态重置 → 允许重试

**特征：** `rescue` 后重置状态标记

**包含：**
| 位置 | 行为 |
|-----|------|
| `DestroyJob#perform` | `rescue => e` + `update!(scheduled_for_deletion: false)` |

**结果：**
- 数据库记录保留
- `scheduled_for_deletion` 重置为 `false`
- 用户可重新触发删除

### 7.5 事务边界总结图

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        SyncJob#perform (无外层事务)                        │
└──────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────────────┐
│  start! (独立事务 A)                                                      │
│  └── Sync: pending → syncing                                              │
└──────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────────────┐
│  阶段一: import_latest_plaid_data                                         │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │  fetch_and_import_item_data (无事务)                               │  │
│  │  ├── get_item API (事务外) ← ITEM_NOT_FOUND 在此抛异常              │  │
│  │  ├── get_institution API (事务外)                                  │  │
│  │  ├── .save! (独立事务 B1: 更新 plaid_item)                          │  │
│  │  └── .save! (独立事务 B2: 更新 institution)                         │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │  fetch_and_import_accounts_data                                    │  │
│  │  ├── get_item_accounts API (事务外)                                │  │
│  │  ├── get_transactions API (事务外)                                 │  │
│  │  ├── get_investments API (事务外)                                  │  │
│  │  ├── get_liabilities API (事务外)                                  │  │
│  │  │                                                                  │  │
│  │  └── 事务 T1 (PlaidItem.transaction)                                │  │
│  │      ├── PlaidAccount1 导入 ← 失败 → T1 回滚 → 异常上抛             │  │
│  │      ├── PlaidAccount2 导入                                         │  │
│  │      ├── PlaidAccount3 导入                                         │  │
│  │      └── update!(next_cursor) ← 全部成功才执行                      │  │
│  └────────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────────────┐
│  阶段二: process_accounts                                                 │
│                                                                          │
│  遍历每个 PlaidAccount:                                                  │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │  PlaidAccount1                                                      │  │
│  │  ├── 事务 T2_1 (PlaidAccount.transaction)                            │  │
│  │  │   ├── Account find_or_create                                     │  │
│  │  │   ├── account.save!                                              │  │
│  │  │   └── set_current_balance                                        │  │
│  │  └── 若失败 → T2_1 回滚 → 异常上抛 → 整个同步失败                     │  │
│  │                                                                      │  │
│  │  ├── process_transactions ← rescue 吞掉 → 仅 Sentry 上报            │  │
│  │  ├── process_investments ← rescue 吞掉 → 仅 Sentry 上报            │  │
│  │  └── process_liabilities ← rescue 吞掉 → 仅 Sentry 上报            │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │  PlaidAccount2 (同上，独立事务 T2_2)                                │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  ⚠️ 注意：T2_1、T2_2 是独立事务。若 T2_2 失败，T2_1 已提交的数据不会回滚  │
└──────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────────────┐
│  异常处理 (rescue => e)                                                   │
│  ├── fail! (独立事务 C: Sync syncing → failed)                           │
│  ├── update(error: e.message) (独立事务 D)                                │
│  └── Sentry.capture_exception                                             │
└──────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────────────┐
│  ensure: finalize_if_all_children_finalized                              │
│  └── Sync.transaction (事务 E)                                            │
│      ├── 若 all_children_finalized?                                       │
│      │   ├── 有 failed 子同步 → fail!                                     │
│      │   └── 无 failed 子同步 → complete!                                 │
│      └── perform_post_sync                                                │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 八、Webhook 处理

### 8.1 Webhook 验证

**位置** `app/models/provider/plaid.rb` (L14-L43)

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

**三重验证：**
1. JWT 签名验证（ES256）
2. 时间戳检查（5 分钟内）
3. 请求体哈希验证

### 8.2 Webhook 处理器

**位置** `app/models/plaid_item/webhook_processor.rb` (L1-L56)

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

**Webhook 类型处理：**

| Webhook 类型 | Code | 处理方式 |
|-------------|------|---------|
| TRANSACTIONS | SYNC_UPDATES_AVAILABLE | 触发同步 |
| INVESTMENTS_TRANSACTIONS | DEFAULT_UPDATE | 触发同步 |
| HOLDINGS | DEFAULT_UPDATE | 触发同步 |
| ITEM | ERROR + ITEM_LOGIN_REQUIRED | 标记 requires_update |
| ITEM | ERROR + 其他 error_code | **静默忽略**（不打 warn，不报错，不更新状态） |
| ITEM | ERROR + error 为 nil | **NoMethodError → Sentry** |

**⚠️ 注意：** Webhook 中的 `ITEM.ERROR` 事件处理逻辑如下：

```ruby
when [ "ITEM", "ERROR" ]
  if error["error_code"] == "ITEM_LOGIN_REQUIRED"  # ⚠️ 如果 error 为 nil，会抛 NoMethodError
    plaid_item.update!(status: :requires_update)
  end
  # 其他 error_code（如 ITEM_NOT_FOUND）：什么都不做，直接跳过
```

**完整调用链：**
```ruby
def process
  case [ webhook_type, webhook_code ]
  when [ "ITEM", "ERROR" ]
    if error["error_code"] == "ITEM_LOGIN_REQUIRED"  # ← error 为 nil 时抛 NoMethodError
      plaid_item.update!(status: :requires_update)
    end
  else
    Rails.logger.warn("Unhandled Plaid webhook type: ...")
  end
rescue => e  # ← 外层 rescue 捕获 NoMethodError
  Sentry.capture_exception(e)  # ← 上报 Sentry
end
```

**各种场景的实际表现：**

| 场景 | 处理方式 | 日志/上报 | 状态变化 |
|-----|---------|---------|---------|
| `ITEM.ERROR` + `error_code = ITEM_LOGIN_REQUIRED` | `update!(status: :requires_update)` | 无 | `good` → `requires_update` |
| `ITEM.ERROR` + `error_code = ITEM_NOT_FOUND` | **静默跳过** | ❌ 无 warn，❌ 无 Sentry | ❌ 无 |
| `ITEM.ERROR` + `error` 为 `nil`（Plaid 未传 error 字段） | `nil["error_code"]` → `NoMethodError` → `rescue` 捕获 | ✅ `Sentry.capture_exception(e)` | ❌ 无 |
| `ITEM.ERROR` + 其他错误码 | 静默跳过 | ❌ 无 warn，❌ 无 Sentry | ❌ 无 |
| **未匹配的 webhook_type/code**（如 `ITEM.WEBHOOK_UPDATE_ACKNOWLEDGED`） | 走 `else` 分支 | ✅ `Rails.logger.warn("Unhandled Plaid webhook type: ...")` | ❌ 无 |

**关键区别：**
- `ITEM.ERROR` + 有 `error` 字段但非 ITEM_LOGIN_REQUIRED：**静默忽略**，没有任何日志
- `ITEM.ERROR` + **error 为 nil**：抛 `NoMethodError`，被外层 `rescue` 捕获后 **上报 Sentry**
- 其他未匹配的 webhook 类型/代码组合：**会打 warn 日志**

### 8.3 Webhook 中的"缺失 Item"场景

**位置** `app/models/plaid_item/webhook_processor.rb` (L44-L55)

```ruby
def handle_missing_item
  return if plaid_item.present?

  # If we cannot find an item in our DB, that means we've reached an invalid data state where
  # the Plaid Item (upstream) still exists (and is being billed), but doesn't exist internally.
  #
  # Since we don't have the item which has the access token, there is nothing we can do programmatically
  # here, so we just need to report it to Sentry and manually handle it.
  Sentry.capture_exception(MissingItemError.new("Received Plaid webhook for item no longer in our DB.  Manual action required to resolve.")) do |scope|
    scope.set_tags(plaid_item_id: item_id)
  end
end
```

**场景说明：**
- 这是 **反向的** `ITEM_NOT_FOUND`：Plaid 侧的 item 还在，但本地数据库已删除
- 系统无法自动处理（没有 access_token 无法调用 Plaid API）
- 仅上报 Sentry，等待人工处理

---

## 九、删除流程详解

### 9.1 删除调用链

```
用户点击删除
    │
    ▼
PlaidItemsController#destroy
    │
    └── @plaid_item.destroy_later
          │
          ├── update!(scheduled_for_deletion: true)  ← 独立事务
          │
          └── DestroyJob.perform_later
                │
                ▼
          DestroyJob#perform
                │
                └── PlaidItem#destroy
                      │
                      ├── before_destroy :remove_plaid_item
                      │     │
                      │     └── plaid_provider.remove_item(access_token)
                      │           │
                      │           ├── 成功 → 继续
                      │           │
                      │           └── Plaid::ApiError
                      │                 │
                      │                 ├── ITEM_NOT_FOUND → 静默忽略
                      │                 │
                      │                 └── 其他错误 → raise e → 删除中断
                      │
                      └── 级联删除 (dependent: :destroy)
                            │
                            ├── PlaidAccount.destroy_all
                            │     │
                            │     └── Account.destroy (每个 PlaidAccount has_one :account)
                            │
                            └── 其他关联数据删除
```

### 9.2 删除相关代码

**软删除标记** `app/models/plaid_item.rb` (L44-L47)

```ruby
def destroy_later
  update!(scheduled_for_deletion: true)
  DestroyJob.perform_later(self)
end
```

**Plaid 侧删除** `app/models/plaid_item.rb` (L100-L112)

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

**级联关系** `app/models/plaid_item.rb` (L18-L19)

```ruby
has_many :plaid_accounts, dependent: :destroy
has_many :accounts, through: :plaid_accounts
```

**PlaidAccount 级联** `app/models/plaid_account.rb` (L4)

```ruby
has_one :account, dependent: :destroy
```

**DestroyJob 异常处理** `app/jobs/destroy_job.rb` (L1-L9)

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

### 9.3 删除失败场景

| 失败原因 | 处理方式 | 结果 |
|---------|---------|------|
| Plaid API 返回 `ITEM_NOT_FOUND` | 静默忽略 | 本地删除继续执行，成功 |
| Plaid API 返回其他错误 | `raise e` | 删除中断，`scheduled_for_deletion` 重置为 false |
| 数据库删除失败 | `rescue => e` | `scheduled_for_deletion` 重置为 false |

---

## 十、完整时序图

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
 │                       │                        │──事务 T1 开始──▶│
 │                       │                        │──INSERT/UPDATE plaid_accounts──▶│
 │                       │                        │──UPDATE next_cursor──▶│
 │                       │                        │──事务 T1 提交──▶│
 │                       │                        │                        │
 │                       │                        │                        │
 │                       │                        │──process_accounts──▶│
 │                       │                        │                        │
 │                       │                        │──事务 T2_1 (Account1)──▶│
 │                       │                        │──find_or_create accounts──▶│
 │                       │                        │──事务 T2_1 提交──▶│
 │                       │                        │                        │
 │                       │                        │──事务 T2_2 (Account2)──▶│
 │                       │                        │──事务 T2_2 提交──▶│
 │                       │                        │                        │
 │                       │                        │──schedule_account_syncs──▶│
 │                       │                        │                        │
```

---

## 十一、关键设计亮点

### 11.1 三层数据存储策略

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

### 11.2 事务策略

| 策略 | 应用场景 | 目的 |
|-----|---------|------|
| **大事务 T1** | 账户批量导入 | 保证所有 PlaidAccount 导入和 cursor 更新的原子性 |
| **独立小事务 T2n** | 每个账户的领域模型转换 | 账户间隔离，一个失败不影响其他已提交的 |
| **无事务 API 调用** | 所有 Plaid API 调用 | 避免持有数据库锁等待外部 IO |
| **状态标记而非回滚** | 错误恢复场景 | 允许用户手动干预和重试 |

### 11.3 错误处理分层

| 层级 | 处理方式 | 示例 |
|-----|---------|------|
| **必须成功层** | 异常上抛，Sync 失败 | 账户基础信息导入、类型映射 |
| **优雅降级层** | 状态标记，等待用户 | ITEM_LOGIN_REQUIRED → requires_update |
| **静默忽略层** | Sentry 上报，继续执行 | 交易/投资/负债处理失败 |
| **预期容错层** | 特殊错误码处理 | ITEM_NOT_FOUND 在删除时的处理 |

### 11.4 安全措施

1. **Token 加密**
   - `encrypts :access_token, deterministic: true`

2. **Webhook 验证**
   - JWT 签名验证
   - 时间戳检查（5 分钟内）
   - 请求体哈希验证

3. **错误上报**
   - Sentry 集成
   - 详细的上下文标签

---

## 十二、相关文件索引

| 文件路径 | 说明 |
|---------|------|
| `app/controllers/plaid_items_controller.rb` | Plaid 项目控制器 |
| `app/models/family/plaid_connectable.rb` | Family 与 Plaid 连接模块 |
| `app/models/provider/plaid.rb` | Plaid API 提供者 |
| `app/models/plaid_item.rb` | PlaidItem 主模型（含 ITEM_NOT_FOUND 处理） |
| `app/models/plaid_item/importer.rb` | PlaidItem 数据导入器（事务 T1） |
| `app/models/plaid_item/syncer.rb` | PlaidItem 同步器 |
| `app/models/plaid_item/accounts_snapshot.rb` | 账户数据快照 |
| `app/models/plaid_item/webhook_processor.rb` | Webhook 处理器 |
| `app/models/plaid_account.rb` | PlaidAccount 模型 |
| `app/models/plaid_account/importer.rb` | PlaidAccount 数据导入器 |
| `app/models/plaid_account/processor.rb` | PlaidAccount 处理器（事务 T2n） |
| `app/models/plaid_account/type_mappable.rb` | 类型映射模块 |
| `app/models/concerns/syncable.rb` | 同步通用模块 |
| `app/models/sync.rb` | Sync 记录模型（状态机） |
| `app/jobs/sync_job.rb` | 同步作业 |
| `app/jobs/destroy_job.rb` | 销毁作业 |
| `app/javascript/controllers/plaid_controller.js` | 前端 Plaid 控制器 |
| `config/initializers/plaid.rb` | Plaid 配置初始化 |
| `config/routes.rb` | 路由配置 |
| `db/schema.rb` | 数据库结构 |

---

## 十三、附录：主要流程总结

### 13.1 新建银行连接

1. 用户点击"连接银行"
2. 后端生成 Link Token
3. 前端打开 Plaid Link 界面
4. 用户完成银行授权
5. Plaid 返回 Public Token
6. 后端交换为 Access Token
7. 创建 PlaidItem 记录
8. 触发第一次同步

### 13.2 同步流程

1. 拉取 Item 和 Institution 数据（无事务）
2. 拉取账户列表（无事务）
3. 根据产品支持情况拉取交易/投资/负债数据（无事务）
4. **事务 T1** 内更新所有 PlaidAccount + cursor
5. 遍历每个账户，**事务 T2n** 内创建/更新本地 Account
6. 处理交易、投资、负债（独立 rescue，不影响整体）
7. 调度账户级同步

### 13.3 ITEM_NOT_FOUND 在各阶段

| 阶段 | 处理方式 | 日志/上报 | 状态变化 |
|-----|---------|---------|---------|
| 导入阶段 | 异常上抛 → Sync failed | `sync.error = e.message` | PlaidItem.status 不变 |
| 更新 Token 阶段 | 标记 requires_update + Sentry | Sentry.capture_exception | `good` → `requires_update` |
| 删除阶段 | 静默忽略 → 继续本地删除 | 无 | N/A（已删除） |
| **Webhook 阶段** | **静默跳过**（在 ITEM.ERROR 分支内） | ❌ 无 warn，❌ 无 Sentry | ❌ 无 |

**⚠️ Webhook 阶段注意事项：**
- `ITEM.ERROR` + `ITEM_NOT_FOUND`：静默跳过，没有任何日志
- `ITEM.ERROR` + 其他错误码：同样静默跳过
- `error` 字段为 `nil`：同样静默跳过
- 只有**未匹配** `case/when` 分支的 webhook 类型才会打 warn 日志

### 13.4 事务回滚边界

| 事务 | 回滚触发条件 | 回滚范围 |
|-----|-------------|---------|
| **T1** (账户导入) | 事务内任何 `.save!` 失败 | T1 内所有 PlaidAccount 操作 + cursor 更新 |
| **T2n** (单个账户处理) | `account.save!` 失败或映射异常 | 当前 T2n 内 Account 操作 |
| **无事务** (API 调用) | 任何 API 错误 | ❌ 无（异常上抛导致 Sync 失败，但已提交的事务不回滚） |

### 13.5 错误处理分类

| 分类 | 处理方式 | 示例错误码/场景 |
|-----|---------|----------------|
| **必须成功** | 异常上抛 → Sync failed | 类型映射错误、`save!` 验证失败 |
| **状态标记** | 更新 status，不抛异常 | `ITEM_LOGIN_REQUIRED`、更新时的 `ITEM_NOT_FOUND` |
| **静默忽略** | Sentry 上报，继续执行 | 交易/投资/负债处理失败、删除时的 `ITEM_NOT_FOUND` |
| **状态重置** | 重置标记，允许重试 | 删除失败时 `scheduled_for_deletion = false` |
