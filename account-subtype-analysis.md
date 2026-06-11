# 账户子类型（subtype）对新建账户表单的级联影响分析

> 梳理范围：手动创建路径、Property 多步向导路径、Plaid 导入路径
> 核心问题：subtype 如何影响默认值、可见字段、必填约束；区分表单约束 vs 模型校验

---

## 一、架构基础

### 1.1 三层模型结构

本项目采用 Rails **Delegated Type**（委托多态）模式：

| 层级 | 实现位置 | 职责 |
|------|---------|------|
| 通用账户壳 | [account.rb](file:///d:/fz/0601-1/solo-dogfeeding/code/5-maybe/app/models/account.rb) | 通用属性(name/balance/currency/subtype/classification)、状态机(AASM)、持有 accountable |
| Accountable 协议 | [accountable.rb](file:///d:/fz/0601-1/solo-dogfeeding/code/5-maybe/app/models/concerns/accountable.rb) | 定义 `TYPES` 列表、`SUBTYPES` 协议、`classification/icon/color` 抽象方法 |
| 具体 Accountable | Depository / Investment / CreditCard / Loan / Property / Vehicle / Crypto / OtherAsset / OtherLiability | 各自声明 `SUBTYPES` 常量 + 特有字段 |

关键代码 [account.rb#L29](file:///d:/fz/0601-1/solo-dogfeeding/code/5-maybe/app/models/account.rb#L29-L29)：
```ruby
delegated_type :accountable, types: Accountable::TYPES, dependent: :destroy
```

### 1.2 SUBTYPES 的定义与数据结构

`subtype` 字段位于 `accounts` 表（自由字符串，[schema.rb#L23](file:///d:/fz/0601-1/solo-dogfeeding/code/5-maybe/db/schema.rb#L23-L23)），值空间由各 Accountable 的 `SUBTYPES` 常量约束。

数据结构：
```ruby
SUBTYPES = {
  "key" => { short: "短标签", long: "长标签" }
}.freeze
```

各类型的 SUBTYPES 声明情况：

| Accountable | classification | SUBTYPES keys | 是否有 SUBTYPES |
|------------|----------------|---------------|----------------|
| **Depository** | asset | checking, savings, hsa, cd, money_market | ✅ 5 个 |
| **Investment** | asset | brokerage, pension, retirement, 401k, roth_401k, 529_plan, hsa, mutual_fund, ira, roth_ira, angel | ✅ 11 个 |
| **CreditCard** | liability | credit_card | ✅ 1 个 |
| **Loan** | liability | mortgage, student, auto, other | ✅ 4 个 |
| **Property** | asset | single_family_home, multi_family_home, condominium, townhouse, investment_property, second_home | ✅ 6 个 |
| **Vehicle** | asset | *(无)* | ❌ |
| **Crypto** | asset | *(无)* | ❌ |
| **OtherAsset** | asset | *(无)* | ❌ |
| **OtherLiability** | liability | *(无)* | ❌ |

---

## 二、表单约束 vs 模型校验的边界

这是理解整个系统的关键——两套约束机制互相独立。

### 2.1 表单约束（View 层，required 属性）

**实现位置**：
- 通用表单字段：[styled_form_builder.rb](file:///d:/fz/0601-1/solo-dogfeeding/code/5-maybe/app/helpers/styled_form_builder.rb)
- 金额字段：[shared/_money_field.html.erb](file:///d:/fz/0601-1/solo-dogfeeding/code/5-maybe/app/views/shared/_money_field.html.erb)

**工作机制**：
1. `styled_form_builder` 的 `required` 选项做两件事：
   - 在 label 文本后追加 `<span class="text-red-500">*</span>` 红星视觉提示（[styled_form_builder.rb#L117-L122](file:///d:/fz/0601-1/solo-dogfeeding/code/5-maybe/app/helpers/styled_form_builder.rb#L117-L122)）
   - 将 `required` 作为 HTML 属性透传给 `<input>` 元素（通过 `super` 传递）
2. `_money_field.html.erb` 中 `required: options[:required]` 显式写入 HTML 属性

**效果**：这是纯浏览器端的 HTML5 原生校验（不通过浏览器的 "required" 属性无法提交）。**没有任何 JavaScript 级联逻辑**。

### 2.2 模型校验（Model 层，validates）

**Account 基类的校验**（[account.rb#L4](file:///d:/fz/0601-1/solo-dogfeeding/code/5-maybe/app/models/account.rb#L4-L4)）：
```ruby
validates :name, :balance, :currency, presence: true
```

**关键发现**：所有 9 个 Accountable 子类（Depository/Investment/...）**完全没有任何 `validates` 声明**。包括：
- Loan 的 `initial_balance` / `interest_rate` / `term_months` ——**无模型校验**
- CreditCard 的 `available_credit` / `apr` ——**无模型校验**
- Property 的 `year_built` / `area_value` ——**无模型校验**
- Address 的 `line1` / `locality` / `country` ——**无模型校验**（[address.rb](file:///d:/fz/0601-1/solo-dogfeeding/code/5-maybe/app/models/address.rb)）
- **subtype 字段本身 —— 无任何 inclusion/presence 校验**

### 2.3 两种约束的错位

| 字段 | 表单 required | 模型 validates presence | 是否一致 |
|------|:---:|:---:|:---:|
| name | ✅ | ✅ | ✅ 一致 |
| balance（非link账户） | ✅ | ✅ | ✅ 一致 |
| currency | ❌（隐含默认值） | ✅ | ⚠️ 表单靠默认值兜底 |
| Loan#initial_balance | ✅ | ❌ | ❌ 不一致（用户绕过浏览器即可写入空值） |
| Property#subtype | ✅（向导里） | ❌ | ❌ 不一致 |
| Property#name | ✅（向导里） | ✅ | ✅ 一致 |
| Loan#interest_rate / term_months | ❌ | ❌ | — （都不约束） |
| 所有其他 Accountable 字段 | ❌ | ❌ | — （都不约束） |

**风险点**：Loan 的 `initial_balance` 在表单上标记了 `required: true`，但模型层无校验。若用户绕过浏览器 HTML5 校验（如 Postman/curl 直接 POST），可创建 `initial_balance = nil` 的 Loan，而后续 `monthly_payment` 计算依赖这些字段可能出错。

---

## 三、路径一：手动创建（AccountableResource 标准路径）

### 3.1 入口与路由

用户访问 `/accounts/new` → [accounts/new.html.erb](file:///d:/fz/0601-1/solo-dogfeeding/code/5-maybe/app/views/accounts/new.html.erb)

第一步是**分类过滤**（硬编码分支，非配置驱动）：
```erb
<% unless params[:classification] == "liability" %>
  <%= render "account_type", accountable: Depository.new %>
  <%= render "account_type", accountable: Investment.new %>
  ...
<% end %>

<% unless params[:classification] == "asset" %>
  <%= render "account_type", accountable: CreditCard.new %>
  <%= render "account_type", accountable: Loan.new %>
  ...
<% end %>
```

`classification` 参数不存储到数据库，仅用于展示过滤。

### 3.2 进入类型专属表单

点击某类型后，路由到该类型的专属 controller（如 `DepositoriesController`），所有 8 个标准类型 controller 均 include [AccountableResource](file:///d:/fz/0601-1/solo-dogfeeding/code/5-maybe/app/controllers/concerns/accountable_resource.rb)（Property 除外，见第四节）。

`new` action（[accountable_resource.rb#L18-L23](file:///d:/fz/0601-1/solo-dogfeeding/code/5-maybe/app/controllers/concerns/accountable_resource.rb#L18-L23)）：
```ruby
def new
  @account = Current.family.accounts.build(
    currency: Current.family.currency,    # 默认填充：家庭币种
    accountable: accountable_type.new      # 默认填充：空壳 accountable
  )
end
```

**默认填充只有 2 项**：
- `currency` ← `Current.family.currency`（配置驱动）
- `accountable_type` ← controller 名称推断（`controller_name.classify.constantize`）

**subtype 无默认值**，始终为 `nil`。

### 3.3 表单渲染与 subtype 可见性

每个类型有独立的 `_form.html.erb`，通过 `yield` 包裹公共表单 [accounts/_form.html.erb](file:///d:/fz/0601-1/solo-dogfeeding/code/5-maybe/app/views/accounts/_form.html.erb)。

公共表单内容（固定 3 项）：
| 字段 | required |
|------|:--------:|
| `name` | ✅ |
| `balance`（非 link 账户） | ✅ |
| `accountable_type`（hidden） | — |

各类型专属差异：

#### Depository（[depositories/_form.html.erb](file:///d:/fz/0601-1/solo-dogfeeding/code/5-maybe/app/views/depositories/_form.html.erb)）
```erb
<%= form.select :subtype,
               Depository::SUBTYPES.map { |k, v| [v[:long], k] },
               { label: true, prompt: "...", include_blank: "None" } %>
```
- subtype：下拉框，5 选项，**非 required**，允许 blank

#### Investment（[investments/_form.html.erb](file:///d:/fz/0601-1/solo-dogfeeding/code/5-maybe/app/views/investments/_form.html.erb)）
- subtype：下拉框，11 选项，**非 required**，允许 blank

#### CreditCard（[credit_cards/_form.html.erb](file:///d:/fz/0601-1/solo-dogfeeding/code/5-maybe/app/views/credit_cards/_form.html.erb)）
- **无 subtype 字段**（虽定义了 SUBTYPES = { "credit_card" => ... }，但表单不渲染选择框）
- 额外字段：`available_credit`、`minimum_payment`、`apr`、`expiration_date`、`annual_fee`
- 所有额外字段**均非 required**

#### Loan（[loans/_form.html.erb](file:///d:/fz/0601-1/solo-dogfeeding/code/5-maybe/app/views/loans/_form.html.erb)）
- **无 subtype 字段**（虽定义了 SUBTYPES，但表单不渲染选择框）
- 额外字段：`initial_balance`（✅ required）、`interest_rate`、`rate_type`、`term_months`
- `initial_balance` 是唯一的 required 专属字段

#### Vehicle（[vehicles/_form.html.erb](file:///d:/fz/0601-1/solo-dogfeeding/code/5-maybe/app/views/vehicles/_form.html.erb)）
- **无 subtype**
- 额外字段：`make`、`model`、`year`、`mileage_value`、`mileage_unit`（全部非 required）

#### Crypto / OtherAsset / OtherLiability
- **无 subtype**
- 无额外字段，直接渲染公共表单

### 3.4 subtype 在手动路径中的作用总结

**选择 subtype 不会触发任何运行时行为变化**：
- ❌ 不会动态显示/隐藏其他字段（前端无 JS 监听）
- ❌ 不会改变其他字段的 required 状态
- ❌ 不会改变默认值
- ❌ 不会改变提交 URL
- ✅ 仅作为一个静态枚举值存入 `accounts.subtype` 列

subtype 的消费点只有两处展示层：
- 账户列表的副标题（[_account.html.erb#L20-L22](file:///d:/fz/0601-1/solo-dogfeeding/code/5-maybe/app/views/accounts/_account.html.erb#L20-L22)）：调用 `account.long_subtype_label`
- 账户卡片标签（[_accountable_group.html.erb#L38](file:///d:/fz/0601-1/solo-dogfeeding/code/5-maybe/app/views/accounts/_accountable_group.html.erb#L38-L38)）：调用 `account.short_subtype_label`

这两个方法通过 [accountable.rb#L34-L49](file:///d:/fz/0601-1/solo-dogfeeding/code/5-maybe/app/models/concerns/accountable.rb#L34-L49) 查表：
```ruby
def subtype_label_for(subtype, format: :short)
  label_type = format == :long ? :long : :short
  self::SUBTYPES[subtype]&.fetch(label_type, nil)
end
```

### 3.5 提交与参数白名单

`create` action（[accountable_resource.rb#L36-L41](file:///d:/fz/0601-1/solo-dogfeeding/code/5-maybe/app/controllers/concerns/accountable_resource.rb#L36-L41)）：
```ruby
def create
  @account = Current.family.accounts.create_and_sync(account_params.except(:return_to))
  @account.lock_saved_attributes!
  redirect_to ...
end
```

`account_params` 白名单（[accountable_resource.rb#L81-L86](file:///d:/fz/0601-1/solo-dogfeeding/code/5-maybe/app/controllers/concerns/accountable_resource.rb#L81-L86)）：
```ruby
def account_params
  params.require(:account).permit(
    :name, :balance, :subtype, :currency, :accountable_type, :return_to,
    accountable_attributes: self.class.permitted_accountable_attributes
  )
end
```

各类型通过类方法 `permitted_accountable_attributes` 声明专属字段白名单（**配置驱动**）：
- CreditCardsController：`[:id, :available_credit, :minimum_payment, :apr, :annual_fee, :expiration_date]`
- LoansController：`[:id, :rate_type, :interest_rate, :term_months, :initial_balance]`
- VehiclesController：`[:id, :make, :model, :year, :mileage_value, :mileage_unit]`
- 其余：默认 `[:id]`

---

## 四、路径二：Property 多步向导（独立流程）

Property **不使用** AccountableResource，而是在 [properties_controller.rb](file:///d:/fz/0601-1/solo-dogfeeding/code/5-maybe/app/controllers/properties_controller.rb) 中实现独立的 4 步向导：

### 4.1 四步流程

| 步骤 | Action | View | 内容 |
|------|--------|------|------|
| Step 1 | `new` / `create` | [properties/new.html.erb](file:///d:/fz/0601-1/solo-dogfeeding/code/5-maybe/app/views/properties/new.html.erb) + [_overview_fields.html.erb](file:///d:/fz/0601-1/solo-dogfeeding/code/5-maybe/app/views/properties/_overview_fields.html.erb) | 输入 name / subtype / year_built / area |
| Step 2 | `balances` | [properties/balances.html.erb](file:///d:/fz/0601-1/solo-dogfeeding/code/5-maybe/app/views/properties/balances.html.erb) | 输入当前市价 balance |
| Step 3 | `address` | [properties/address.html.erb](file:///d:/fz/0601-1/solo-dogfeeding/code/5-maybe/app/views/properties/address.html.erb) | 输入地址（嵌套属性） |
| Step 4 | `update_address` 中 `activate!` | — | 状态从 draft → active |

### 4.2 Step 1 中 subtype 的特殊地位

在 [_overview_fields.html.erb](file:///d:/fz/0601-1/solo-dogfeeding/code/5-maybe/app/views/properties/_overview_fields.html.erb)：

```erb
<%= form.select :subtype,
              Property::SUBTYPES.map { |k, v| [v[:long], k] },
              { prompt: "Select type", label: "Property type" },
              required: true %>   <%# 注意：这里是 HTML 属性 required %>
```

**Property 是唯一将 subtype 标记为表单 required 的类型**。但再次强调：模型层无校验。

创建逻辑（[properties_controller.rb#L10-L16](file:///d:/fz/0601-1/solo-dogfeeding/code/5-maybe/app/controllers/properties_controller.rb#L10-L16)）：
```ruby
def create
  @account = Current.family.accounts.create!(
    property_params.merge(currency: Current.family.currency, balance: 0, status: "draft")
  )
  redirect_to balances_property_path(@account)
end
```

注意：
- `create!`（非 `create_and_sync`）跳过了 OpeningBalanceManager
- `balance: 0` 硬编码（绕过了 Account 的 `validates :balance, presence: true`—— presence 不检查数值是否为 0）
- `status: "draft"`——账户尚未激活

### 4.3 subtype 在向导中的级联影响

**无任何级联**：
- subtype 在 Step 1 选中后
- Step 2（balances）和 Step 3（address）的字段集完全固定
- 没有根据 subtype（如 condominium vs investment_property）调整任何字段的出现/必填/默认值

### 4.4 Property 的专属默认值

Property 模型有两个独立的 `attribute` 默认值声明：
- [property.rb#L17](file:///d:/fz/0601-1/solo-dogfeeding/code/5-maybe/app/models/property.rb#L17-L17)：`attribute :area_unit, :string, default: "sqft"`
- [vehicle.rb#L4](file:///d:/fz/0601-1/solo-dogfeeding/code/5-maybe/app/models/vehicle.rb#L4-L4)（同类）：`attribute :mileage_unit, :string, default: "mi"`

这是**模型级的配置驱动**，与 subtype 无关。

---

## 五、路径三：Plaid 导入

Plaid 导入的 subtype 处理发生在服务端处理流程中，用户不直接操作。

### 5.1 映射配置（纯配置驱动）

核心配置表在 [type_mappable.rb#L29-L76](file:///d:/fz/0601-1/solo-dogfeeding/code/5-maybe/app/models/plaid_account/type_mappable.rb#L29-L76) 的 `TYPE_MAPPING` 常量：

```ruby
TYPE_MAPPING = {
  depository: {
    accountable: Depository,
    subtype_mapping: {
      "checking"      => "checking",
      "savings"       => "savings",
      "hsa"           => "hsa",
      "cd"            => "cd",
      "money market"  => "money_market"
    }
  },
  credit: {
    accountable: CreditCard,
    subtype_mapping: { "credit card" => "credit_card" }
  },
  loan: {
    accountable: Loan,
    subtype_mapping: {
      "mortgage"       => "mortgage",
      "student"        => "student",
      "auto"           => "auto",
      "business"       => "business",     # ⚠️ 不一致：Loan::SUBTYPES 中无此 key
      "home equity"    => "home_equity",  # ⚠️ 不一致
      "line of credit" => "line_of_credit" # ⚠️ 不一致
    }
  },
  investment: {
    accountable: Investment,
    subtype_mapping: {
      "brokerage"    => "brokerage",
      "pension"      => "pension",
      "retirement"   => "retirement",
      "401k"         => "401k",
      "roth 401k"    => "roth_401k",
      "529"          => "529_plan",
      "hsa"          => "hsa",
      "mutual fund"  => "mutual_fund",
      "roth"         => "roth_ira",
      "ira"          => "ira"
    }
  },
  other: { accountable: OtherAsset, subtype_mapping: {} }
}
```

映射逻辑在 [type_mappable.rb#L19-L25](file:///d:/fz/0601-1/solo-dogfeeding/code/5-maybe/app/models/plaid_account/type_mappable.rb#L19-L25)：
```ruby
def map_subtype(plaid_type, plaid_subtype)
  TYPE_MAPPING.dig(plaid_type.to_sym, :subtype_mapping, plaid_subtype) || "other"
end
```

当映射不存在时，fallback 为 `"other"`。

### 5.2 写入 Account

在 [PlaidAccount::Processor#process_account!](file:///d:/fz/0601-1/solo-dogfeeding/code/5-maybe/app/models/plaid_account/processor.rb#L31-L62)：

```ruby
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
```

关键点：
1. **name + subtype** 通过 `enrich_attributes` 写入（因为用户可以覆盖，受 [Enrichable](file:///d:/fz/0601-1/solo-dogfeeding/code/5-maybe/app/models/concerns/enrichable.rb) 的 lock 机制保护）
2. **accountable/balance/currency/cash_balance** 直接 `assign_attributes`（用户不能覆盖）
3. `enrich_attributes` 会跳过 `locked?` 的属性——用户一旦手动改过 name 或 subtype，Plaid 同步就不会再覆盖

### 5.3 subtype 对 Liabilities 处理的分支影响

**这是整个系统中 subtype 真正参与运行时分支判断的唯一位置**（[processor.rb#L77-L88](file:///d:/fz/0601-1/solo-dogfeeding/code/5-maybe/app/models/plaid_account/processor.rb#L77-L88)）：

```ruby
def process_liabilities
  case [ plaid_account.plaid_type, plaid_account.plaid_subtype ]
  when [ "credit", "credit card" ]
    PlaidAccount::Liabilities::CreditProcessor.new(plaid_account).process
  when [ "loan", "mortgage" ]
    PlaidAccount::Liabilities::MortgageProcessor.new(plaid_account).process
  when [ "loan", "student" ]
    PlaidAccount::Liabilities::StudentLoanProcessor.new(plaid_account).process
  end
end
```

**注意**：这里分支判断使用的是 `plaid_account.plaid_subtype`（Plaid 原始值），不是 `account.subtype`（映射后的本地值）。影响如下：

| [plaid_type, plaid_subtype] | 触发的 Processor | 写入的字段 |
|----------------------------|-----------------|-----------|
| `["credit", "credit card"]` | CreditProcessor | `credit_card.minimum_payment`、`credit_card.apr` |
| `["loan", "mortgage"]` | MortgageProcessor | `loan.rate_type`、`loan.interest_rate` |
| `["loan", "student"]` | StudentLoanProcessor | `loan.rate_type="fixed"`、`loan.interest_rate`、`loan.initial_balance`、`loan.term_months` |
| `["loan", "auto"]` 及其他所有组合 | **不触发任何 Processor** | — |

**级联影响总结（Plaid 路径内）**：
- ✅ subtype 决定了是否写入 CreditCard / Loan 的专属字段
- ✅ subtype 决定了 StudentLoan 是否自动计算 term_months
- ❌ 不影响 accountable_type 的选择（那个由 plaid_type 决定）
- ❌ 不影响 balance 的计算方式（由 plaid_type == "investment" 决定）

### 5.4 映射不一致的问题

Plaid loan subtype 有 6 个值，但 Loan::SUBTYPES 只定义了 4 个 key：

| Plaid loan subtype | map_subtype 结果 | Loan::SUBTYPES 中存在？ | 展示标签 fallback |
|-------------------|-----------------|:---:|---|
| mortgage | "mortgage" | ✅ | "Mortgage" |
| student | "student" | ✅ | "Student Loan" |
| auto | "auto" | ✅ | "Auto Loan" |
| **business** | "business" | ❌ | display_name（"Loans"） |
| **home_equity** | "home_equity" | ❌ | display_name（"Loans"） |
| **line_of_credit** | "line_of_credit" | ❌ | display_name（"Loans"） |
| (其他未知) | "other" | ✅ | "Other Loan" |

同时，`process_liabilities` 的 case 分支也只处理了 mortgage / student，auto / business / home_equity / line_of_credit 都不会触发专属字段填充。

---

## 六、驱动模式判定总表

| 场景 | 实现方式 | 驱动模式 |
|------|---------|---------|
| 手动入口的 classification 过滤 | `unless params[:classification] == "liability"` 硬编码 | **分支判断** |
| 类型 → 专属 controller | Rails 路由 + controller 名称推断（`controller_name.classify`） | **约定优于配置** |
| 类型 → 专属表单 partial | 每个类型目录下独立 `_form.html.erb` | **静态分派**（编译期确定） |
| SUBTYPES 选项列表 | Model 中 `SUBTYPES = {...}.freeze` 常量 | **配置驱动** |
| subtype 下拉选项渲染 | `Xxx::SUBTYPES.map { ... }` 查表 | **配置驱动** |
| 表单字段白名单 | `permitted_accountable_attributes` 类方法声明 | **配置驱动** |
| Property area_unit 默认值 | `attribute :area_unit, default: "sqft"` | **配置驱动** |
| Property 向导步骤路由 | `balances` / `address` 独立 action | **分支判断** |
| Plaid type → accountable | `TYPE_MAPPING.dig(:accountable)` 查表 | **配置驱动** |
| Plaid subtype → 本地 subtype | `TYPE_MAPPING.dig(:subtype_mapping, x) \|\| "other"` | **配置驱动 + fallback 分支** |
| Plaid liabilities 处理器选择 | `case [plaid_type, plaid_subtype]` | **分支判断**（且用的是原始 plaid_subtype 非映射值） |
| Account balance_type 计算 | `case accountable_type`（[account.rb#L151-L162](file:///d:/fz/0601-1/solo-dogfeeding/code/5-maybe/app/models/account.rb#L151-L162)） | **分支判断** |
| AccountPage tabs 选择 | `case account.accountable_type` | **分支判断** |

---

## 七、结论

### 7.1 subtype 对新建账户表单的级联影响（三条路径横向对比）

| 影响维度 | 手动创建（标准） | Property 向导 | Plaid 导入 |
|---------|:---:|:---:|:---:|
| **默认值** | subtype 始终为 nil；无任何根据 subtype 设定的字段默认值 | subtype 无默认值；area_unit 默认 "sqft" 与 subtype 无关 | subtype = `map_subtype` 查表结果；**根据 subtype 触发不同 Processor 写入专属字段默认值**（仅 liabilities 场景） |
| **可见字段** | 字段集由 accountable_type 静态决定，与 subtype 完全无关 | 三步向导的字段集完全固定，与 subtype 无关 | （非表单场景，不适用） |
| **必填约束** | 只有 Property 在 Step 1 表单层标记 subtype required；**模型层无任何 subtype 校验** | Step 1 中 subtype 和 name 为表单 required；其余字段全部非 required；模型层除 name/balance/currency 外无校验 | （非表单场景，不适用） |
| **前端级联 JS** | 无 | 无 | （非表单场景） |
| **后端分支判断** | 无（提交后直接存入字段） | 无（提交后直接存入字段） | **有**：`process_liabilities` 中按 subtype 选 Processor |

### 7.2 核心判断

1. **"subtype 驱动表单"是一个错觉**：在两条手动路径中，subtype 只是标签字段，不驱动任何表单行为变化。唯一真正用 subtype 做分支判断的场景是 Plaid 导入路径中的 Liabilities 处理（且判断对象是 `plaid_subtype` 原始值，不是映射后的 `account.subtype`）。

2. **真正驱动表单差异的是 accountable_type**：通过"每个类型一个独立 View 目录"实现静态分派，而不是通过运行时配置查表渲染。如果新增 Accountable 类型，需要新建 controller、view 目录、form partial，而不是改配置表。

3. **表单 required ≠ 模型 presence 校验**：至少 3 处不一致（Loan#initial_balance、Property#subtype、Property 向导中的所有 required 标记）。用户一旦绕过 HTML5 原生校验，脏数据可直接入库。

4. **驱动模式是"配置 + 分支 + 静态分派"三者的混合**，其中：
   - 枚举值列表（SUBTYPES、TYPE_MAPPING）是**配置驱动**
   - 视图渲染、路由分派是**静态分派**（Rails 约定）
   - 入口过滤、Liabilities Processor 选择、balance_type 计算是**分支判断**

5. **Plaid 映射存在 3 个漏洞**：`business` / `home_equity` / `line_of_credit` 三个 loan subtype 在映射后不在 Loan::SUBTYPES 内，也不触发任何 Liability Processor，展示标签会退化到通用的 display_name。
