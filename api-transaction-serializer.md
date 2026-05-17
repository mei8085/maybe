# API v1 交易类响应序列化方式纵深调研报告

## 1. 概述

本报告针对对外开放的 v1 接口中交易类（Transaction）响应的序列化方式进行深入分析，涵盖模型层数据映射、字段口径差异、分页与过滤机制，以及控制器、序列化器、家庭隔离三者的协作机制。

## 2. 模型层数据映射到对外字段

### 2.1 数据模型结构

交易数据采用**双层多态关联**设计：

```
Family (家庭)
  └── Account (账户)
       └── Entry (条目)
            ├── amount (金额，整数分)
            ├── date (日期)
            ├── name (名称)
            ├── currency (币种)
            ├── notes (备注)
            └── entryable (多态关联)
                 └── Transaction (交易)
                      ├── category_id (分类ID)
                      ├── merchant_id (商户ID)
                      ├── kind (类型: standard/funds_movement/cc_payment/loan_payment/one_time)
                      ├── tags (标签，多对多)
                      └── transfer (转账关联)
```

**核心关联文件**：
- 模型定义：`app/models/transaction.rb:1-34`
- Entry模型：`app/models/entry.rb:1-99`
- 多态类型：`app/models/entryable.rb:1-31`

### 2.2 字段映射关系

API 响应字段在 `app/views/api/v1/transactions/_transaction.json.jbuilder:1-76` 中定义，映射关系如下：

| API 字段 | 来源 | 说明 |
|---------|------|------|
| `id` | `transaction.id` | 交易ID |
| `date` | `transaction.entry.date` | 交易日期 |
| `amount` | `transaction.entry.amount_money.format` | **格式化金额字符串**（如 "$10.00"、"-$10.00"），包含货币符号和正负号，非原始数值 |
| `currency` | `transaction.entry.currency` | 币种代码（如 "USD"） |
| `name` | `transaction.entry.name` | 交易名称 |
| `notes` | `transaction.entry.notes` | 备注 |
| `classification` | `transaction.entry.classification` | 交易类型：`"income"`（收入）或 `"expense"`（支出），根据数据库存储的金额正负判断 |
| `account.id` | `transaction.entry.account.id` | 账户ID |
| `account.name` | `transaction.entry.account.name` | 账户名称 |
| `account.account_type` | `transaction.entry.account.accountable_type.underscore` | 账户类型（如 "depository"） |
| `category` | `transaction.category` | 分类对象（可选） |
| `merchant` | `transaction.merchant` | 商户对象（可选） |
| `tags` | `transaction.tags` | 标签数组 |
| `transfer` | `transaction.transfer` | 转账信息（可选），其中 `transfer.amount` 为绝对金额（无符号） |
| `created_at` | `transaction.created_at.iso8601` | 创建时间（ISO8601格式） |
| `updated_at` | `transaction.updated_at.iso8601` | 更新时间（ISO8601格式） |

> **重要提示**：`amount` 字段是格式化后的字符串，不是数值类型。如需进行数值计算，请调用方自行解析字符串提取数值部分，或通过 `currency` 字段结合自定义逻辑处理。

### 2.3 金额处理机制与符号规则

#### 2.3.1 数据库存储

金额在数据库中以**整数分**存储（`entry.amount` 字段），符号规则为：
- **收入**（income）：存储为**负数**（如 -1000 表示 $10.00 收入）
- **支出**（expense）：存储为**正数**（如 1000 表示 $10.00 支出）

#### 2.3.2 金额转换流程

通过 `Monetizable` concern 将整数分转换为 `Money` 对象：

```ruby
# app/models/concerns/monetizable.rb:1-22
module Monetizable
  def monetize(*fields)
    fields.each do |field|
      define_method("#{field}_money") do
        Money.new(value, monetizable_currency)  # value 为数据库存储的整数分
      end
    end
  end
end
```

#### 2.3.3 API 序列化格式

在 Jbuilder 模板中调用 `.format` 方法生成格式化字符串：

```ruby
# app/views/api/v1/transactions/_transaction.json.jbuilder:5
json.amount transaction.entry.amount_money.format
```

**格式化示例**：
| 数据库存储（整数分） | classification | API 返回字符串 | 含义 |
|---------------------|----------------|--------------|------|
| -1000 | `"income"` | `"-$10.00"` | $10.00 收入 |
| 1000 | `"expense"` | `"$10.00"` | $10.00 支出 |
| -2550 | `"income"` | `"-$25.50"` | $25.50 收入 |
| 2550 | `"expense"` | `"$25.50"` | $25.50 支出 |

#### 2.3.4 classification 判断逻辑

`classification` 字段完全由数据库存储的金额正负决定：

```ruby
# app/models/entry.rb:37-39
def classification
  amount.negative? ? "income" : "expense"
end
```

> **转账特殊说明**：转账交易包含两条记录，流入方（to_account）金额为负（income），流出方（from_account）金额为正（expense）。`transfer.amount` 字段返回的是绝对值（无符号）。

## 3. 字段口径与界面所见数据的差异

### 3.1 主要差异点

| 维度 | API 响应 | Web 界面 | 差异说明 |
|------|---------|---------|---------|
| **金额显示** | `amount` 字段直接返回格式化字符串（如 "$25.50"） | 收入显示为绿色，支出显示为默认色；转账显示为 "+/-" 格式 | 界面根据 `classification` 和 `transfer?` 状态动态调整显示方式 |
| **转账去重** | 转账的两条交易（流入/流出）都会返回 | 转账只显示一条记录（隐藏流入交易） | 界面通过 `EntriesHelper#entries_by_date` 进行去重处理 |
| **交易类型标识** | 未返回 `kind` 字段 | 显示 one-time 标记（橙色星号）、转账/贷款支付标签 | API 序列化器未暴露 `kind` 枚举值 |
| **商户Logo** | 只返回 `merchant.id` 和 `merchant.name` | 显示商户 Logo 或自动生成的首字母图标 | API 不返回 logo_url 字段 |
| **分类展示** | 返回完整 category 对象（含 color、icon） | 显示分类颜色图标和名称 | 序列化完整，无差异 |

### 3.2 界面特殊处理逻辑

```erb
<!-- app/views/transactions/_transaction.html.erb:98-99 -->
<%= content_tag :p,
    transaction.transfer? && view_ctx == "global" ? "+/- #{format_money(entry.amount_money.abs)}" : format_money(-entry.amount_money),
    class: ["text-green-600": entry.amount.negative?] %>
```

**关键差异**：
1. 界面对转账交易使用 `+/-` 前缀并取绝对值
2. 非转账交易对金额取反（因内部存储收入为负、支出为正）
3. 根据金额正负应用绿色样式

## 4. 分页与过滤参数对响应结构的影响

### 4.1 分页机制

**分页参数**（`app/controllers/api/v1/transactions_controller.rb:313-326`）：
- `page`: 页码，默认 1，必须 > 0
- `per_page`: 每页数量，默认 25，范围 1-100

**分页实现**：使用 Pagy 库

```ruby
@pagy, @transactions = pagy(
  transactions_query,
  page: safe_page_param,
  limit: safe_per_page_param
)
```

**响应结构**（`app/views/api/v1/transactions/index.json.jbuilder:1-12`）：
```json
{
  "transactions": [ ... ],
  "pagination": {
    "page": 1,
    "per_page": 25,
    "total_count": 100,
    "total_pages": 4
  }
}
```

### 4.2 过滤参数

支持的过滤参数（`app/controllers/api/v1/transactions_controller.rb:172-240`）：

| 参数 | 类型 | 说明 |
|------|------|------|
| `account_id` | 单个ID | 按账户过滤 |
| `account_ids` | ID数组 | 按多个账户过滤 |
| `category_id` | 单个ID | 按分类过滤 |
| `category_ids` | ID数组 | 按多个分类过滤 |
| `merchant_id` | 单个ID | 按商户过滤 |
| `merchant_ids` | ID数组 | 按多个商户过滤 |
| `start_date` | 日期字符串 | 开始日期 |
| `end_date` | 日期字符串 | 结束日期 |
| `min_amount` | 浮点数 | 最小金额 |
| `max_amount` | 浮点数 | 最大金额 |
| `tag_ids` | ID数组 | 按标签过滤 |
| `type` | 字符串 | "income" 或 "expense" |
| `search` | 字符串 | 搜索名称、备注、商户名 |

**过滤实现示例**：
```ruby
def apply_filters(query)
  query = query.joins(:entry).where(entries: { account_id: params[:account_id] }) if params[:account_id].present?
  query = query.where("entries.amount < 0") if params[:type]&.downcase == "income"
  # ... 其他过滤逻辑
end
```

### 4.3 搜索机制

搜索使用 PostgreSQL ILIKE 进行不区分大小写的模糊匹配：

```ruby
# app/controllers/api/v1/transactions_controller.rb:242-251
def apply_search(query)
  search_term = "%#{params[:search]}%"
  query.joins(:entry)
       .left_joins(:merchant)
       .where(
         "entries.name ILIKE ? OR entries.notes ILIKE ? OR merchants.name ILIKE ?",
         search_term, search_term, search_term
       )
end
```

### 4.4 数据预加载

为避免 N+1 查询问题，控制器在查询时预加载所有必要关联：

```ruby
# app/controllers/api/v1/transactions_controller.rb:22-27
transactions_query = transactions_query.includes(
  { entry: :account },
  :category, :merchant, :tags,
  transfer_as_outflow: { inflow_transaction: { entry: :account } },
  transfer_as_inflow: { outflow_transaction: { entry: :account } }
).reverse_chronological
```

## 5. 控制器、序列化器、家庭隔离三者的协作

### 5.1 协作架构图

```
API 请求
   ↓
BaseController (认证层)
   ├─ authenticate_request! (OAuth/API Key)
   ├─ check_api_key_rate_limit (限流)
   └─ setup_current_context_for_api (设置Current上下文)
   ↓
TransactionsController (业务层)
   ├─ ensure_read_scope / ensure_write_scope (权限检查)
   ├─ current_resource_owner.family (获取家庭上下文)
   ├─ family.transactions.visible (家庭隔离)
   ├─ apply_filters / apply_search (过滤搜索)
   ├─ pagy (分页)
   └─ render :index / :show (渲染)
   ↓
Jbuilder Templates (序列化层)
   ├─ index.json.jbuilder (列表 + 分页信息)
   ├─ show.json.jbuilder (详情)
   └─ _transaction.json.jbuilder (单个交易序列化)
```

### 5.2 控制器职责

**文件**：`app/controllers/api/v1/transactions_controller.rb:1-327`

**核心职责**：
1. **权限控制**：通过 `before_action` 确保 read/write scope
2. **家庭隔离**：所有查询通过 `current_resource_owner.family` 进行
3. **数据查询**：应用过滤、搜索、排序、分页
4. **参数处理**：安全处理分页参数，避免越界
5. **错误处理**：统一捕获异常并返回 JSON 错误响应

**关键方法**：
- `index` - 交易列表查询
- `show` - 交易详情
- `create` - 创建交易
- `update` - 更新交易
- `destroy` - 删除交易
- `apply_filters` - 应用过滤条件
- `apply_search` - 应用搜索
- `calculate_signed_amount` - 根据 nature 参数计算带符号金额

### 5.3 序列化器职责

**文件**：
- 列表：`app/views/api/v1/transactions/index.json.jbuilder:1-12`
- 详情：`app/views/api/v1/transactions/show.json.jbuilder:1-3`
- 局部模板：`app/views/api/v1/transactions/_transaction.json.jbuilder:1-76`

**序列化特点**：
1. 使用 Rails Jbuilder 模板进行 JSON 序列化
2. 列表和详情复用同一个 `_transaction` 局部模板
3. 列表额外包含 `pagination` 元数据
4. 可选字段（category, merchant, transfer）不存在时显式返回 `null`
5. 关联对象按需展开，不返回完整嵌套结构

### 5.4 家庭隔离机制

**家庭隔离是整个 API 安全的核心，通过多层保障实现**：

#### 第一层：认证层（BaseController）
- `app/controllers/api/v1/base_controller.rb:42-104`
- 通过 OAuth token 或 API Key 认证用户
- 设置 `@current_user` 和 `Current.session`

#### 第二层：数据访问层（TransactionsController）
```ruby
# app/controllers/api/v1/transactions_controller.rb:12-13
family = current_resource_owner.family
transactions_query = family.transactions.visible
```

- 所有查询从 `current_resource_owner.family` 出发
- 通过 `has_many :transactions, through: :accounts` 关联自动限制范围
- `visible` scope 进一步过滤掉非活跃账户的交易

#### 第三层：单条记录访问
```ruby
# app/controllers/api/v1/transactions_controller.rb:153-162
def set_transaction
  family = current_resource_owner.family
  @transaction = family.transactions.find(params[:id])
end
```

- 单条记录查询也通过 `family.transactions.find` 进行
- 跨家庭访问会直接抛出 `ActiveRecord::RecordNotFound`，返回 404

#### 第四层：辅助保障
- `app/controllers/api/v1/base_controller.rb:235-245` 提供 `ensure_current_family_access` 通用方法
- 创建/更新操作也通过家庭关联进行：`family.accounts.find(...)`

### 5.5 协作流程示例（查询交易列表）

```
1. 请求 GET /api/v1/transactions?page=1&per_page=10&category_id=5
   └─ 携带 X-Api-Key 或 Authorization 头

2. BaseController 处理
   ├─ authenticate_request! → 验证 API Key，找到 @current_user
   ├─ check_api_key_rate_limit → 检查并更新限流计数
   └─ setup_current_context_for_api → 设置 Current.session

3. TransactionsController#index 处理
   ├─ ensure_read_scope → 检查 read 或 read_write 权限
   ├─ 获取 family = current_resource_owner.family
   ├─ 构建初始查询 family.transactions.visible
   ├─ apply_filters → 应用 category_id=5 过滤
   ├─ includes(...) → 预加载关联避免 N+1
   ├─ pagy → 分页（page=1, per_page=10）
   └─ render :index → 交给 Jbuilder 序列化

4. Jbuilder 序列化
   ├─ 遍历 @transactions，每个调用 _transaction 局部模板
   ├─ 生成 transactions 数组
   └─ 生成 pagination 元数据

5. 返回 JSON 响应
```

## 6. 现存问题与优化建议

### 6.1 现存问题

1. **kind 字段未暴露**：API 未返回交易类型（standard/one_time/funds_movement 等），调用方无法区分一次性交易和常规交易
2. **转账去重不一致**：API 返回转账的两条记录，Web 界面只显示一条，可能造成调用方困惑
3. **金额口径不一致**：API 返回原始带符号金额，Web 界面对非转账交易取反显示
4. **缺少排除字段**：无法过滤掉 excluded 标记的交易
5. **排序不可配置**：固定按日期倒序，无法自定义排序字段

### 6.2 优化建议

1. **暴露 kind 字段**：在 `_transaction.json.jbuilder` 中添加 `json.kind transaction.kind`
2. **添加去重参数**：增加 `dedup_transfers=true/false` 参数，控制是否对转账去重
3. **统一金额口径**：考虑添加 `amount_raw` 字段返回原始数值，或在文档中明确说明金额符号规则
4. **支持 excluded 过滤**：增加 `include_excluded` 参数，默认不返回 excluded 交易
5. **可配置排序**：支持 `sort` 和 `order` 参数，允许按 amount、date 等字段排序

## 7. 总结

API v1 交易类响应序列化采用了清晰的三层架构：
- **控制器**负责认证、权限、家庭隔离、数据查询
- **Jbuilder 模板**负责将模型数据映射为对外字段
- **家庭隔离**通过多层机制确保数据安全，是整个系统的核心保障

字段映射基本满足需求，但与 Web 界面存在一些差异，主要体现在金额显示、转账去重、交易类型标识等方面。建议在后续版本中统一口径，提升 API 的一致性和可用性。
