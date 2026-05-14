# AI 账户查询机制与权限边界

## 概述

Maybe Finance 的 AI 助手通过工具函数（Function Calling）机制，为用户提供账户数据查询能力。本文档详细说明 AI 助手可调用的查询维度、权限校验机制以及安全边界。

---

## 一、查询能力维度

AI 助手可调用 4 个核心工具函数，每个函数提供不同的查询维度：

### 1. GetAccounts - 账户信息查询

**功能**：获取用户所有账户的基本信息和历史余额

**查询维度**：
- 账户名称
- 当前余额（原始值 + 格式化显示）
- 货币类型
- 账户分类（资产/负债）
- 账户类型（Depository、CreditCard、Investment、Crypto、Loan、Property 等）
- 账户开户日期
- 是否通过 Plaid 自动同步
- 账户状态（active、draft、disabled、pending_deletion）
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
    }
  ]
}
```

**代码位置**：`app/models/assistant/function/get_accounts.rb`

---

### 2. GetTransactions - 交易搜索与筛选

**功能**：多维度搜索用户交易记录，支持分页

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
| 分页 | page + page_size（默认 50 条/页） |

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

**代码位置**：`app/models/assistant/function/get_transactions.rb`

---

### 3. GetBalanceSheet - 资产负债表与净值分析

**功能**：获取用户资产负债概况和历史趋势

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

**代码位置**：`app/models/assistant/function/get_balance_sheet.rb`

---

### 4. GetIncomeStatement - 收入支出分析

**功能**：按时间段分析收入支出构成

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

**代码位置**：`app/models/assistant/function/get_income_statement.rb`

---

## 二、权限校验机制

### 2.1 整体架构

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
数据查询
```

---

### 2.2 第一层：会话认证

**认证方式**：基于 Cookie 的会话令牌

**关键代码**：`app/controllers/concerns/authentication.rb:18-28`

```ruby
def authenticate_user!
  if session_record = find_session_by_cookie
    Current.session = session_record
  else
    # 重定向到登录页
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

### 2.3 第二层：Chat 归属校验

**机制**：Chat 必须属于当前登录用户

**关键代码**：
- `app/controllers/chats_controller.rb:8`：`@chats = Current.user.chats`
- `app/controllers/chats_controller.rb:51`：`@chat = Current.user.chats.find(params[:id])`

**校验逻辑**：
1. 查询时使用 `Current.user.chats` 而不是 `Chat` 直接查询
2. 这确保只能访问当前用户拥有的聊天记录
3. 即使知道其他用户的 Chat ID，也无法访问（会抛出 ActiveRecord::RecordNotFound）

---

### 2.4 第三层：工具函数作用域隔离

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

### 2.5 数据模型层次结构

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

## 三、安全边界与限制

### 3.1 访问控制边界

**不能访问的数据**：
- 其他 Family 的任何数据
- 其他用户的 Chat 记录（即使同 Family）
- 已标记为 `disabled` 或 `pending_deletion` 的账户（通过 `visible` scope 过滤）

**代码位置**：
- `app/models/account.rb:21`：`scope :visible, -> { where(status: [ "draft", "active" ]) }`
- `app/models/assistant/function/get_balance_sheet.rb:47`：`scope = family.accounts.visible`

---

### 3.2 数据返回限制

**返回格式约束**：
- 所有货币值以原始数值 + 格式化字符串形式返回
- 历史数据限制：最多返回过去 5 年的数据
- 交易分页限制：默认每页 50 条，强制 AI 使用筛选条件节省 Token

**敏感数据保护**：
- 不返回完整的账户号码、凭证等敏感信息
- 不返回用户密码、API keys 等认证信息
- 不返回 Plaid access tokens 等第三方凭证

---

### 3.3 用户角色与权限

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

### 3.4 模拟功能（Impersonation）

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

### 3.5 函数调用安全

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

## 四、系统指令中的安全约束

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

## 五、总结

### 权限校验要点

1. **入口层**：Session Cookie 认证，未登录则重定向
2. **资源层**：Chat 必须属于 Current.user
3. **数据层**：所有查询以 Family 为作用域边界
4. **审计层**：模拟操作全量日志记录

### 安全边界

| 边界类型 | 限制 |
|---------|------|
| 数据隔离 | Family 级别的严格隔离 |
| 时间范围 | 历史数据最多 5 年 |
| 返回数量 | 交易分页 50 条/页 |
| 角色权限 | super_admin 才允许模拟 |
| 函数调用 | 仅 4 个预定义函数，禁止递归 |

### 代码定位速查

| 组件 | 文件路径 |
|------|---------|
| 会话认证 | `app/controllers/concerns/authentication.rb` |
| Chat 控制器 | `app/controllers/chats_controller.rb` |
| 助手主逻辑 | `app/models/assistant.rb` |
| 函数调用器 | `app/models/assistant/function_tool_caller.rb` |
| 工具函数基类 | `app/models/assistant/function.rb` |
| 工具函数目录 | `app/models/assistant/function/*.rb` |
| 当前上下文 | `app/models/current.rb` |
| 用户模型 | `app/models/user.rb` |
| 家族模型 | `app/models/family.rb` |
| 模拟功能 | `app/models/impersonation_session.rb` |
