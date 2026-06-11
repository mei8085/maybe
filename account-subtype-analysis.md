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

---

## 八、补充：Property 控制器与 AccountableResource 的关联与分歧

### 8.1 关联：Property 仍然 include 了 AccountableResource

前文说 Property "不使用 AccountableResource"是不准确的。实际代码（[properties_controller.rb#L2](file:///d:/fz/0601-1/solo-dogfeeding/code/5-maybe/app/controllers/properties_controller.rb#L2-L2)）：

```ruby
class PropertiesController < ApplicationController
  include AccountableResource, StreamExtensions
```

Property **确实 include 了 AccountableResource**，因此它继承了该 concern 提供的全部基础设施：

| 继承自 AccountableResource 的内容 | Property 是否使用 |
|------|:---:|
| `before_action :set_account, only: [:show, :edit, :update]` | ✅ `edit` 由 concern 的空方法体渲染，`update` 被 Property 重写 |
| `before_action :set_link_options, only: :new` | ✅ 但 Property 的 `new` 没有渲染 method_selector，所以 `@show_us_link` / `@show_eu_link` 虽被设置但从未在视图中使用 |
| `accountable_type` 方法（`controller_name.classify.constantize` → `Property`） | ✅ 隐式使用 |
| `set_account` 方法 | ✅ 被 `before_action` 调用 |
| `Periodable` concern | ✅ 被引入 |

### 8.2 分歧：Property 覆写了 4 个 concern 方法

Property 对 AccountableResource 的覆写是**选择性替换**，不是完全脱离：

| 方法 | concern 原始行为 | Property 覆写行为 | 分歧原因 |
|------|----------------|-----------------|---------|
| `new` | `build(currency: family.currency, accountable: accountable_type.new)` | `build(accountable: Property.new)` — **没有设 currency** | 向导 Step 1 不需要 currency/balance 字段，留到 Step 2 处理 |
| `create` | `create_and_sync(account_params)` → `lock_saved_attributes!` → `redirect_to @account` | `create!(property_params.merge(currency: ..., balance: 0, status: "draft"))` → `redirect_to balances_property_path` | 向导需要分步：Step 1 只创建 draft，Step 2 设 balance，Step 3 设 address，最终才 activate |
| `update` | 通用：处理 balance 更新 + 属性更新 + `lock_saved_attributes!` | Property 专用：更新 property_params 后根据 `active?` 判断重定向到 edit 还是 balances | 编辑已激活的 Property 与未完成的 Property 走不同路径 |
| `account_params` | `permit(:name, :balance, :subtype, :currency, :accountable_type, :return_to, accountable_attributes: [...])` | 替换为 `property_params`：`permit(:name, :subtype, :accountable_type, accountable_attributes: [...])` — **没有 :balance, :currency, :return_to** | Step 1 表单不提交 balance/currency，这些在 Step 2 单独处理 |

此外 Property 还新增了 concern 中没有的 4 个 action 和对应的 `before_action`：
- `balances` / `update_balances`：Step 2（balance 输入）
- `address` / `update_address`：Step 3（地址输入 + 激活）
- `before_action :set_property`：替代 `set_account`（额外设 `@property`），但只用于 Step 2/3/4

### 8.3 分歧的本质：draft → active 的生命周期差异

标准 AccountableResource 的 `create` 流程是**原子操作**：

```
用户填写 name + balance + (subtype)
  → create_and_sync（含 OpeningBalanceManager）
    → lock_saved_attributes!
      → redirect_to @account  （账户直接进入 active 状态）
```

Property 的流程是**多步事务**，状态沿 `draft → active` 渐进：

```
Step 1: name + subtype + year_built + area
  → create!（status: "draft", balance: 0）
    → redirect_to balances_property_path

Step 2: balance
  → update_balances（set_current_balance）
    → redirect_to address_property_path

Step 3: address
  → update_address（property.update + activate!）
    → redirect_to account_path(@account)  （此时才 active）
```

这导致 Property 不能使用 `create_and_sync`（会立即触发 sync 和 opening balance），也不能使用 concern 的 `account_params`（Step 1 不应接受 balance）。但 `edit` action 和 `set_account` 等"只读"基础设施仍直接复用 concern 的实现。

### 8.4 Property 没有进入 method_selector

Property 的 `new` action 没有 `step: "method_select"` 的分支判断——它的 [new.html.erb](file:///d:/fz/0601-1/solo-dogfeeding/code/5-maybe/app/views/properties/new.html.erb) 直接渲染表单，标题是 "Enter property manually"。也就是说，Property 类型**没有 Plaid 连接选项**，用户只能手动录入。这一点与 Depository/Investment/Crypto/CreditCard/Loan 的入口行为不同（后五者先展示 method_selector，提供"手动录入"和"Plaid 连接"两种选择）。

---

## 九、补充：method_select 入口对可见选项的影响

### 9.1 完整的用户入口流程

用户创建新账户时，实际经历了**三层选择**：

```
Layer 1: 选择 classification（asset / liability）
  → /accounts/new?classification=asset

Layer 2: 选择 accountable_type（Depository / Investment / ...）
  → 点击某个类型 → /depositories/new?step=method_select

Layer 3: 选择创建方式（手动 / Plaid US / Plaid EU）
  → 手动 → /depositories/new（表单）
  → Plaid → /plaid_items/new?accountable_type=Depository
```

### 9.2 Layer 1：classification 过滤

[accounts/new.html.erb](file:///d:/fz/0601-1/solo-dogfeeding/code/5-maybe/app/views/accounts/new.html.erb) 中，`params[:classification]` 控制哪些类型可见：

| classification 值 | 可见的 Accountable 类型 |
|---|---|
| 未传参（默认） | 全部 9 个类型 |
| `"asset"` | Depository, Investment, Crypto, Property, Vehicle, OtherAsset |
| `"liability"` | CreditCard, Loan, OtherLiability |

### 9.3 Layer 2：点击类型 → method_selector

[_account_type.html.erb](file:///d:/fz/0601-1/solo-dogfeeding/code/5-maybe/app/views/accounts/_account_type.html.erb) 中，点击某个类型后跳转的 URL 为：

```erb
<%= link_to new_polymorphic_path(accountable, step: "method_select", return_to: params[:return_to]) %>
```

即所有类型的默认入口都带 `step=method_select`。但各类型的 `new.html.erb` 对此参数的处理**分为两类**：

#### 类型 A：有 method_select 步骤（5 个类型）

Depository、Investment、CreditCard、Loan、**Crypto** 的 `new.html.erb` 均有如下分支：

```erb
<% if params[:step] == "method_select" %>
  <%= render "accounts/new/method_selector", ... %>
<% else %>
  <%= render DS::Dialog.new do |dialog| %>
    <% dialog.with_body do %>
      <%= render "xxx/form", ... %>
    <% end %>
  <% end %>
<% end %>
```

这 5 个类型在 method_selector 中都提供"手动录入"和"Plaid 连接"两种选项（前提是家庭配置了对应区域的 Plaid）。

#### 类型 B：没有 method_select 步骤（4 个类型）

Property、Vehicle、OtherAsset、OtherLiability 的 `new.html.erb` 直接渲染表单，**忽略 `step` 参数**：

```erb
<%= render DS::Dialog.new do |dialog| %>
  <% dialog.with_body do %>
    <%= render "xxx/form", ... %>
  <% end %>
<% end %>
```

尽管 `_account_type.html.erb` 跳转到这些类型时 URL 中也带了 `step=method_select`，但这些类型的 view 不检查该参数，直接进入表单。这 4 个类型**只能手动创建，不能通过 Plaid 连接创建**——这也解释了为什么它们不需要 method_selector。

### 9.4 method_selector 的可见选项

[accounts/new/_method_selector.html.erb](file:///d:/fz/0601-1/solo-dogfeeding/code/5-maybe/app/views/accounts/new/_method_selector.html.erb) 提供三个入口：

| 入口 | 链接目标 | 可见条件 |
|------|---------|---------|
| 手动录入 | `new_xxx_path(return_to: ...)` （同类型，无 step 参数） | **始终可见** |
| Plaid US 连接 | `new_plaid_item_path(region: "us", accountable_type: ...)` | `@show_us_link == true`（取决于家庭是否配置了 Plaid US） |
| Plaid EU 连接 | `new_plaid_item_path(region: "eu", accountable_type: ...)` | `@show_eu_link == true`（取决于家庭是否配置了 Plaid EU + 家庭是否为 EU 区域） |

`@show_us_link` 和 `@show_eu_link` 由 [accountable_resource.rb#L68-L71](file:///d:/fz/0601-1/solo-dogfeeding/code/5-maybe/app/controllers/concerns/accountable_resource.rb#L68-L71) 的 `set_link_options` before_action 设置：

```ruby
def set_link_options
  @show_us_link = Current.family.can_connect_plaid_us?
  @show_eu_link = Current.family.can_connect_plaid_eu?
end
```

### 9.5 accountable_type 对 Plaid Link Token 的影响

用户选择 Plaid 连接后，`accountable_type` 参数传入 [plaid_items_controller.rb#L11](file:///d:/fz/0601-1/solo-dogfeeding/code/5-maybe/app/controllers/plaid_items_controller.rb#L11-L11)：

```ruby
accountable_type: params[:accountable_type] || "Depository"
```

这个参数最终影响 Plaid Link Token 的产品配置（[plaid.rb#L182-L198](file:///d:/fz/0601-1/solo-dogfeeding/code/5-maybe/app/models/provider/plaid.rb#L182-L198)）：

```ruby
def get_primary_product(accountable_type)
  return "transactions" if eu?

  case accountable_type
  when "Investment"
    "investments"
  when "CreditCard", "Loan"
    "liabilities"
  else
    "transactions"
  end
end

def get_additional_consented_products(accountable_type)
  return [] if eu?

  MAYBE_SUPPORTED_PLAID_PRODUCTS - [ get_primary_product(accountable_type) ]
end
```

**这意味着 `accountable_type` 决定了 Plaid 在建立连接时请求哪个核心产品**：

| 用户选择的类型 | Plaid 主产品（US） | Plaid 额外同意产品（US） |
|---|---|---|
| Depository / Crypto / Property / Vehicle / OtherAsset / OtherLiability | `"transactions"` | `["investments", "liabilities"]` |
| Investment | `"investments"` | `["transactions", "liabilities"]` |
| CreditCard / Loan | `"liabilities"` | `["transactions", "investments"]` |

**EU 区域特殊处理**：EU 模式下只请求 `"transactions"` 产品，不请求额外同意产品。这是 Plaid EU API 的限制。

### 9.6 method_select 与 subtype 的关系

method_select 这一步**不涉及 subtype**。用户在 Layer 2 选择的是 `accountable_type`（如 "Depository"），不是 subtype（如 "checking"）。subtype 要等到 Layer 3 选择"手动录入"后，在具体类型的表单中才会出现。

但 `accountable_type` 通过上述 Plaid 产品选择机制，**间接决定了 Plaid 是否会拉取 liabilities 数据**，而 liabilities 数据的可用性又取决于账户的 `plaid_subtype`。这个因果链在第十章完整串联。

---

## 十、补充：Plaid 路径中 subtype 的两层作用串联

Plaid 导入路径中，subtype 实际上参与了**两个不同层次的决策**，前文将它们混在了一起，此处重新拆解并串联。

### 10.1 第一层：subtype 决定是否拉取 liabilities 数据

在 [accounts_snapshot.rb#L93-L98](file:///d:/fz/0601-1/solo-dogfeeding/code/5-maybe/app/models/plaid_item/accounts_snapshot.rb#L93-L98) 中，`can_fetch_liabilities?` 方法**使用 Plaid 返回的账户原始 subtype** 来决定是否调用 Liabilities API：

```ruby
def can_fetch_liabilities?
  plaid_item.supports_product?("liabilities") &&
  accounts.any? do |a|
    a.type == "credit" && a.subtype == "credit card" ||
    a.type == "loan" && (a.subtype == "mortgage" || a.subtype == "student")
  end
end
```

这里有两个前置条件：

**前置条件 A**：`plaid_item.supports_product?("liabilities")`

这个方法（[plaid_item.rb#L96-L98](file:///d:/fz/0601-1/solo-dogfeeding/code/5-maybe/app/models/plaid_item.rb#L96-L98)）检查 Plaid Item 是否支持 liabilities 产品：

```ruby
def supports_product?(product)
  supported_products.include?(product)
end

def supported_products
  available_products + billed_products
end
```

`available_products` 和 `billed_products` 是 Plaid 在 Link Token 创建后返回的。**它们是否包含 "liabilities" 取决于**：

- US 区域：Link Token 创建时 `products` 或 `additional_consented_products` 包含了 `"liabilities"`（即用户在 method_select 中选择了 CreditCard 或 Loan 类型，或者选择了其他类型但 Plaid 额外同意了 liabilities）
- EU 区域：永远不包含 `"liabilities"`（EU 模式只请求 `"transactions"`）

**前置条件 B**：该 Plaid Item 下的账户中至少有一个的 `[type, subtype]` 组合属于以下之一：

| Plaid type | Plaid subtype | 含义 |
|---|---|---|
| `"credit"` | `"credit card"` | 信用卡 |
| `"loan"` | `"mortgage"` | 房贷 |
| `"loan"` | `"student"` | 学生贷款 |

**注意**：`"loan"` + `"auto"` / `"business"` / `"home equity"` / `"line of credit"` **不满足**前置条件 B，即使 Plaid Item 支持 liabilities 产品，也不会调用 Liabilities API。

**第一层决策的完整因果链**：

```
用户在 method_select 选择了 CreditCard 或 Loan 类型
  → Link Token 的 primary_product = "liabilities"（US 区域）
    → Plaid 建立 Item 后，liabilities 出现在 available/billed_products 中
      → supports_product?("liabilities") = true

Plaid 返回的账户列表中，某些账户的 [type, subtype] 为
  ["credit", "credit card"] 或 ["loan", "mortgage"] 或 ["loan", "student"]
  → can_fetch_liabilities? = true
    → 调用 Plaid Liabilities API 拉取数据
      → 数据存储到 plaid_account.raw_liabilities_payload
```

如果用户选择了 Depository 类型，但 Plaid 额外同意了 liabilities 产品，且该 Item 下恰好也有 loan 类账户，liabilities 数据**也会被拉取**——因为 `can_fetch_liabilities?` 检查的是 Plaid 返回的所有账户，不仅限于用户选择的 accountable_type 对应的账户。

### 10.2 第二层：subtype 决定如何分发 liabilities 数据

当 liabilities 数据成功拉取后，[processor.rb#L77-L88](file:///d:/fz/0601-1/solo-dogfeeding/code/5-maybe/app/models/plaid_account/processor.rb#L77-L88) 的 `process_liabilities` 对**每个 PlaidAccount** 单独执行分发：

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

第二层的分发逻辑使用 `plaid_account.plaid_subtype`（Plaid 原始值），不使用映射后的 `account.subtype`。分发结果：

| [plaid_type, plaid_subtype] | Processor | 写入目标 | 写入的字段 |
|---|---|---|---|
| `["credit", "credit card"]` | CreditProcessor | `account.credit_card` | `minimum_payment`, `apr` |
| `["loan", "mortgage"]` | MortgageProcessor | `account.loan` | `rate_type`, `interest_rate` |
| `["loan", "student"]` | StudentLoanProcessor | `account.loan` | `rate_type="fixed"`, `interest_rate`, `initial_balance`, `term_months` |
| 其他所有组合 | *(nil，无 Processor)* | — | — |

### 10.3 两层作用的关键差异

| 维度 | 第一层（数据拉取） | 第二层（数据分发） |
|------|---|---|
| **决策对象** | 整个 PlaidItem（item 级别） | 单个 PlaidAccount（account 级别） |
| **判断的 subtype 来源** | Plaid API 返回的 `account.type` + `account.subtype`（AccountsSnapshot 中遍历所有账户） | 数据库中 `plaid_account.plaid_type` + `plaid_account.plaid_subtype`（已存储的快照） |
| **判断时机** | 数据获取阶段（AccountsSnapshot 构建时） | 数据处理阶段（Processor 执行时） |
| **影响** | 是否调用 `LiabilitiesGet` API（昂贵的外部调用） | 是否将拉取到的 liabilities 数据写入 CreditCard/Loan 专属字段 |
| **auto loan 的命运** | 不满足条件 B → 不会触发 Liabilities API 调用 | 即使被调用也不会匹配任何 Processor |

### 10.4 两层之间的串联

第一层和第二层的**解耦**导致一个重要结果：如果某个 Plaid Item 下同时有 credit card 和 auto loan 账户：

1. 第一层：`can_fetch_liabilities?` 扫描全部账户，发现存在 `["credit", "credit card"]` → 返回 true → 调用 Liabilities API → **拉取所有 liabilities 数据**（包括 credit、mortgage、student 三类）
2. 数据存储到每个 `plaid_account.raw_liabilities_payload`
3. 第二层：对 credit card 账户执行 CreditProcessor → 写入 credit_card 字段；对 auto loan 账户 → case 不匹配 → **不写入 loan 字段**，即使 `raw_liabilities_payload` 中可能存在 auto loan 的数据（Plaid Liabilities API 实际不返回 auto loan 的结构化数据）

这意味着：**auto loan 在第一层就被排除在 liabilities 数据拉取之外**（`can_fetch_liabilities?` 不认为它需要 liabilities），**在第二层也被排除在 processor 分发之外**（case 不匹配）。两层决策对 auto loan 的态度一致——忽略它。

但如果将来 Plaid Liabilities API 增加了 auto loan 数据支持，需要同时修改两处：`can_fetch_liabilities?` 的账户过滤条件 + `process_liabilities` 的 case 分支。目前的解耦设计使得这两处修改容易遗漏其中一处。
