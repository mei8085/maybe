# AI 账户查询机制与权限边界

## 概述

Maybe Finance 的 AI 助手通过工具函数（Function Calling）机制，为用户提供账户数据查询能力。本文档详细说明 AI 助手可调用的查询维度、权限校验机制、各查询间的过滤差异以及对回答口径的影响。

**修订说明**：本文档校正了各查询函数对停用/待删除账户的过滤行为差异，以及分页机制的真实限制。

---

## 一、查询能力与过滤行为

AI 助手可调用 4 个核心工具函数。各函数对"停用账户"（`disabled`）和"待删除账户"（`pending_deletion`）的过滤行为**不一致**，这是理解回答口径差异的关键。

### 1.1 账户状态说明

账户有 4 种状态（`app/models/account.rb:34-55`）：

| 状态 | 说明 | `visible` scope |
|------|------|----------------|
| `active` | 正常活跃账户 | ✅ 包含 |
| `draft` | 草稿/创建中账户 | ✅ 包含 |
| `disabled` | 已停用账户 | ❌ 排除 |
| `pending_deletion` | 标记待删除账户 | ❌ 排除 |

**`visible` scope 定义**：`app/models/account.rb:21`
```ruby
scope :visible, -> { where(status: [ "draft", "active" ]) }
```

---

### 1.2 各查询函数的过滤行为对比

| 查询函数 | 过滤停用/待删除账户 | 实现方式 | 代码位置 |
|---------|-------------------|---------|---------|
| **GetAccounts** | ❌ **不过滤** | `family.accounts.includes(:balances)` | `get_accounts.rb:15` |
| **GetTransactions** | ✅ **过滤** | `Transaction::Search` 默认 `active_accounts_only: true` | `get_transactions.rb:137` |
| **GetBalanceSheet** - 当前值 | ✅ **过滤** | `BalanceSheet::AccountTotals.visible_accounts` | `account_totals.rb:27` |
| **GetBalanceSheet** - 历史数据 | ✅ **过滤** | `family.accounts.visible` | `get_balance_sheet.rb:47` |
| **GetIncomeStatement** | ✅ **过滤** | `family.transactions.visible` → `Entry.visible` | `income_statement.rb:13` |

---

### 1.3 GetAccounts - 账户信息查询（不过滤）

**功能**：获取用户所有账户的基本信息和历史余额

**⚠️ 关键特性**：**不**过滤停用和待删除账户

**代码实现**：`app/models/assistant/function/get_accounts.rb:15`
```ruby
accounts: family.accounts.includes(:balances).map do |account|
  # 返回所有账户，包括 disabled 和 pending_deletion
end
```

**查询维度**：
- 账户名称
- 当前余额（原始值 + 格式化显示）
- 货币类型
- 账户分类（资产/负债）
- 账户类型（Depository、CreditCard、Investment、Crypto、Loan、Property 等）
- 账户开户日期
- 是否通过 Plaid 自动同步
- **账户状态**（active、draft、disabled、pending_deletion）
- 历史余额（过去 5 年，按月统计）

**返回示例**：
```json
{
  "as_of_date": "2024-05-14",
  "accounts": [
    {
      "name": "Checking Account",
      "balance": 5000.00,
      "currency": "USD",
      "balance_formatted": "$5,000.00",
      "classification": "asset",
      "type": "Depository",
      "start_date": "2020-01-01",
      "is_plaid_linked": true,
      "status": "active",
      "historical_balances": {
        "start_date": "2019-05-14",
        "end_date": "2024-05-14",
        "interval": "1 month",
        "values": ["$4,000.00", "$4,200.00", "..."]
      }
    },
    {
      "name": "Old Car Loan",
      "balance": 0.00,
      "currency": "USD",
      "balance_formatted": "$0.00",
      "classification": "liability",
      "type": "Loan",
      "start_date": "2018-01-01",
      "is_plaid_linked": false,
      "status": "disabled",
      "historical_balances": {
        "start_date": "2019-05-14",
        "end_date": "2024-05-14",
        "interval": "1 month",
        "values": ["$15,000.00", "$10,000.00", "...", "$0.00"]
      }
    }
  ]
}
```

**注意**：返回结果中的 `status` 字段明确标识账户状态，但该函数不会根据状态过滤任何账户。

---

### 1.4 GetTransactions - 交易搜索与筛选（过滤）

**功能**：多维度搜索用户交易记录，支持分页

**⚠️ 关键特性**：**会**过滤停用账户下的交易

**代码实现**：`app/models/transaction/search.rb:16, 30, 82-88`
```ruby
attribute :active_accounts_only, :boolean, default: true  # 默认过滤停用账户

def transactions_scope
  query = family.transactions
  query = apply_active_accounts_filter(query, active_accounts_only)  # 应用过滤
  ...
end

def apply_active_accounts_filter(query, active_accounts_only_filter)
  if active_accounts_only_filter
    query.where(accounts: { status: [ "draft", "active" ] })  # 仅可见账户
  else
    query
  end
end
```

**查询维度**：

| 维度 | 说明 |
|------|------|
| 时间范围 | start_date ~ end_date（YYYY-MM-DD 格式） |
| 关键词搜索 | search（按交易名称搜索） |
| 金额筛选 | amount + amount_operator（equal/less/greater） |
| 账户筛选 | accounts（按账户名称数组筛选） |
| 分类筛选 | categories（按分类名称数组筛选） |
| 商家筛选 | merchants（按商家名称数组筛选） |
| 标签筛选 | tags（按标签名称数组筛选） |
| 排序 | order（asc/desc 按日期排序） |
| 分页 | page（页码，从 1 开始） |

**返回维度**：
- 交易日期
- 金额（绝对值 + 格式化）
- 货币类型
- 收支分类（income/expense）
- 所属账户
- 分类（category）
- 商家（merchant）
- 标签列表（tags）
- 是否为转账（is_transfer）
- 汇总统计（total_results、total_pages、total_income、total_expenses）

**账户筛选选项限制**：

`accounts` 参数的可选项来自 `family_account_names`（`app/models/assistant/function.rb:58-60`）：
```ruby
def family_account_names
  @family_account_names ||= family.accounts.visible.pluck(:name)  # 仅返回可见账户
end
```

这意味着：
- AI 只能选择 `visible` 状态的账户进行筛选
- 与 GetAccounts 返回的完整账户列表**不一致**
- 如果用户询问"我那个已停用的旧信用卡上的交易"，AI 无法通过参数直接筛选

---

### 1.5 GetBalanceSheet - 资产负债表与净值分析（过滤）

**功能**：获取用户资产负债概况和历史趋势

**⚠️ 关键特性**：**会**过滤停用账户，**当前值与历史数据一致**

**代码实现**：

**当前净值计算**（`app/models/balance_sheet/account_totals.rb:26-28`）：
```ruby
def visible_accounts
  @visible_accounts ||= family.accounts.visible.with_attached_logo
end
```

**历史数据计算**（`app/models/assistant/function/get_balance_sheet.rb:47`）：
```ruby
def historical_data(period, classification: nil)
  scope = family.accounts.visible  # 仅可见账户
  ...
end
```

**查询维度**：
- 当前净值（Net Worth）
- 当前总资产
- 当前总负债
- 历史趋势（过去 5 年净值变化，按月）
- 资产历史趋势（按月）
- 负债历史趋势（按月）
- 债务资产比（debt_to_asset_ratio）

**返回示例**：
```json
{
  "as_of_date": "2024-05-14",
  "oldest_account_start_date": "2020-01-01",
  "currency": "USD",
  "net_worth": {
    "current": "$150,000.00",
    "monthly_history": {...}
  },
  "assets": {
    "current": "$200,000.00",
    "monthly_history": {...}
  },
  "liabilities": {
    "current": "$50,000.00",
    "monthly_history": {...}
  },
  "insights": {
    "debt_to_asset_ratio": "25%"
  }
}
```

**一致性说明**：
- 当前值和历史数据**都**使用 `family.accounts.visible`
- 债务资产比基于过滤后的数据计算
- 不会出现"历史包含停用账户、当前不包含"的跳变问题

---

### 1.6 GetIncomeStatement - 收入支出分析（过滤）

**功能**：按时间段分析收入支出构成

**⚠️ 关键特性**：**会**过滤停用账户下的交易

**代码实现**：`app/models/income_statement.rb:13, 65`
```ruby
def totals(transactions_scope: nil)
  transactions_scope ||= family.transactions.visible  # 仅可见账户的交易
  ...
end

def build_period_total(classification:, period:)
  totals = totals_query(transactions_scope: family.transactions.visible.in_period(period))...
end
```

**`Entry.visible` 定义**：`app/models/entry.rb:17-19`
```ruby
scope :visible, -> {
  joins(:account).where(accounts: { status: [ "draft", "active" ] })
}
```

**查询参数**：
- start_date：统计起始日期
- end_date：统计结束日期

**返回维度**：
- 收入总计
- 支出总计
- 收入分类明细（含子分类、百分比占比）
- 支出分类明细（含子分类、百分比占比）
- 洞察指标：
  - 净收入（net_income）
  - 储蓄率（savings_rate）
  - 月均收入中位数
  - 月均支出中位数
  - 月均支出平均值

**注意**：月均统计（median/avg）也基于相同的过滤逻辑。

---

## 二、分页机制的真实限制

### 2.1 Schema 声明 vs 实际行为

**声明的分页参数**（`app/models/assistant/function/get_transactions.rb:71`）：
```ruby
required: [ "order", "page", "page_size" ],
properties: {
  page: { type: "integer", description: "Page number" },
  ...
}
```

**实际实现**（`app/models/assistant/function/get_transactions.rb:142-150`）：
```ruby
pagy, paginated_transactions = pagy(
  pagy_query.includes(...),
  page: params["page"] || 1,      # 仅使用 page 参数
  limit: default_page_size        # 固定使用默认值，忽略 params["page_size"]
)
```

### 2.2 真实限制

| 限制项 | 真实值 | 说明 |
|--------|--------|------|
| 每页大小 | **固定 50 条** | `page_size` 参数在 Schema 中声明为 required，但实际代码中**完全忽略** |
| 页码 | 从 1 开始 | 使用 `params["page"] \|\| 1` |
| 设计目的 | 强制 AI 使用筛选条件 | 代码注释："By default, we give a small page size to force the AI to use filters effectively and save on tokens" |

### 2.3 Schema 与实现不一致问题

- Schema 声明 `page_size` 为 `required`
- 但代码中 `limit: default_page_size` 硬编码为 50
- 这意味着：
  - AI 可能尝试传递 `page_size` 参数（因为 Schema 要求）
  - 但该参数会被**静默忽略**
  - AI 无法调整每页大小，始终只能获取 50 条/页
  - 返回结果中 `page_size` 字段始终为 50

---

## 三、过滤差异对回答口径的影响

### 3.1 问题场景：用户询问"我有多少个账户？"

**可能的回答路径**：

| 路径 | AI 调用函数 | 返回结果 | 回答内容 |
|------|------------|---------|---------|
| A | GetAccounts | 8 个账户（含 2 个已停用） | "您有 8 个账户" |
| B | GetBalanceSheet | 基于 6 个可见账户计算净值 | "您的净值由 6 个账户构成" |

**口径差异**：同一用户同一时刻的问题，可能得到不同的"账户数量"答案。

---

### 3.2 问题场景：用户询问"我去年还了多少贷款？"

**假设**：用户的房贷账户在 3 个月前被标记为 `disabled`

| 查询函数 | 是否包含该账户 | 对回答的影响 |
|---------|---------------|-------------|
| GetAccounts | ✅ 包含 | AI 会看到这个贷款账户存在 |
| GetTransactions | ❌ 不包含 | 搜索不到该账户下的还款记录 |
| GetIncomeStatement | ❌ 不包含 | 支出统计中缺少该账户的还款 |
| GetBalanceSheet | ❌ 不包含 | 负债中不包含该贷款余额 |

**口径差异**：AI 可能会困惑"账户列表显示有这笔贷款，但交易查询找不到相关记录"。

---

### 3.3 问题场景：用户询问"我 2023 年在信用卡上花了多少钱？"

**假设**：用户 2024 年初注销了旧信用卡（标记为 `disabled`）

| 查询函数 | 是否包含 2023 年数据 | 对回答的影响 |
|---------|---------------------|-------------|
| GetTransactions | ❌ 不包含 | 无法查询该卡的历史交易 |
| GetIncomeStatement | ❌ 不包含 | 2023 年支出统计缺失该卡数据 |

**口径差异**：
- GetAccounts 会显示该信用卡（状态为 `disabled`）
- 但 AI 无法查询该卡的任何历史交易
- 用户可能得到"您没有该信用卡的交易记录"这样的回答，但账户列表明明存在

---

### 3.4 问题场景：用户询问"我可以用哪些账户筛选交易？"

**关键发现**：`family_account_names` 使用 `visible` scope

**代码**：`app/models/assistant/function.rb:58-60`
```ruby
def family_account_names
  @family_account_names ||= family.accounts.visible.pluck(:name)
end
```

**影响**：
- Schema 中 `accounts` 参数的 `enum` 选项仅包含可见账户
- AI 无法通过参数筛选已停用账户
- 即使用户明确指定"已停用账户 X"，AI 也可能无法选择（因为不在 enum 中）

---

### 3.5 影响总结

| 差异类型 | 影响场景 | 对回答的影响 |
|---------|---------|-------------|
| GetAccounts 与其他函数不一致 | 账户数量、资产构成 | 回答可能出现数字矛盾 |
| 停用账户的交易不可查 | 历史支出查询、预算对比 | 数据缺失但无明确提示 |
| Schema 与实现不一致 | 交易分页 | AI 无法调整每页大小，可能遗漏数据 |
| 账户筛选 enum 限制 | 按账户筛选交易 | 无法选择已停用账户 |

---

## 四、权限校验机制

### 4.1 整体架构

权限校验是一个多层级的防护体系，贯穿从请求入口到数据查询的全链路：

```
请求入口
    ↓
会话认证 (Session Cookie)
    ↓
用户认证 (Current.user)
    ↓
Chat 归属校验
    ↓
工具函数作用域隔离 (Family 级别)
    ↓
数据查询（各函数过滤行为见前文）
```

---

### 4.2 第一层：会话认证

**认证方式**：基于 Cookie 的会话令牌

**关键代码**：`app/controllers/concerns/authentication.rb:18-28`

```ruby
def authenticate_user!
  if session_record = find_session_by_cookie
    Current.session = session_record
  else
    redirect_to new_session_url
  end
end
```

**认证流程**：
1. 请求到达时，从 Cookie 中读取加密的 `session_token`
2. 通过 `Session.find_by(id: cookie_value)` 查找会话记录
3. 若会话有效，设置 `Current.session`
4. 否则重定向到登录页面

**安全特性**：
- Cookie 采用 `signed` + `permanent` 加密存储
- 设置 `httponly: true` 防止 XSS 攻击
- 会话与用户 IP、User-Agent 绑定（创建时记录）

---

### 4.3 第二层：Chat 归属校验

**机制**：Chat 必须属于当前登录用户

**关键代码**：
- `app/controllers/chats_controller.rb:8`：`@chats = Current.user.chats`
- `app/controllers/chats_controller.rb:51`：`@chat = Current.user.chats.find(params[:id])`

**校验逻辑**：
1. 查询时使用 `Current.user.chats` 而不是 `Chat` 直接查询
2. 这确保只能访问当前用户拥有的聊天记录
3. 即使知道其他用户的 Chat ID，也无法访问（会抛出 ActiveRecord::RecordNotFound）

---

### 4.4 第三层：工具函数作用域隔离

**核心原则**：所有数据查询都以 `user.family` 为作用域边界

**代码位置**：`app/models/assistant/function.rb:78-80`

```ruby
def family
  user.family
end
```

所有查询函数都继承自 `Assistant::Function`，通过以下方式隔离数据：

| 函数 | 作用域实现 |
|------|-----------|
| GetAccounts | `family.accounts.includes(:balances)` |
| GetTransactions | `Transaction::Search.new(family, filters: search_params)` |
| GetBalanceSheet | `family.balance_sheet` |
| GetIncomeStatement | `family.income_statement` |

**关键设计**：
- 工具函数初始化时传入的是 `chat.user`（`app/models/assistant.rb:69-71`）
- 所有数据查询都从 `family` 出发，而不是全局模型
- 这确保即使用户尝试通过 Prompt 注入，也无法跨越 Family 边界

---

### 4.5 数据模型层次结构

```
Family (隔离单元)
  ├── User (成员)
  │     ├── Chat (对话)
  │     └── Session (会话)
  ├── Account (账户)
  │     ├── Entry (账目条目)
  │     │     └── Transaction (交易)
  │     ├── Balance (余额记录)
  │     └── Holding (持仓)
  ├── Category (分类)
  ├── Tag (标签)
  └── Merchant (商家)
```

**隔离特性**：
- 所有数据通过 `belongs_to :family` 或 `through: :accounts` 关联到 Family
- 不存在跨 Family 的直接关联
- 查询时自动限制在当前用户的 Family 范围内

---

## 五、安全边界与限制

### 5.1 访问控制边界

**不能访问的数据**：
- 其他 Family 的任何数据
- 其他用户的 Chat 记录（即使同 Family）

**对停用/待删除账户的访问**（见前文详细分析）：
- GetAccounts：**可以**访问（不过滤）
- GetTransactions：**不可以**访问（默认过滤）
- GetBalanceSheet：**不可以**访问（过滤）
- GetIncomeStatement：**不可以**访问（过滤）

**代码位置**：
- `app/models/account.rb:21`：`scope :visible, -> { where(status: [ "draft", "active" ]) }`
- `app/models/assistant/function/get_balance_sheet.rb:47`：`scope = family.accounts.visible`
- `app/models/transaction/search.rb:16`：`active_accounts_only: true`

---

### 5.2 数据返回限制

**返回格式约束**：
- 所有货币值以原始数值 + 格式化字符串形式返回
- 历史数据限制：最多返回过去 5 年的数据
- 交易分页限制：**固定** 50 条/页（Schema 声明可配置，但实现硬编码）

**敏感数据保护**：
- 不返回完整的账户号码、凭证等敏感信息
- 不返回用户密码、API keys 等认证信息
- 不返回 Plaid access tokens 等第三方凭证

---

### 5.3 用户角色与权限

**角色定义**：`app/models/user.rb:23`

```ruby
enum :role, { member: "member", admin: "admin", super_admin: "super_admin" }, validate: true
```

| 角色 | AI 助手权限 | 说明 |
|------|------------|------|
| member | ✅ 可访问同 Family 数据 | 普通成员，可使用 AI 助手 |
| admin | ✅ 可访问同 Family 数据 | 家族管理员，AI 权限同 member |
| super_admin | ⚠️ 可模拟用户（需记录） | 超级管理员，支持模拟功能 |

---

### 5.4 模拟功能（Impersonation）

**适用场景**：仅 super_admin 可使用

**代码位置**：
- `app/models/current.rb:8-18`
- `app/models/impersonation_session.rb`
- `app/controllers/concerns/impersonatable.rb`

**模拟流程**：
1. super_admin 创建 `ImpersonationSession`（需目标用户同意 pending → in_progress）
2. 模拟期间 `Current.user` 返回被模拟用户
3. AI 助手以被模拟用户身份运行，访问其 Family 数据
4. 所有操作记录到 `ImpersonationSessionLog`

**安全措施**：
- 必须是 super_admin 才能发起模拟
- 不能模拟另一个 super_admin
- 所有操作（控制器、动作、路径、方法、IP）都会被记录
- 模拟会话有明确的生命周期（pending → in_progress → complete）

---

### 5.5 函数调用安全

**代码位置**：`app/models/assistant/function_tool_caller.rb`

**安全设计**：
1. 函数名称白名单：只能调用预定义的 4 个函数
2. 参数 Schema 验证：使用 JSON Schema 严格校验参数格式
3. 不支持递归调用：`responder.rb:42` 明确禁止 follow-up 的函数调用
4. 错误隔离：函数执行异常不会暴露系统内部信息

```ruby
# 关键限制：不支持递归的 function execution for follow-up response
# app/models/assistant/responder.rb:42
# We do not currently support function executions for a follow-up response
# (avoid recursive LLM calls that could lead to high spend)
```

---

## 六、系统指令中的安全约束

**代码位置**：`app/models/assistant/configurable.rb:25-79`

AI 系统指令中包含明确的行为约束：

```markdown
### Rules about financial advice

- Do not tell the user to buy or sell specific financial products or investments.
- Do not make assumptions about the user's financial situation.
  Use the functions available to get the data you need.

### Function calling rules

- Use the functions available to you to get user financial data and enhance your responses
- If you suspect that you do not have enough data to 100% accurately answer,
  be transparent about it and state exactly what the data you're presenting
  represents and what context it is in (i.e. date range, account, etc.)
```

---

## 七、总结

### 7.1 账户过滤行为（核心校正）

| 查询函数 | 过滤停用/待删除 | 代码实现 | 对回答的影响 |
|---------|---------------|---------|-------------|
| GetAccounts | ❌ 不过滤 | `family.accounts` | 返回所有账户（含已停用） |
| GetTransactions | ✅ 过滤 | `Transaction::Search` (active_accounts_only: true) | 无法查询停用账户的历史交易 |
| GetBalanceSheet | ✅ 过滤 | `family.accounts.visible` | 净值不包含停用账户 |
| GetIncomeStatement | ✅ 过滤 | `family.transactions.visible` | 收支统计不包含停用账户 |

### 7.2 分页真实限制（核心校正）

| 限制项 | 真实值 | 说明 |
|--------|--------|------|
| 每页大小 | **固定 50 条** | Schema 声明 `page_size` 为 required，但代码硬编码 `default_page_size` |
| 页码 | 从 1 开始 | 使用 `params["page"] \|\| 1` |
| Schema 一致性 | ❌ 不一致 | AI 无法通过参数调整每页大小 |

### 7.3 权限校验要点

1. **入口层**：Session Cookie 认证，未登录则重定向
2. **资源层**：Chat 必须属于 Current.user
3. **数据层**：所有查询以 Family 为作用域边界
4. **审计层**：模拟操作全量日志记录

### 7.4 安全边界

| 边界类型 | 限制 |
|---------|------|
| 数据隔离 | Family 级别的严格隔离 |
| 时间范围 | 历史数据最多 5 年 |
| 返回数量 | 交易分页固定 50 条/页 |
| 角色权限 | super_admin 才允许模拟 |
| 函数调用 | 仅 4 个预定义函数，禁止递归 |

### 7.5 代码定位速查

| 组件 | 文件路径 |
|------|---------|
| 会话认证 | `app/controllers/concerns/authentication.rb` |
| Chat 控制器 | `app/controllers/chats_controller.rb` |
| 助手主逻辑 | `app/models/assistant.rb` |
| 函数调用器 | `app/models/assistant/function_tool_caller.rb` |
| 工具函数基类 | `app/models/assistant/function.rb` |
| GetAccounts | `app/models/assistant/function/get_accounts.rb` |
| GetTransactions | `app/models/assistant/function/get_transactions.rb` |
| GetBalanceSheet | `app/models/assistant/function/get_balance_sheet.rb` |
| GetIncomeStatement | `app/models/assistant/function/get_income_statement.rb` |
| Transaction Search | `app/models/transaction/search.rb` |
| BalanceSheet AccountTotals | `app/models/balance_sheet/account_totals.rb` |
| IncomeStatement | `app/models/income_statement.rb` |
| Entry visible scope | `app/models/entry.rb` |
| Account visible scope | `app/models/account.rb` |
| 当前上下文 | `app/models/current.rb` |
| 用户模型 | `app/models/user.rb` |
| 家族模型 | `app/models/family.rb` |
| 模拟功能 | `app/models/impersonation_session.rb` |

---

## 附录：不一致问题列表

### A. 功能不一致

1. **GetAccounts 与其他函数过滤行为不一致**
   - GetAccounts 返回所有账户（含 disabled/pending_deletion）
   - 其他函数仅返回 visible 账户
   - **影响**：回答口径可能出现矛盾

2. **GetTransactions 的 page_size 参数被忽略**
   - Schema 声明为 required
   - 代码硬编码为 50
   - **影响**：AI 无法调整每页大小，可能遗漏数据

3. **family_account_names 与 GetAccounts 不一致**
   - `family_account_names` 使用 `family.accounts.visible`
   - GetAccounts 使用 `family.accounts`
   - **影响**：AI 无法通过参数筛选已停用账户，即使 GetAccounts 能看到

### B. 潜在改进方向

1. 统一各函数的账户过滤策略（或在 GetAccounts 中也使用 visible）
2. 移除 Schema 中无效的 `page_size` 参数，或在代码中真正支持它
3. 考虑为 AI 提供明确的"是否包含停用账户"参数选项
4. 在系统指令中明确各函数的数据范围差异，帮助 AI 理解并向用户说明

