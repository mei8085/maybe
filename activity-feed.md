# 账户活动信息流（Activity Feed）链路分析报告

## 1. 整体架构概览

账户活动信息流采用经典的 **MVC + 组件化** 架构，数据从底层模型通过聚合层、控制层最终流向视图组件。整个链路可以分为五个核心层级：

```
数据源层 (Entry/Entryable)
        ↓
数据聚合层 (ActivityFeedData)
        ↓
控制层 (AccountsController)
        ↓
组件层 (ActivityFeed / ActivityDate / BalanceReconciliation)
        ↓
模板层 (Valuation/Transaction/Trade 专用模板)
```

## 2. 数据源层：Entry 模型与多态设计

### 2.1 核心模型 Entry

**文件位置**：`app/models/entry.rb:1-99`

Entry 是活动信息流的基础数据单元，采用 **delegated_type** 多态设计支持三种活动类型：

```ruby
delegated_type :entryable, types: Entryable::TYPES, dependent: :destroy
```

### 2.2 Entryable 类型定义

**文件位置**：`app/models/entryable.rb:1-31`

```ruby
TYPES = %w[Valuation Transaction Trade]
```

| 类型 | 含义 | 所属领域 |
|------|------|----------|
| Valuation | 估值/余额调整 | 对账领域 |
| Transaction | 普通交易 | 交易领域 |
| Trade | 证券交易 | 持仓领域 |

### 2.3 数据排序规则

**文件位置**：`app/models/entry.rb:29-35`

信息流默认采用 `reverse_chronological` 排序：
- 主排序：`date: :desc`（日期倒序）
- 次排序：Valuation 类型排在 Transaction/Trade 之后
- 末排序：`created_at: :desc`（创建时间倒序）

## 3. 数据聚合层：ActivityFeedData

### 3.1 核心职责

**文件位置**：`app/models/account/activity_feed_data.rb:1-85`

`Account::ActivityFeedData` 是专门为活动信息流设计的数据聚合对象，主要职责：
- 避免 N+1 查询问题
- 按日期分组聚合 entries
- 预加载关联的 balance 和 transfer 数据
- 提供统一的数据接口给视图组件

### 3.2 数据结构

```ruby
ActivityDateData = Data.define(:date, :entries, :balance, :transfers)
```

每个日期分组包含：
- `date`：日期
- `entries`：当日所有活动条目
- `balance`：当日余额数据（用于对账）
- `transfers`：当日涉及的转账记录

### 3.3 聚合逻辑

#### 3.3.1 条目按日期分组
```ruby
def grouped_entries
  @grouped_entries ||= entries.group_by(&:date)
end
```

#### 3.3.2 余额数据预加载
```ruby
def balances_by_date
  @balances_by_date ||= begin
    dates = grouped_entries.keys
    account.balances
      .where(date: dates, currency: account.currency)
      .index_by(&:date)
  end
end
```

#### 3.3.3 转账关联查询
```ruby
def transfers_by_date
  @transfers_by_date ||= begin
    transfers = Transfer
      .where(inflow_transaction_id: transaction_ids)
      .or(Transfer.where(outflow_transaction_id: transaction_ids))
      .to_a
    # 按交易条目日期分组转账
  end
end
```

## 4. 控制层：AccountsController

### 4.1 信息流入口

**文件位置**：`app/controllers/accounts_controller.rb:17-26`

```ruby
def show
  @chart_view = params[:chart_view] || "balance"
  @tab = params[:tab]
  @q = params.fetch(:q, {}).permit(:search)
  
  # 1. 查询过滤后的条目
  entries = @account.entries.search(@q).reverse_chronological
  
  # 2. 分页处理
  @pagy, @entries = pagy(entries, limit: params[:per_page] || "10")
  
  # 3. 构建 feed 数据对象
  @activity_feed_data = Account::ActivityFeedData.new(@account, @entries)
end
```

### 4.2 搜索过滤能力

**文件位置**：`app/models/entry_search.rb:1-69`

`EntrySearch` 支持多种过滤维度：
- 名称模糊搜索（`search`）
- 金额范围过滤（`amount` + `amount_operator`）
- 日期范围过滤（`start_date` / `end_date`）
- 账户过滤（`accounts` / `account_ids`）

### 4.3 权限控制

通过 `Current.family` 实现数据隔离：
```ruby
def set_account
  @account = family.accounts.find(params[:id])  # 仅能访问当前 family 的账户
end

def family
  Current.family
end
```

## 5. 组件层：ActivityFeed 组件体系

### 5.1 ActivityFeed 主组件

**文件位置**：
- 类定义：`app/components/UI/account/activity_feed.rb:1-35`
- 模板：`app/components/UI/account/activity_feed.html.erb:1-94`

#### 核心功能：
1. Turbo Stream 实时刷新支持
2. 搜索框（自动提交表单）
3. 空状态处理
4. 批量选择功能
5. 分页渲染

### 5.2 ActivityDate 日期分组组件

**文件位置**：
- 类定义：`app/components/UI/account/activity_date.rb:1-31`
- 模板：`app/components/UI/account/activity_date.html.erb:1-42`

#### 核心功能：
1. 按日期折叠/展开
2. 显示当日条目数量
3. 显示当日终余额
4. 嵌入 BalanceReconciliation 对账组件
5. 渲染当日所有 entries

### 5.3 BalanceReconciliation 对账组件

**文件位置**：
- 类定义：`app/components/UI/account/balance_reconciliation.rb:1-155`
- 模板：`app/components/UI/account/balance_reconciliation.html.erb:1-22`

这是信息流与**对账领域**协作的核心组件，根据账户类型展示不同的对账明细：

| 账户类型 | 对账项目 |
|----------|----------|
| Depository/OtherAsset/OtherLiability | 期初余额 → 净现金流 → 调整项 → 期末余额 |
| CreditCard | 期初余额 → 支出 → 还款 → 调整项 → 期末余额 |
| Investment | 期初余额 → 现金变动 → 持仓买卖 → 市价变动 → 调整项 → 期末余额 |
| Loan | 期初本金 → 本金净变动 → 调整项 → 期末本金 |
| Property/Vehicle | 期初价值 → 价值净变动 → 调整项 → 期末价值 |
| Crypto | 期初余额 → 买入 → 卖出 → 市价变动 → 调整项 → 期末余额 |

## 6. 模板层：不同类别活动的展示模板

### 6.1 模板分发机制

**文件位置**：`app/views/entries/_entry.html.erb:1-4`

```erb
<%= render partial: entry.entryable.to_partial_path,
           locals: { entry: entry, balance_trend: balance_trend, view_ctx: view_ctx } %>
```

Rails 约定自动映射到对应类型的部分模板：
- `Valuation` → `app/views/valuations/_valuation.html.erb`
- `Transaction` → `app/views/transactions/_transaction.html.erb`
- `Trade` → `app/views/trades/_trade.html.erb`

### 6.2 Valuation 模板（对账领域）

**文件位置**：`app/views/valuations/_valuation.html.erb:1-33`

展示特点：
- 紫色/灰色图标区分开户锚点与普通调整
- 显示估值金额
- 支持点击查看详情

### 6.3 Transaction 模板（交易领域）

**文件位置**：`app/views/transactions/_transaction.html.erb:1-103`

展示特点：
- 显示商家 logo 或文字图标
- 标记转账/一次性交易等特殊类型
- 显示分类标签
- 金额根据收支显示不同颜色（绿色表示收入）
- 排除条目显示半透明效果

### 6.4 Trade 模板（持仓领域）

**文件位置**：`app/views/trades/_trade.html.erb:1-43`

展示特点：
- 显示交易名称（如 "Buy 10 shares of AAPL"）
- 显示分类徽章
- 金额着色规则与 Transaction 一致

## 7. 与三大领域的协作机制

### 7.1 与对账领域的协作

1. **数据关联**：ActivityFeedData 预加载每日 balance 数据
2. **组件嵌入**：ActivityDate 模板中嵌入 BalanceReconciliation 组件
3. **分类展示**：BalanceReconciliation 根据账户类型动态生成对账项目
4. **调整标记**：Valuation 条目作为对账调整的可视化体现

### 7.2 与交易领域的协作

1. **多态关联**：Transaction 通过 Entryable 接口接入信息流
2. **转账识别**：ActivityFeedData 自动识别关联的 Transfer 记录
3. **类型区分**：Transaction 支持多种 kind（standard/funds_movement/cc_payment/loan_payment/one_time）
4. **分类系统**：Transaction 关联 Category、Merchant、Tag 等业务对象

### 7.3 与持仓领域的协作

1. **交易记录**：Trade 类型记录证券买卖活动
2. **证券关联**：Trade belongs_to Security，支持查询当前价格
3. **收益计算**：Trade 提供 `unrealized_gain_loss` 方法计算浮动盈亏
4. **余额对账**：Investment 类型账户在 BalanceReconciliation 中区分现金与持仓变动

## 8. 边界场景处理

### 8.1 空 Feed 场景

**文件位置**：`app/components/UI/account/activity_feed.html.erb:53-54`

```erb
<% if activity_dates.empty? %>
  <p class="text-secondary text-sm p-4">No entries yet</p>
<% else %>
```

- 触发条件：`activity_dates.empty?` 为 true
- 呈现方式：简洁的文本提示，居中显示
- 全局空状态：`app/views/entries/_empty.html.erb` 提供更丰富的空状态模板（标题 + 描述）

### 8.2 无权限场景

通过多层数据隔离实现：
1. **控制器层**：`Current.family.accounts.find(params[:id])` 确保只能访问当前家庭的账户
2. **模型层**：`Entry.visible` scope 过滤掉非 active/draft 状态的账户条目
3. **失败模式**：找不到账户时抛出 ActiveRecord::RecordNotFound，Rails 默认返回 404

### 8.3 加载中场景

**文件位置**：`app/views/entries/_loading.html.erb:1-5`

```erb
<div class="bg-container space-y-4 p-5 shadow-border-xs rounded-xl">
  <div class="p-5 flex justify-center items-center">
    <%= tag.p t(".loading"), class: "text-secondary animate-pulse text-sm" %>
  </div>
</div>
```

- 呈现方式：脉冲动画的 "Loading..." 文本
- 使用时机：Turbo 加载或异步刷新时显示骨架屏

### 8.4 余额数据缺失场景

**文件位置**：`app/components/UI/account/activity_date.html.erb:29-33`

```erb
<% if balance %>
  <%= render UI::Account::BalanceReconciliation.new(balance: balance, account: account) %>
<% else %>
  <p class="text-sm text-secondary">No balance data available for this date</p>
<% end %>
```

### 8.5 搜索无结果场景

与空 Feed 场景复用相同逻辑，搜索过滤后无结果时显示 "No entries yet"

### 8.6 加载失败场景

#### 8.6.1 组件超时错误态确认

**ActivityFeed 组件本体不存在独立的超时错误态**。代码证据：

- `app/components/UI/account/activity_feed.html.erb:1` 中的 Turbo Frame 定义：
  ```erb
  <%= turbo_frame_tag dom_id(account, "entries") do %>
  ```
  未添加 `data: { controller: "turbo-frame-timeout" }` 属性，因此不会触发前端超时检测。

**仅 sparkline 组件使用超时机制**：

- `app/views/accounts/_accountable_group.html.erb:16` 和 `:43` 两处 sparkline 配置：
  ```erb
  <%= turbo_frame_tag dom_id(account, :sparkline), 
      src: sparkline_account_path(account), 
      loading: "lazy", 
      data: { controller: "turbo-frame-timeout", turbo_frame_timeout_timeout_value: 10000 } do %>
  ```
  这是仓内唯一使用 `turbo-frame-timeout` 控制器的场景，与 Activity Feed 组件无关联。

#### 8.6.2 失败触发原因

Activity Feed 加载失败仅能通过 Rails 全局异常处理机制触发，仓内可定位的触发点：

| 失败类型 | 触发代码位置 | 触发条件 |
|----------|--------------|----------|
| **账户不存在** | `app/controllers/accounts_controller.rb:70-72` | `family.accounts.find(params[:id])` 查找失败 |
| **控制器执行异常** | `app/controllers/accounts_controller.rb:17-26` | `show` 动作中任意代码抛出未捕获异常 |
| **模板渲染异常** | `app/components/UI/account/activity_feed.html.erb` | 模板渲染过程中发生错误 |

**关键代码证据：**

1. **账户查找**：`app/controllers/accounts_controller.rb:70-72`
   ```ruby
   def set_account
     @account = family.accounts.find(params[:id])
   end
   ```

2. **全局异常配置**：
   - 开发环境：`config/environments/development.rb:15` → `config.consider_all_requests_local = true`
   - 生产环境：`config/environments/production.rb:16` → `config.consider_all_requests_local = false`

#### 8.6.3 状态切换链路

```
用户点击账户链接 / 输入账户 URL
        ↓
[浏览器] 发送 GET /accounts/:id 请求
        ↓
[Rails 路由] 匹配到 AccountsController#show
        ↓
[过滤器] 执行 set_account
        ├─ 成功 → 继续执行 show 动作
        └─ 失败（ActiveRecord::RecordNotFound）
            ├─ 被 StoreLocation concern 捕获
            ├─ 调用 handle_not_found 方法
            └─ 返回 404 响应或重定向
        ↓
[控制器] 执行 show 动作
        ├─ entries 查询、分页、构建 ActivityFeedData
        ├─ 成功 → 渲染 accounts/show.html.erb 模板
        └─ 失败（任意未捕获异常）
            ├─ 开发环境：渲染 Rails 错误栈页面
            └─ 生产环境：
                ├─ 设置响应状态码 500
                └─ 渲染 public/500.html 静态错误页
        ↓
[模板渲染] 渲染 ActivityFeed 组件
        ├─ 成功 → 输出完整 HTML 到浏览器
        └─ 失败（模板错误）→ 同上异常处理流程
```

**无前端超时切换**：由于 ActivityFeed 未绑定 `turbo-frame-timeout` 控制器，不存在从加载态自动切换到错误态的前端逻辑。

#### 8.6.4 用户可见提示

根据失败阶段不同，用户可见内容：

| 失败阶段 | 代码位置 | 用户可见内容 |
|----------|----------|--------------|
| **账户不存在** | `app/controllers/concerns/store_location.rb:17-22` | 空白页面（`head :not_found`）或重定向到首页 |
| **控制器/模板异常** | `public/500.html` | 全屏静态错误页："We're sorry, but something went wrong (500)" |
| **开发环境异常** | Rails 中间件 | 完整错误栈追踪页面，包含异常类型、调用链、请求参数 |

**无专用错误 UI 组件**：仓内未发现 ActivityFeed 专用的错误态模板或组件。

#### 8.6.5 可复现证据矩阵

| 入口场景 | 请求路径 | 命中代码分支 | HTTP 状态码 | 页面可见结果 |
|----------|----------|--------------|-------------|--------------|
| **不存在账户直连** | `GET /accounts/999999999` | `accounts_controller.rb:70` → `family.accounts.find` 抛出 `ActiveRecord::RecordNotFound` → `rescue_from` 触发 `handle_not_found` → 比较 `request.fullpath == session[:return_to]`（session 为空）→ `else` 分支 | `404 Not Found` | 空白页面，无任何内容 |
| **单请求带 return_to** | `GET /accounts/999999999?return_to=/accounts/999999999` | `before_action :store_return_to` 执行 → `session[:return_to] = "/accounts/999999999"` → 执行 `set_account` → `find` 抛出 `RecordNotFound` → `handle_not_found` 比较 `request.fullpath == session[:return_to]`（两者相等）→ `if` 分支 | `302 Found` 重定向 | 跳转到首页（`/`），无错误提示 |
| **跨请求带 return_to** | 请求1：`GET /any_page?return_to=/accounts/999999999`<br>请求2：`GET /accounts/999999999` | 请求1：`store_return_to` 写入 session → 请求2：`store_return_to` 不修改 session → `set_account` 抛出 `RecordNotFound` → `handle_not_found` 比较相等 → `if` 分支 | `302 Found` 重定向 | 跳转到首页（`/`），无错误提示 |
| **控制器异常导致 500** | `GET /accounts/:id`（在 `show` 动作中人为触发异常） | `accounts_controller.rb:17-26` 中任意代码抛出未捕获异常 → Rails 全局异常处理 | `500 Internal Server Error` | 全屏静态错误页："We're sorry, but something went wrong (500)" |

**复现步骤说明：**

1. **不存在账户直连**：
   - 登录后直接在浏览器地址栏输入 `/accounts/999999999`（确保该 ID 不存在）
   - 执行顺序：`store_return_to`（无操作，无 return_to 参数）→ `set_account` → 抛出 `RecordNotFound` → `handle_not_found` → `session[:return_to]` 为 nil → 进入 `else` 分支
   - 代码分支：`store_location.rb:22` → `head :not_found`
   - 验证：浏览器开发者工具 Network 面板显示 404 状态，页面空白

2. **单请求带 return_to（真实触发场景）**：
   - 登录后访问 `/accounts/999999999?return_to=/accounts/999999999`
   - 执行顺序：`store_return_to` 写入 session → `set_account` 抛出异常 → `handle_not_found` 比较相等
   - 代码分支：`store_location.rb:18-20` → `redirect_to fallback_path`
   - 验证：浏览器 Network 面板显示 302，随后跳转到首页

3. **跨请求带 return_to（真实触发场景）**：
   - 先访问任意页面并携带 return_to 参数：`/dashboard?return_to=/accounts/999999999`
   - 再访问不存在的账户：`/accounts/999999999`（不带 return_to 参数）
   - 执行顺序：请求1写入 session → 请求2的 `store_return_to` 不修改 → `set_account` 抛出异常 → 比较相等
   - 代码分支：`store_location.rb:18-20` → 重定向到首页
   - 验证：Network 面板显示 302 重定向

4. **控制器异常导致 500**（开发环境验证）：
   - 在 `accounts_controller.rb:17` 的 `show` 方法第一行添加 `raise "test error"`
   - 访问任意存在的账户详情页
   - 代码分支：Rails 异常中间件捕获，开发环境显示错误栈，生产环境渲染 `public/500.html`
   - 验证：Network 面板显示 500，页面显示静态错误页

**关键执行顺序说明（基于 store_location.rb）：**
```
[过滤器链] before_action :store_return_to → 其他 before_action（如 set_account）
                                                          ↓
                                                    抛出 RecordNotFound
                                                          ↓
[异常处理] rescue_from 捕获 → handle_not_found → 比较 request.fullpath == session[:return_to]
```
由于 `store_return_to` 是 `before_action`，且在 `included` 块中最先声明，因此它总是在 `set_account` 之前执行。

#### 8.6.6 与其他边界场景的边界区别

| 场景 | 代码判断位置 | 判断逻辑 | HTTP 状态 | 页面完整性 |
|------|--------------|----------|-----------|------------|
| **空 Feed** | `app/components/UI/account/activity_feed.html.erb:53-54` | `activity_dates.empty?` 为 true | 200 OK | 完整页面，仅 feed 区域显示提示 |
| **搜索无结果** | `app/components/UI/account/activity_feed.html.erb:53-54` | 搜索过滤后 `activity_dates.empty?` 为 true | 200 OK | 完整页面，仅 feed 区域显示提示 |
| **无权限（无 return_to）** | `app/controllers/concerns/store_location.rb:22` | `find` 抛出 `RecordNotFound`，session 无 return_to | 404 Not Found | 空白页面 |
| **无权限（带 return_to）** | `app/controllers/concerns/store_location.rb:18-20` | `find` 抛出 `RecordNotFound`，请求路径与 session 中 return_to 相等 | 302 Found 重定向 | 跳转到首页 |
| **加载失败** | Rails 全局异常中间件 | 控制器/模板执行中抛出未捕获异常 | 500 Internal Server Error | 全屏错误页，无应用布局 |

**边界区分代码证据：**

1. **空 Feed 与加载失败的边界**：
   - 空 Feed 是**条件分支**：`if activity_dates.empty?` 明确判断数据存在性
   - 加载失败是**异常抛出**：无代码判断，直接中断执行流
   - 关键区分：空 Feed 会完整渲染搜索框、标题等页面元素；加载失败会完全跳过这些渲染

2. **无权限与加载失败的边界**：
   - 无权限是**预期内的查找失败**：`find` 方法语义上允许找不到记录
   - 加载失败是**预期外的执行错误**：如数据库连接断开、代码 bug 等
   - 关键区分：无权限由 `rescue_from ActiveRecord::RecordNotFound` 显式处理；加载失败由 Rails 全局异常兜底

3. **搜索无结果与加载失败的边界**：
   - 搜索无结果是**过滤后的正常空集**：`EntrySearch` 正常执行，返回 0 条匹配
   - 加载失败是**搜索过程本身出错**：如 SQL 语法错误、数据库超时
   - 关键区分：搜索无结果仍会显示搜索框，用户可修改搜索条件；加载失败无交互元素

## 9. 实时刷新机制

### 9.1 Turbo Stream 广播

**文件位置**：`app/components/UI/account/activity_feed.rb:18-25`

```ruby
def broadcast_refresh!
  Turbo::StreamsChannel.broadcast_replace_to(
    broadcast_channel,
    target: id,
    renderable: self,
    layout: false
  )
end
```

### 9.2 频道定义

- ActivityFeed：`broadcast_channel` 直接使用 account 对象作为频道标识
- ActivityDate：同样使用 account 作为频道，支持按日期分组刷新

## 10. 关键设计决策总结

1. **多态设计**：使用 delegated_type 统一三种活动类型（Valuation/Transaction/Trade），便于扩展新类型
2. **数据聚合对象**：ActivityFeedData 解决 N+1 问题，按日期预加载 balance 和 transfer 数据，提供清晰的数据边界
3. **组件化视图**：将复杂 UI 拆分为 ActivityFeed → ActivityDate → BalanceReconciliation 三层组件，职责分离
4. **账户类型差异化**：BalanceReconciliation 根据 7 种账户类型动态生成对账项目，提供针对性展示
5. **实时更新**：Turbo Stream 支持无刷新更新活动列表，频道标识复用 account 对象
6. **批量操作**：内置 bulk-select 控制器支持全选/按日期分组选择，方便批量编辑
7. **边界场景分层处理**：空状态在组件层判断，无权限在过滤器层处理，加载失败由 Rails 全局异常兜底
