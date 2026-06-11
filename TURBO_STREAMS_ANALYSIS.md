# Turbo Streams 局部刷新机制代码分析报告

## 一、技术栈概述

本项目基于 **Rails 7.2 + Hotwire/Turbo** 技术栈实现服务端推送的局部刷新：

- **框架**: Rails 7.2.2 + turbo-rails gem
- **传输层**: ActionCable (WebSocket) + HTTP 响应
- **客户端**: `@hotwired/turbo-rails` JavaScript 库
- **组件**: ViewComponent + Stimulus
- **页面刷新策略**: morph（idomorph 增量 DOM diff），滚动位置保持

核心依赖定义在 [Gemfile](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/Gemfile#L24) 和 [importmap.rb](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/config/importmap.rb#L4) 中。

---

## 二、服务端推送通道建立

### 2.1 通道订阅（客户端）

客户端通过 `turbo_stream_from` 辅助方法建立 ActionCable 连接，订阅特定频道的更新：

```erb
<!-- 全局家庭频道 - 布局层，所有页面均可用 -->
<%= turbo_stream_from Current.family %>
```
[app/views/layouts/shared/_htmldoc.html.erb](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/views/layouts/shared/_htmldoc.html.erb#L29-L31)

```erb
<!-- 聊天频道 - 页面层，仅聊天页订阅 -->
<%= turbo_stream_from @chat %>
```
[app/views/chats/show.html.erb](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/views/chats/show.html.erb#L3)

```erb
<!-- 账户频道 - 组件层，账户详情页独享 -->
<%= turbo_stream_from account %>
```
[app/components/UI/account_page.html.erb](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/components/UI/account_page.html.erb#L1)

三层订阅形成级联推送网络：Family 频道负责全局仪表盘更新，Account 频道负责账户页内局部刷新，Chat 频道负责聊天消息流。

### 2.2 推送方式（服务端）

#### 方式一：模型回调自动广播
```ruby
class Message < ApplicationRecord
  after_create_commit -> { broadcast_append_to chat, target: "messages" }, if: :broadcast?
  after_update_commit -> { broadcast_update_to chat }, if: :broadcast?
end
```
[app/models/message.rb](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/models/message.rb#L13-L14)

#### 方式二：显式调用广播 API
```ruby
Turbo::StreamsChannel.broadcast_replace_to(
  broadcast_channel,
  target: id,
  renderable: self,
  layout: false
)
```
[app/components/UI/account/activity_date.rb](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/components/UI/account/activity_date.rb#L24-L29)

#### 方式三：控制器响应 Turbo Stream 格式
```ruby
format.turbo_stream do
  render turbo_stream: [
    turbo_stream.replace(dom_id(@entry, :header), partial: "..."),
    turbo_stream.replace(@entry),
    *flash_notification_stream_items
  ]
end
```
[app/controllers/transactions_controller.rb](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/controllers/transactions_controller.rb#L94-L104)

#### 方式四：broadcast_refresh（Page Refresh 广播）
```ruby
account.broadcast_refresh
```
[app/models/account/sync_complete_event.rb](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/models/account/sync_complete_event.rb#L36)

此方式向订阅了该 account 频道的客户端发送 `<turbo-stream action="refresh">`，触发客户端对当前页面执行一次全页面重新获取与 morph 渲染（详见第六章）。

---

## 三、目标定位机制

### 3.1 目标元素 ID 生成规则

#### 规则一：基于 `dom_id` 的自动生成
```ruby
dom_id(@entry)           # => "entry_123"
dom_id(@entry, :header)  # => "header_entry_123"
dom_id(transaction, "category_menu")  # => "category_menu_transaction_456"
```

#### 规则二：固定字符串 ID
对于全局或语义明确的元素，直接使用固定 ID：
- `"main"` - 主内容区域
- `"messages"` - 消息列表容器
- `"notification-tray"` - 通知托盘
- `"cta"` - 行动召唤区域
- `"balance-sheet"` - 仪表盘资产负债表
- `"net-worth-chart"` - 净值图表
- `"thinking-indicator"` - AI 思考指示器
- `"chat-error"` - 聊天错误提示

### 3.2 replace 与 update 的实际 DOM 落点对照

**关键区别**：`replace` 替换目标元素本身（外层标签一起换），`update` 只替换目标元素的 innerHTML（保留外层标签和属性）。

#### replace 落点对照表

| 推送方 target | 视图中对应的 DOM 元素 | 外层包裹 | 替换结果 |
|--------------|----------------------|----------|----------|
| `dom_id(@entry)` → `"entry_123"` | `<turbo-frame id="entry_123">` ([_transaction.html.erb](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/views/transactions/_transaction.html.erb#L5)) | `turbo_frame_tag` | 整个 `<turbo-frame>` 标签被替换 |
| `dom_id(@entry, :header)` → `"header_entry_123"` | `<header id="header_entry_123">` ([_header.html.erb](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/views/transactions/_header.html.erb#L3)) | `tag.header` | `<header>` 标签整体替换 |
| `dom_id(@transfer.inflow_transaction, "category_menu")` | `<div id="category_menu_transaction_456">` ([_transaction_category.html.erb](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/views/transactions/_transaction_category.html.erb#L3)) | 普通 `div` | `<div>` 标签整体替换 |
| `dom_id(@transfer.inflow_transaction, "transfer_match")` | `<div id="transfer_match_transaction_456">` ([_transfer_match.html.erb](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/views/transactions/_transfer_match.html.erb#L3)) | 普通 `div` | `<div>` 标签整体替换 |
| `"account_#{account.id}"` | `<turbo-frame id="account_1">` ([_account.html.erb](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/views/accounts/_account.html.erb#L3)) | `turbo_frame_tag` | 整个 `<turbo-frame>` 标签被替换 |
| `dom_id(account, :container)` → `"account_1_container"` | `<turbo-frame id="account_1_container">` ([account_page.html.erb](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/components/UI/account_page.html.erb#L3)) | `turbo_frame_tag` | 整个 `<turbo-frame>` 标签被替换 |
| `dom_id(account, :activity_feed)` | ActivityFeed 组件容器 | `turbo_frame_tag` | 整个 frame 被替换 |
| `dom_id(account, "entries_#{date}")` | ActivityDate 组件容器 | 普通 `div` | `<div>` 标签整体替换 |
| `"balance-sheet"` | 仪表盘中 Balance Sheet 区域 | 普通 `div` | `<div>` 标签整体替换 |
| `"net-worth-chart"` | 仪表盘中净值图表区域 | 普通 `div` | `<div>` 标签整体替换 |

#### update 落点对照表

| 推送方 target | 视图中对应的 DOM 元素 | 替换结果 |
|--------------|----------------------|----------|
| `"main"` | 主内容区 `<div id="main">` | 只替换 `main` 的 innerHTML，外层 `div` 及其属性保留 |
| `dom_id(sibling, :form)` | 预算分类表单容器 | 只替换表单内部，外层容器保留 |
| `dom_id(budget, :allocation_progress)` | 预算分配进度区域 | 只替换进度内部内容 |

#### 关键陷阱：replace 遇 turbo_frame_tag

当 `turbo_stream.replace` 的目标是 `<turbo-frame>` 元素时，替换的是整个 frame 标签本身。如果推送内容中也包含同 id 的 `<turbo-frame>` 标签，则 Turbo Frame 功能正常延续；但如果推送内容不包含 `<turbo-frame>` 包裹，frame 语义会丢失。项目中所有 replace 推送的目标 `<turbo-frame>` 均通过 partial 渲染，partial 内部仍以 `turbo_frame_tag` 包裹，确保了语义一致性。

### 3.3 目标定位匹配流程

1. 服务端推送 Turbo Stream 消息，包含 `target` 属性
2. 客户端 Turbo 库接收消息，通过 `document.getElementById(target)` 查找目标元素
3. 如果找到目标元素，执行对应的动作；如果未找到，**静默忽略**该条消息

---

## 四、替换语义（动作类型）

项目中使用了以下 7 种 Turbo Stream 动作（含 1 种自定义动作）：

### 4.1 `replace` - 替换整个元素

**语义**: 用新内容完全替换目标元素（包括元素本身及其标签）

**DOM 变更**: `targetElement.replaceWith(newContent)`

```ruby
turbo_stream.replace @entry
turbo_stream.replace dom_id(@transfer.inflow_transaction, "category_menu"),
                    partial: "transactions/transaction_category",
                    locals: { transaction: @transfer.inflow_transaction }
```
[app/views/transfers/update.turbo_stream.erb](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/views/transfers/update.turbo_stream.erb#L2-L11)

### 4.2 `update` - 更新元素内部内容

**语义**: 只替换目标元素的内部 HTML，保留元素本身及其属性

**DOM 变更**: `targetElement.innerHTML = newContent`

```ruby
turbo_stream.update "main" do
  # ... 新内容
end
```
[app/views/settings/api_keys/created.turbo_stream.erb](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/views/settings/api_keys/created.turbo_stream.erb#L1)

```ruby
turbo_stream.update dom_id(sibling, :form),
                    partial: "budget_categories/budget_category_form",
                    locals: { budget_category: sibling }
```
[app/views/budget_categories/update.turbo_stream.erb](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/views/budget_categories/update.turbo_stream.erb#L10-L11)

### 4.3 `remove` - 移除元素

**语义**: 从 DOM 中完全删除目标元素

**DOM 变更**: `targetElement.remove()`

```ruby
chat.broadcast_remove target: "thinking-indicator"
chat.broadcast_remove target: "chat-error"
```
[app/models/assistant/broadcastable.rb](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/models/assistant/broadcastable.rb#L9-L11) / [app/models/chat.rb](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/models/chat.rb#L50-L53)

### 4.4 `append` - 追加内容

**语义**: 在目标元素的子元素末尾添加新内容

**DOM 变更**: `targetElement.append(newContent)`

```ruby
after_create_commit -> { broadcast_append_to chat, target: "messages" }, if: :broadcast?
```
[app/models/message.rb](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/models/message.rb#L13)

```ruby
turbo_stream.append("notification-tray", **notification)
```
[app/controllers/concerns/notifiable.rb](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/controllers/concerns/notifiable.rb#L25)

### 4.5 `prepend` - 前置内容

**语义**: 在目标元素的子元素开头添加新内容（框架支持，项目中未直接使用）

### 4.6 `refresh` - 页面刷新（Morph 模式）

**语义**: 触发客户端重新获取当前页面并做增量 DOM diff 合并

**DOM 变更**: 通过 idiomorph 算法对 `<body>` 做精细化 diff，只修改差异节点

```ruby
account.broadcast_refresh
```
[app/models/account/sync_complete_event.rb](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/models/account/sync_complete_event.rb#L36)

详见第六章"morph 刷新"。

### 4.7 自定义动作 `redirect` - 页面跳转

**语义**: 触发整页导航跳转

服务端定义：
```ruby
render turbo_stream: turbo_stream.action(:redirect, redirect_target_url)
```
[app/controllers/concerns/stream_extensions.rb](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/controllers/concerns/stream_extensions.rb#L18)

客户端实现：
```javascript
Turbo.StreamActions.redirect = function () {
  Turbo.visit(this.target);
};
```
[app/javascript/application.js](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/javascript/application.js#L5-L7)

---

## 五、Turbo Frame 层级导航体系

### 5.1 Frame 层级结构

项目中的 Turbo Frame 构成三级导航体系：

```
顶层页面 (Page)
├── <turbo-frame id="modal">          ← 模态框，全局唯一
├── <turbo-frame id="drawer">         ← 侧滑抽屉，全局唯一
├── <turbo-frame id="sidebar_chat">   ← 右侧聊天面板（turbo_permanent）
│   └── <turbo-frame id="chat_title_X"> ← 聊天标题（嵌套）
└── <turbo-frame id="account_1_container"> ← 账户详情页
    ├── <turbo-frame id="entry_123">   ← 条目外层
    │   └── <turbo-frame id="transaction_456"> ← 交易内层
    └── <turbo-frame id="account_1_entries"> ← 活动列表
```

Frame 骨架定义在布局文件：
```erb
<%= turbo_frame_tag "modal" %>
<%= turbo_frame_tag "drawer" %>
```
[app/views/layouts/shared/_htmldoc.html.erb](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/views/layouts/shared/_htmldoc.html.erb#L33-L34)

### 5.2 `turbo_frame: "_top"` —— 顶层 Frame 导航

`data-turbo-frame="_top"` 明确指示 Turbo 突破当前 Frame 上下文，以整页方式导航。这是项目中**最常用的"跳出 Frame"机制**，共出现 33 处。

**触发场景分类**：

| 场景 | 示例 | 代码位置 |
|------|------|----------|
| 从 Frame 内链接跳转整页 | 账户名点击跳转账户详情 | [_account.html.erb](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/views/accounts/_account.html.erb#L19) |
| Frame 内表单提交需整页 | 转账表单提交 | [_form.html.erb](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/views/transfers/_form.html.erb#L1) |
| Frame 内删除操作 | 批量删除条目 | [_selection_bar.html.erb](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/views/transactions/_selection_bar.html.erb#L17) |
| 翻页导航 | 分页链接 | [_pagination.html.erb](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/views/shared/_pagination.html.erb#L8) |
| 设置页面跳转 | 分类/标签/规则编辑 | [index.html.erb](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/views/transactions/index.html.erb#L8-L12) |

```erb
<!-- 从 Frame 内的账户名链接跳转整页 -->
<%= link_to account.name, account, data: { turbo_frame: "_top" } %>

<!-- 表单提交后整页刷新 -->
<%= styled_form_with model: transfer, data: { turbo_frame: "_top" } do |f| %>
```

### 5.3 `turbo_frame: "modal"` —— 模态框导航

链接或表单指定 `data-turbo-frame="modal"` 时，响应内容被注入到 `<turbo-frame id="modal">` 中，由 [dialog_controller.js](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/components/DS/dialog_controller.js) 管理的 `<dialog>` 元素承接。

典型场景：编辑分类、编辑标签、新建交易、新建估值等弹出式操作。

```erb
<% menu.with_item(variant: "link", text: "Edit", href: edit_category_path(category),
    data: { turbo_frame: "modal" }) %>
```
[app/views/categories/_category.html.erb](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/views/categories/_category.html.erb#L16)

### 5.4 `turbo_frame: "drawer"` —— 侧滑抽屉导航

链接指定 `data-turbo-frame="drawer"` 时，响应内容被注入到 `<turbo-frame id="drawer">` 中，以侧滑抽屉形式呈现。

典型场景：查看交易详情、查看估值详情、查看持仓详情。

```erb
<%= link_to entry.name, entry_path(entry),
    data: { turbo_frame: "drawer", turbo_prefetch: false } %>
```
[app/views/transactions/_transaction.html.erb](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/views/transactions/_transaction.html.erb#L43)

### 5.5 `turbo_permanent` —— 聊天面板持久化

```erb
<%= tag.div id: "chat-container", data: { controller: "chat hotkey", turbo_permanent: true } do %>
```
[app/views/layouts/application.html.erb](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/views/layouts/application.html.erb#L140)

`data-turbo-permanent="true"` 使该元素在 Turbo 页面导航过程中被保留，不会被重新渲染。聊天面板在用户浏览不同页面时保持状态不丢失，内部通过 `<turbo-frame id="sidebar_chat" src="...">` 懒加载聊天内容。

### 5.6 Turbo Frame 响应不匹配时的自动升级

当请求由某个 `<turbo-frame>` 发起（如 modal 或 drawer），但服务端响应中不包含匹配 `id` 的 `<turbo-frame>` 标签时，Turbo 自动将此次导航**升级为整页导航**，把响应内容直接渲染到当前窗口。

这是 Turbo Frame 的内置降级机制——无需额外代码，完全由客户端框架自动处理。

---

## 六、聊天消息实时更新链路

### 6.1 消息模型的两种广播回调

[message.rb](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/models/message.rb#L13-L14) 定义了两个核心广播回调：

```ruby
after_create_commit -> { broadcast_append_to chat, target: "messages" }, if: :broadcast?
after_update_commit -> { broadcast_update_to chat }, if: :broadcast?
```

两者的关键区别：

| 回调 | 动作 | 显式 target | 语义 |
|------|------|------------|------|
| `after_create_commit` | `append` | `"messages"` | 在消息列表末尾追加新消息节点 |
| `after_update_commit` | `update` | **无** | 更新已存在的消息节点内容 |

### 6.2 `broadcast_update_to chat` 无显式 target 时的定位机制

`broadcast_update_to chat` 省略了 `target` 参数。turbo-rails 的默认行为是：**以模型的 `dom_id` 作为 target**。

具体流程：

1. `AssistantMessage#append_text!(text)` 调用 `save!` 触发 `after_update_commit`
2. turbo-rails 执行 `broadcast_update_to chat`，由于无显式 target，自动使用 `dom_id(self)` 即 `"assistant_message_42"` 作为 target
3. 同时，turbo-rails 自动渲染该模型的 partial（`assistant_messages/assistant_message`）作为更新内容
4. 客户端收到 Turbo Stream 消息：
   ```xml
   <turbo-stream action="update" target="assistant_message_42">
     <template>... 重新渲染的 partial 内容 ...</template>
   </turbo-stream>
   ```
5. 客户端通过 `document.getElementById("assistant_message_42")` 定位 DOM 节点，执行 `update`（替换 innerHTML）

### 6.3 消息从创建到 DOM 落地的完整链路

#### 阶段一：用户发送消息

1. 用户在聊天表单提交 → `ChatsController#update` 或 `MessagesController#create`
2. 创建 `UserMessage` 记录，触发 `after_create_commit`
3. `broadcast_append_to chat, target: "messages"` 发送 append 动作
4. 客户端在 `<div id="messages">` 末尾追加 `user_messages/_user_message.html.erb` 渲染的 DOM 节点
5. DOM 落点：`<div id="user_message_1">` （由 [dom_id(user_message)](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/views/user_messages/_user_message.html.erb#L3) 生成）

#### 阶段二：AI 开始思考

1. `UserMessage#request_response_later` 调度 `AssistantResponseJob`
2. `Assistant#respond_to` 开始执行，调用 `update_thinking("Thinking...")`
3. `chat.broadcast_update target: "thinking-indicator"` 发送 update 动作
4. 客户端更新 `<div id="thinking-indicator">` 的内容
5. [thinking_indicator.html.erb](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/views/chats/_thinking_indicator.html.erb#L3) 渲染思考提示

#### 阶段三：AI 流式响应（逐 token 追加）

1. LLM 流式回调触发 `assistant_message.append_text!(text)`
2. 每次 `save!` 触发 `after_update_commit`
3. `broadcast_update_to chat` 发送 update 动作，target 自动为 `"assistant_message_42"`
4. 客户端更新 `<div id="assistant_message_42">` 的 innerHTML
5. [assistant_message.html.erb](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/views/assistant_messages/_assistant_message.html.erb#L3) 重新渲染，包含累积的全部文本

**关键细节**：首次 token 到达时，先执行 `stop_thinking`（`broadcast_remove target: "thinking-indicator"` 移除思考指示器），然后创建 `AssistantMessage` 记录。

#### 阶段四：响应完成或出错

- **成功**：`chat.update_latest_response!(response_id)` 更新最新响应 ID
- **失败**：`chat.add_error(e)` 追加错误提示（`broadcast_append target: "messages"`），用户可点击 Retry

### 6.4 消息 DOM ID 映射表

| 模型类 | dom_id 示例 | 视图 Partial | DOM 元素 |
|--------|------------|-------------|----------|
| `UserMessage` | `"user_message_1"` | [user_messages/_user_message.html.erb](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/views/user_messages/_user_message.html.erb#L3) | `<div id="user_message_1">` |
| `AssistantMessage` | `"assistant_message_42"` | [assistant_messages/_assistant_message.html.erb](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/views/assistant_messages/_assistant_message.html.erb#L3) | `<div id="assistant_message_42">` |
| 思考指示器 | `"thinking-indicator"` | [chats/_thinking_indicator.html.erb](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/views/chats/_thinking_indicator.html.erb#L3) | `<div id="thinking-indicator">` |
| 错误提示 | `"chat-error"` | [chats/_error.html.erb](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/views/chats/_error.html.erb#L3) | `<div id="chat-error">` |

### 6.5 无显式 target 的通用规则

turbo-rails 对 `broadcast_update_to` / `broadcast_replace_to` 省略 `target` 时的默认行为：

```
默认 target = dom_id(模型实例)
默认内容   = 渲染模型的 to_partial_path 对应的 partial
```

这意味着：
- `broadcast_update_to chat` 等价于 `broadcast_update_to chat, target: dom_id(self), partial: to_partial_path`
- 对于 `AssistantMessage` 实例：target 为 `"assistant_message_42"`，partial 为 `"assistant_messages/assistant_message"`
- 对于 `UserMessage` 实例：target 为 `"user_message_1"`，partial 为 `"user_messages/user_message"`

这要求视图 partial 的**最外层元素必须设置与 `dom_id` 一致的 `id` 属性**，否则客户端无法定位目标节点。项目中两个消息 partial 均满足此要求。

---

## 七、Morph 刷新机制

### 6.1 配置

```erb
<%= turbo_refreshes_with method: :morph, scroll: :preserve %>
```
[app/views/layouts/shared/_head.html.erb](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/views/layouts/shared/_head.html.erb#L13)

此配置注册了两个关键元标签：
- `<meta name="turbo-refresh-method" content="morph">` — 使用 idiomorph 算法做增量 DOM diff
- `<meta name="turbo-refresh-scroll" content="preserve">` — 刷新时保持滚动位置

### 6.2 Morph 刷新的触发方式

#### 触发方式一：`broadcast_refresh`（WebSocket 推送）

```ruby
account.broadcast_refresh
```
[app/models/account/sync_complete_event.rb](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/models/account/sync_complete_event.rb#L36)

`broadcast_refresh` 是 turbo-rails 提供的模型方法，向订阅了该模型的 ActionCable 频道发送 `<turbo-stream action="refresh">` 消息。客户端收到后执行 `Turbo.session.refresh()`，重新 fetch 当前页面的完整 HTML，然后用 idiomorph 做 morph 渲染。

与 `broadcast_replace` 的区别：`broadcast_replace` 需要指定 `target` 并渲染 partial，是精确的局部替换；`broadcast_refresh` 不需要指定 target，而是让客户端重新获取整页再 morph diff，适合多处联动变化的场景。

#### 触发方式二：组件级 `broadcast_refresh!`

```ruby
class UI::AccountPage < ApplicationComponent
  def broadcast_refresh!
    Turbo::StreamsChannel.broadcast_replace_to(
      broadcast_channel, target: id, renderable: self, layout: false
    )
  end
end
```
[app/components/UI/account_page.rb](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/components/UI/account_page.rb#L21-L23)

注意：虽然方法命名为 `broadcast_refresh!`，但实际实现使用的是 `broadcast_replace_to`，即用组件自身重新渲染的结果替换目标 frame。这与模型层的 `broadcast_refresh`（触发整页 morph）是不同的机制。

ActivityDate 和 ActivityFeed 组件同理：
- [activity_date.rb](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/components/UI/account/activity_date.rb#L23-L29)
- [activity_feed.rb](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/components/UI/account/activity_feed.rb#L18-L25)

#### 触发方式三：Turbo 自动 Page Refresh

当 Turbo Drive 导航到一个新页面时，如果响应 HTML 的 `<head>` 中包含 `turbo-refresh-method: morph`，Turbo 会用 morph 策略渲染而非直接替换 `<body>`。这确保了：
- Stimulus controller 不被销毁重建
- DOM 节点最大程度复用
- 滚动位置保持

### 6.3 Morph 与 replace/update 的本质区别

| 维度 | replace / update | morph refresh |
|------|------------------|---------------|
| 精度 | 精确到单个目标元素 | 整页级增量 diff |
| 服务端职责 | 渲染指定 partial | 无需渲染，客户端自行 fetch |
| 网络开销 | 小（只传输 partial） | 大（传输整页 HTML） |
| 适用场景 | 单一元素更新 | 多处联动变化 |
| Stimulus 影响 | 目标内 controller 重连 | 尽量保留现有 controller |
| 滚动 | 需手动处理 | 自动 preserve |

---

## 七、自定义 Turbo.visit 的触发条件

项目中共有 4 处 `Turbo.visit()` 调用，各有不同的触发条件：

### 7.1 Turbo Stream redirect 动作

```javascript
Turbo.StreamActions.redirect = function () {
  Turbo.visit(this.target);
};
```
[app/javascript/application.js](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/javascript/application.js#L5-L7)

**触发条件**: 服务端返回 `turbo_stream.action(:redirect, url)` 时自动触发

**导航模式**: 默认整页导航（advance），受 morph 配置影响

**使用场景**: 创建/更新资源后需要跳转到新页面时，如 [stream_extensions.rb](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/controllers/concerns/stream_extensions.rb#L18)、[categories_controller.rb](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/controllers/categories_controller.rb#L28)、[holdings_controller.rb](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/controllers/holdings_controller.rb#L21)

### 7.2 SelectableLink Controller —— URL 参数变更导航

```javascript
handleChange(event) {
  const paramName = this.element.name;
  const currentUrl = new URL(window.location.href);
  currentUrl.searchParams.set(paramName, event.target.value);
  Turbo.visit(currentUrl.toString());
}
```
[app/javascript/controllers/selectable_link_controller.js](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/javascript/controllers/selectable_link_controller.js#L13-L19)

**触发条件**: `<select>` 元素的 `change` 事件

**导航模式**: 整页导航（advance），URL 变更 + morph 渲染

**使用场景**: 下拉选择器切换筛选条件（如账户类型、时间范围等）

### 7.3 TradeForm Controller —— Frame 内局部导航

```javascript
async changeType(event) {
  const url = new URL(event.params.url, window.location.origin);
  url.searchParams.set(event.params.key, event.target.value);
  Turbo.visit(url, { frame: "modal" });
}
```
[app/javascript/controllers/trade_form_controller.js](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/javascript/controllers/trade_form_controller.js#L6-L10)

**触发条件**: 交易类型选择器的 `change` 事件

**导航模式**: **Frame 内导航**，仅刷新 `<turbo-frame id="modal">` 的内容，不触发整页导航

**使用场景**: 在模态框中切换交易类型时，重新加载对应类型的表单

### 7.4 Dialog Controller —— 关闭后刷新

```javascript
close() {
  this.element.close();
  if (this.reloadOnCloseValue) {
    Turbo.visit(window.location.href);
  }
}
```
[app/components/DS/dialog_controller.js](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/components/DS/dialog_controller.js#L26-L31)

**触发条件**: `<dialog>` 关闭 + `reload-on-close` 属性为 `true`

**导航模式**: 整页导航（advance），对当前 URL 重新 fetch + morph 渲染

**使用场景**: 模态框关闭后需要刷新底层页面数据时（如编辑后）

### 7.5 Turbo.visit 各调用方式对比

| 调用方式 | `frame` 参数 | 导航范围 | URL 变更 |
|----------|-------------|----------|----------|
| `Turbo.visit(url)` | 无 | 整页 | 是 |
| `Turbo.visit(url, { frame: "modal" })` | `"modal"` | 仅 modal frame | 否 |
| `Turbo.visit(window.location.href)` | 无 | 整页（刷新当前页） | 否（相同 URL） |

---

## 八、强制 Reload 机制

### 8.1 `data-turbo-track="reload"`

```erb
<%= stylesheet_link_tag "tailwind", "data-turbo-track": "reload" %>
```
[app/views/layouts/shared/_head.html.erb](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/views/layouts/shared/_head.html.erb#L7)

`data-turbo-track="reload"` 标记的 `<link>` 或 `<script>` 元素，在 Turbo 导航时会被检查。如果新页面中对应元素的 `href` 或 `src` 属性发生变化，Turbo 会强制执行一次完整的页面 reload（而非 morph），确保新资源被加载。

本项目将 Tailwind CSS 样式表标记为 `track: reload`，确保部署新版本后用户能获取到更新后的样式。

### 8.2 `window.location.reload()`

```erb
onclick: "window.location.reload()"
```
[app/views/pages/redis_configuration_error.html.erb](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/views/pages/redis_configuration_error.html.erb#L53)

这是唯一的 `location.reload()` 调用，出现在 Redis 配置错误页面，不属于核心业务流程。

### 8.3 Dialog 关闭后刷新

```javascript
if (this.reloadOnCloseValue) {
  Turbo.visit(window.location.href);
}
```
[app/components/DS/dialog_controller.js](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/components/DS/dialog_controller.js#L28-L30)

通过 `Turbo.visit(window.location.href)` 实现的"伪 reload"——重新 fetch 当前页面并用 morph 渲染。与 `location.reload()` 的区别在于：
- 走 Turbo 的 fetch + morph 流程，不会出现白屏闪烁
- Stimulus controller 尽量复用，不会全部重建
- 滚动位置自动保持

---

## 九、降级到整页跳转的触发场景（完整版）

### 9.1 JS 环境不可用

**触发条件**: 浏览器禁用 JavaScript、Turbo 库加载失败、旧版浏览器

**降级行为**: 所有请求走传统 HTTP，服务器返回完整 HTML，浏览器整页刷新

### 9.2 `turbo_frame: "_top"` 显式顶层导航

**触发条件**: 链接或表单声明 `data-turbo-frame="_top"`

**降级行为**: 突破当前 Frame 上下文，以整页方式导航（morph 渲染）

**项目中共 33 处使用**

### 9.3 Turbo Stream `action(:redirect)` 自定义重定向

**触发条件**: 服务端返回 `turbo_stream.action(:redirect, url)`

**降级行为**: 客户端执行 `Turbo.visit(url)`，触发整页导航

**使用场景**: 资源创建/更新后需跳转（[stream_extensions.rb](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/controllers/concerns/stream_extensions.rb)、[categories_controller.rb](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/controllers/categories_controller.rb#L28)、[family_merchants_controller.rb](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/controllers/family_merchants_controller.rb#L22)、[holdings_controller.rb](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/controllers/holdings_controller.rb#L21)）

### 9.4 Turbo Frame 响应不匹配

**触发条件**: 请求由 `<turbo-frame>` 发起，响应中无匹配 id 的 `<turbo-frame>`

**降级行为**: 自动升级为整页导航

### 9.5 表单验证失败（422 Unprocessable Entity）

**触发条件**: 控制器返回 `status: :unprocessable_entity`

**降级行为**: Turbo 将响应内容渲染到当前页面（替换 `<body>` 或 Frame 内容），**不执行跳转**

### 9.6 非 Turbo 格式的请求/响应

**触发条件**: 直接访问 URL、页面刷新（F5）、无 Turbo 请求头

**降级行为**: 传统整页加载

### 9.7 特殊状态码

| 状态码 | 行为 |
|--------|------|
| **3xx 重定向** | Turbo 跟随重定向，结果取决于目标 |
| **401 Unauthorized** | 触发登录页整页跳转 |
| **404 / 500** | 整页错误页面 |

### 9.8 `data-turbo-track="reload"` 触发强制 reload

**触发条件**: Turbo 导航时检测到标记元素的 `href`/`src` 发生变化

**降级行为**: 从 morph 渲染降级为完整页面 reload（`location.reload()`）

### 9.9 Turbo Stream 目标元素不存在

**触发条件**: 推送消息的 `target` 在当前页面不存在

**降级行为**: **静默忽略**，不触发整页跳转，但局部刷新失效

### 9.10 跨域请求

**触发条件**: 导航到不同域名

**降级行为**: 浏览器传统导航

---

## 十、典型流程分析

### 10.1 成功的局部刷新流程（交易更新）

1. 用户在 drawer 中编辑交易信息，提交表单
2. Turbo 拦截表单提交，AJAX 发送请求（`Accept: text/vnd.turbo-stream.html`）
3. 控制器返回 Turbo Stream 响应：
   ```ruby
   render turbo_stream: [
     turbo_stream.replace(dom_id(@entry, :header), partial: "..."),
     turbo_stream.replace(@entry),
     *flash_notification_stream_items
   ]
   ```
4. 客户端依次执行：
   - 通过 `id="header_entry_123"` 找到 `<header>` → 整体替换
   - 通过 `id="entry_123"` 找到 `<turbo-frame>` → 整体替换
   - 向 `id="notification-tray"` 追加通知

### 10.2 Morph 刷新流程（账户同步完成）

1. 后台同步完成后调用 `account.broadcast_sync_complete`
2. `Account::SyncCompleteEvent#broadcast` 执行：
   - `broadcast_replace_to` 更新侧边栏账户行和分组
   - `account.broadcast_refresh` 触发账户详情页 morph 刷新
3. 客户端收到 `<turbo-stream action="refresh">` → `Turbo.session.refresh()`
4. 客户端 fetch 当前页面完整 HTML → idiomorph diff → 增量更新 DOM → 保持滚动位置

### 10.3 Frame 内导航流程（模态框切换交易类型）

1. 用户在 modal 中选择交易类型（触发 `change` 事件）
2. `trade_form_controller.js` 调用 `Turbo.visit(url, { frame: "modal" })`
3. Turbo 仅对 `<turbo-frame id="modal">` 发起 fetch 请求
4. 服务端返回包含 `<turbo-frame id="modal">` 的 HTML
5. 仅 modal frame 内容被替换，页面其余部分不变

### 10.4 顶层导航流程（转账表单提交）

1. 用户在 drawer 中提交转账表单（`data-turbo-frame="_top"`）
2. Turbo 对整页发起导航请求
3. 控制器返回 Turbo Stream 重定向：`turbo_stream.action(:redirect, url)`
4. 客户端执行 `Turbo.visit(url)` → 整页导航 → morph 渲染

---

## 十一、总结

### 11.1 四种刷新/导航模式对比

| 模式 | 触发方式 | 作用范围 | 服务端渲染量 | Stimulus 影响 |
|------|---------|----------|-------------|--------------|
| **Turbo Stream replace/update** | 控制器响应 / WebSocket 广播 | 单个目标元素 | 小（partial） | 目标内 controller 重连 |
| **Morph Refresh** | `broadcast_refresh` / Turbo 导航 | 整页（增量 diff） | 大（整页 HTML） | 尽量保留 |
| **Frame 导航** | `turbo_frame: "modal"/"drawer"` | 单个 frame | 中（frame 区域） | frame 内 controller 重连 |
| **整页跳转** | `_top` / `action(:redirect)` / 不匹配 | 整页 | 大（整页 HTML） | 全部重建（morph 时尽量保留） |

### 11.2 降级场景分类

| 类别 | 触发场景 | 降级行为 |
|------|----------|----------|
| **显式顶层导航** | `turbo_frame: "_top"` | 整页导航（morph） |
| **Stream 重定向** | `action(:redirect)` | Turbo.visit 整页导航 |
| **Frame 不匹配** | 响应无匹配 Turbo Frame | 自动升级整页导航 |
| **表单验证失败** | 422 Unprocessable Entity | 渲染错误表单（不跳转） |
| **资源变更** | `data-turbo-track="reload"` | 强制完整 reload |
| **JS 环境** | JS 禁用 / 加载失败 | 传统整页请求 |
| **非 Turbo 请求** | 直接访问 / 刷新 | 传统页面加载 |
| **错误状态码** | 401/404/500 | 整页错误页面 |
| **目标不存在** | Turbo Stream target 未找到 | 静默忽略 |
| **跨域请求** | 不同域名 | 浏览器传统导航 |

---

## 十二、核心代码索引

| 模块 | 文件路径 |
|------|----------|
| Turbo 客户端入口 + redirect 动作 | [application.js](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/javascript/application.js) |
| Morph 刷新配置 | [_head.html.erb](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/views/layouts/shared/_head.html.erb#L13) |
| Stream 重定向扩展 | [stream_extensions.rb](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/controllers/concerns/stream_extensions.rb) |
| 通知流处理 | [notifiable.rb](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/controllers/concerns/notifiable.rb) |
| 模型广播（Message） | [message.rb](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/models/message.rb) |
| 模型广播（Chat） | [chat.rb](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/models/chat.rb) |
| 模型广播（Assistant） | [broadcastable.rb](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/models/assistant/broadcastable.rb) |
| 同步完成广播（Account） | [sync_complete_event.rb](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/models/account/sync_complete_event.rb) |
| 同步完成广播（Family） | [sync_complete_event.rb](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/models/family/sync_complete_event.rb) |
| 同步完成广播（PlaidItem） | [sync_complete_event.rb](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/models/plaid_item/sync_complete_event.rb) |
| 组件广播（AccountPage） | [account_page.rb](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/components/UI/account_page.rb) |
| 组件广播（ActivityFeed） | [activity_feed.rb](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/components/UI/account/activity_feed.rb) |
| 组件广播（ActivityDate） | [activity_date.rb](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/components/UI/account/activity_date.rb) |
| 转账更新流视图 | [update.turbo_stream.erb](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/views/transfers/update.turbo_stream.erb) |
| 预算分类更新流视图 | [update.turbo_stream.erb](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/views/budget_categories/update.turbo_stream.erb) |
| API Key 创建流视图 | [created.turbo_stream.erb](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/views/settings/api_keys/created.turbo_stream.erb) |
| SelectableLink Controller | [selectable_link_controller.js](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/javascript/controllers/selectable_link_controller.js) |
| TradeForm Controller | [trade_form_controller.js](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/javascript/controllers/trade_form_controller.js) |
| Dialog Controller | [dialog_controller.js](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/components/DS/dialog_controller.js) |
| PreserveScroll Controller | [preserve_scroll_controller.js](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/javascript/controllers/preserve_scroll_controller.js) |
| TurboFrameTimeout Controller | [turbo_frame_timeout_controller.js](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/javascript/controllers/turbo_frame_timeout_controller.js) |
| 布局骨架（modal/drawer/subscription） | [_htmldoc.html.erb](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/views/layouts/shared/_htmldoc.html.erb) |
| 主布局（chat container/turbo_permanent） | [application.html.erb](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/views/layouts/application.html.erb) |
| 聊天 Helper（chat_frame） | [chats_helper.rb](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/helpers/chats_helper.rb) |
| Syncable concern | [syncable.rb](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/models/concerns/syncable.rb) |
| 交易控制器 | [transactions_controller.rb](file:///d:/fz/0601-1/solo-dogfeeding/code/6-maybe/app/controllers/transactions_controller.rb) |
