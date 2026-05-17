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

#### 8.6.1 失败触发原因

加载失败主要由以下几类异常触发：

| 失败类型 | 触发源 | 典型场景 |
|----------|--------|----------|
| **网络超时** | 前端 Stimulus 控制器 | 服务器响应超过 10 秒未返回 |
| **数据库异常** | `AccountsController#show` | 查询超时、连接断开、锁等待 |
| **服务器内部错误** | 应用代码执行 | NPE、数组越界、第三方服务调用失败 |
| **数据同步失败** | 后台 Sync Job | Plaid 连接失败、市场数据导入错误 |

**关键代码说明：**

1. **前端超时检测**：`app/javascript/controllers/turbo_frame_timeout_controller.js:5-41`

```javascript
// 默认 10 秒超时，监听 turbo:frame-load 事件清除定时器
static values = { timeout: { type: Number, default: 10000 } }

handleTimeout() {
  // 直接替换 innerHTML 为错误状态
  this.element.innerHTML = `...错误UI...`
}
```

2. **后端异常类型**：
   - `ActiveRecord::QueryCanceled`：数据库查询超时
   - `ActiveRecord::ConnectionNotEstablished`：数据库连接失败
   - `PG::ConnectionBad`：PostgreSQL 连接异常
   - 通用 `StandardError`：应用代码异常

#### 8.6.2 状态切换链路

```
用户访问账户页面
        ↓
[浏览器] 发送 HTTP 请求 → 显示 Turbo Frame 加载态
        ↓
[Rails] 路由 → AccountsController#show
        ├─ 成功 → 渲染 ActivityFeed 组件 → 页面展示
        └─ 失败（异常抛出）
            ├─ 开发环境：显示 Rails 错误栈页面
            └─ 生产环境：
                ├─ 响应状态码 500
                ├─ 渲染 public/500.html 静态错误页
                └─ 或被 Turbo 捕获显示局部错误
```

**前端超时切换流程**：
```
页面加载 → Turbo Frame 显示 loading 内容
        ↓
setTimeout(handleTimeout, 10000) 启动
        ├─ 10秒内收到 turbo:frame-load → 清除定时器 → 正常展示
        └─ 10秒内未收到响应 → 触发 handleTimeout → 替换 innerHTML 为错误UI
```

#### 8.6.3 用户可见提示

根据失败类型不同，用户会看到不同的提示：

| 失败类型 | 用户可见内容 | 视觉样式 |
|----------|--------------|----------|
| **前端超时** | ⚠️ 黄色警告图标 + "Timeout" 文本 | 右对齐，小字体，警告色 |
| **服务器500错误** | "We're sorry, but something went wrong (500)" | 全屏静态错误页，红色标题 |
| **同步失败** | 同步状态区显示错误详情 | Plaid 账户卡片显示错误徽章 |

**超时错误 UI 代码**：`app/javascript/controllers/turbo_frame_timeout_controller.js:29-40`

```html
<div class="flex items-center justify-end gap-1">
  <div class="w-8 h-4 flex items-center justify-center">
    <svg ... class="text-warning">⚠️</svg>
  </div>
  <p class="font-mono text-right text-xs text-warning">Timeout</p>
</div>
```

#### 8.6.4 与其他边界场景的边界区别

| 场景 | 触发阶段 | 数据状态 | HTTP 状态码 | 用户感知 |
|------|----------|----------|-------------|----------|
| **空 Feed** | 数据查询后 | 查询成功，返回 0 条记录 | 200 OK | 正常页面，显示 "No entries yet" |
| **无权限** | 数据查询前 | 认证/授权失败，不执行查询 | 404 Not Found / 重定向 | 无法访问页面或跳转到首页 |
| **搜索无结果** | 数据查询后 | 查询成功，过滤后 0 条 | 200 OK | 正常页面，显示 "No entries yet" |
| **加载失败** | 数据查询中 | 查询未完成/异常终止 | 500 Internal Server Error / 无响应 | 错误页面或局部超时提示 |

**关键区分点：**
- **空 Feed vs 加载失败**：空 Feed 是**查询成功但无数据**，页面完整渲染；加载失败是**查询过程异常**，页面渲染中断
- **无权限 vs 加载失败**：无权限是**主动拒绝访问**，失败发生在业务逻辑之前；加载失败是**被动异常**，失败发生在业务逻辑执行中
- **搜索无结果 vs 加载失败**：搜索无结果是**过滤后无匹配**，属于正常业务逻辑分支；加载失败是**系统级异常**

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

1. **多态设计**：使用 delegated_type 统一三种活动类型，便于扩展新类型
2. **数据聚合对象**：ActivityFeedData 解决 N+1 问题，提供清晰的数据边界
3. **组件化视图**：将复杂 UI 拆分为 ActivityFeed → ActivityDate → BalanceReconciliation 三层组件
4. **账户类型差异化**：BalanceReconciliation 根据账户类型动态展示对账项目
5. **实时更新**：Turbo Stream 支持无刷新更新活动列表
6. **批量操作**：内置 bulk-select 控制器支持批量编辑条目
