# 家庭账务助手聊天会话流分析报告

## 一、整体架构概览

家庭账务助手的聊天系统采用 **"同步接收 + 异步处理 + 流式推送"** 的架构模式：

- **同步接收**：用户消息通过 HTTP 请求同步写入数据库
- **异步处理**：AI 响应生成通过后台任务队列（Sidekiq）异步执行
- **流式推送**：通过 Turbo Streams（基于 Action Cable/WebSocket）实时推送更新到前端

```
用户前端 ──HTTP──> 控制器 ──DB写入──> 消息模型 ──入队──> Sidekiq任务
                                          │
                                          └──Turbo Stream──> 前端实时更新
                                          │
                          后台任务处理 <──┘
                               │
                               ├── LLM API 调用（流式）
                               ├── 工具函数调用
                               └── 结果持久化 + Turbo Stream 推送
```

---

## 二、用户消息入队流程

### 2.1 消息创建入口总览

用户发送消息有 **四条独立入口**，分为两类：

| 入口类型 | 路由 | 控制器 | 适用场景 |
|---------|------|--------|---------|
| 网页入口 | `POST /chats` | `ChatsController#create` | 网页端创建新聊天 |
| 网页入口 | `POST /chats/:chat_id/messages` | `MessagesController#create` | 网页端追加消息 |
| API 入口 | `POST /api/v1/chats` | `Api::V1::ChatsController#create` | API 创建新聊天（含首条消息） |
| API 入口 | `POST /api/v1/chats/:chat_id/messages` | `Api::V1::MessagesController#create` | API 追加消息 |

---

### 2.2 网页入口分析

**网页入口1：创建新聊天** (`chats#create`)

`app/controllers/chats_controller.rb:19-23`
```ruby
def create
  @chat = Current.user.chats.start!(chat_params[:content], model: chat_params[:ai_model])
  set_last_viewed_chat(@chat)
  redirect_to chat_path(@chat, thinking: true)
end
```

**网页入口2：在现有聊天中追加消息** (`messages#create`)

`app/controllers/messages_controller.rb:6-13`
```ruby
def create
  @message = UserMessage.create!(
    chat: @chat,
    content: message_params[:content],
    ai_model: message_params[:ai_model]
  )
  redirect_to chat_path(@chat, thinking: true)
end
```

**网页入口入队机制**：完全依赖 `UserMessage` 的 `after_create_commit` 回调自动触发，控制器不手动入队。

---

### 2.3 API 入口分析

**API 认证机制**：

`app/controllers/api/v1/base_controller.rb:41-104`

API 支持两种认证方式：
1. **OAuth 2.0 Bearer Token**：通过 Doorkeeper 管理，支持 `read` / `read_write` 作用域
2. **API Key**：通过 `X-Api-Key` 请求头，支持速率限制

认证成功后通过 `setup_current_context_for_api` 设置 `Current.session` 和 `Current.user`。

**API 入口1：创建新聊天** (`api/v1/chats#create`)

`app/controllers/api/v1/chats_controller.rb:19-43`
```ruby
def create
  @chat = Current.user.chats.build(title: chat_params[:title])

  if @chat.save
    if chat_params[:message].present?
      @message = @chat.messages.build(
        content: chat_params[:message],
        type: "UserMessage",           # 注意：创建的是 UserMessage 类型
        ai_model: chat_params[:model] || "gpt-4"
      )

      if @message.save
        AssistantResponseJob.perform_later(@message)  # 手动入队
        render :show, status: :created
      else
        # ... 错误处理
      end
    else
      render :show, status: :created
    end
  else
    # ... 错误处理
  end
end
```

**API 入口2：在现有聊天中追加消息** (`api/v1/messages#create`)

`app/controllers/api/v1/messages_controller.rb:8-21`
```ruby
def create
  @message = @chat.messages.build(
    content: message_params[:content],
    type: "UserMessage",             # 注意：创建的是 UserMessage 类型
    ai_model: message_params[:model] || "gpt-4"
  )

  if @message.save
    AssistantResponseJob.perform_later(@message)  # 手动入队
    render :show, status: :created
  else
    # ... 错误处理
  end
end
```

**API 入口3：重试消息** (`api/v1/messages#retry`)

`app/controllers/api/v1/messages_controller.rb:23-38`
```ruby
def retry
  last_message = @chat.messages.ordered.last

  if last_message&.type == "AssistantMessage"
    new_message = @chat.messages.create!(
      type: "AssistantMessage",      # 注意：创建的是 AssistantMessage 类型
      content: "",
      ai_model: last_message.ai_model
    )

    AssistantResponseJob.perform_later(new_message)  # 手动入队
    render json: { message: "Retry initiated", message_id: new_message.id }, status: :accepted
  else
    # ... 错误处理
  end
end
```

---

### 2.4 消息自动入队机制（回调）

`app/models/user_message.rb:4-12`
```ruby
class UserMessage < Message
  after_create_commit :request_response_later

  def request_response_later
    chat.ask_assistant_later(self)
  end
end
```

`app/models/chat.rb:59-62`
```ruby
def ask_assistant_later(message)
  clear_error
  AssistantResponseJob.perform_later(message)
end
```

**关键点**：
- `after_create_commit` 回调在消息持久化到数据库后触发
- 仅 `UserMessage` 有此回调，`AssistantMessage` 没有
- 入队前调用 `clear_error` 清除之前的错误状态

---

### 2.5 两类入队机制对比

| 对比项 | 网页入口 | API 入口 |
|-------|---------|---------|
| 消息类型 | `UserMessage` | `UserMessage`（创建/追加）/ `AssistantMessage`（重试） |
| 入队触发方式 | 仅依赖 `after_create_commit` 回调 | `after_create_commit` 回调 + **控制器手动调用** |
| 入队次数 | 1 次 | **2 次**（`UserMessage` 场景） / 1 次（`AssistantMessage` 重试场景） |
| 认证方式 | Session Cookie | OAuth Token / API Key |
| 响应格式 | HTML 重定向 | JSON |
| 作用域检查 | 无 | 需要 `read_write` 作用域 |

---

### 2.6 重复投递风险分析（API 入口 Bug）

**问题描述**：

API 入口创建 `UserMessage` 时存在 **重复入队** 问题：

```
@message.save
  ↓
after_create_commit 回调触发
  → request_response_later
    → chat.ask_assistant_later(self)
      → AssistantResponseJob.perform_later(@message)  // 第 1 次入队
  ↓
控制器继续执行
  → AssistantResponseJob.perform_later(@message)      // 第 2 次入队
```

**受影响的 API 接口**：
1. `POST /api/v1/chats` - 创建聊天并包含首条消息时
2. `POST /api/v1/chats/:chat_id/messages` - 追加消息时

**不受影响的接口**：
- `POST /api/v1/chats/:chat_id/messages/retry` - 创建的是 `AssistantMessage`，没有回调，仅入队 1 次
- 所有网页入口 - 仅依赖回调，无手动入队

**影响范围**：

| 影响类型 | 具体表现 |
|---------|---------|
| **用户体验** | 同一条用户消息会收到两条重复的 AI 回复 |
| **资源浪费** | LLM API 调用次数翻倍，增加成本 |
| **任务队列压力** | 任务数量翻倍，可能导致队列阻塞 |
| **数据一致性** | 聊天历史中出现重复的助手消息 |
| **副作用放大** | 如果 AI 响应包含工具调用（如修改交易分类），可能导致重复执行 |

**根本原因**：

API 控制器设计时忽略了 `UserMessage` 已有的 `after_create_commit` 回调机制，导致回调自动入队和手动入队同时执行。

---

### 2.7 任务队列配置

`app/jobs/assistant_response_job.rb:1-6`
```ruby
class AssistantResponseJob < ApplicationJob
  queue_as :high_priority

  def perform(message)
    message.request_response
  end
end
```

- 任务进入 `high_priority` 队列优先处理
- 任务携带完整的 `message` 对象（通过 Global ID 序列化）
- 执行时调用 `message.request_response` 触发响应生成

---

## 三、后台任务处理流程

### 3.1 任务执行链路

```
AssistantResponseJob.perform_later(message)
  ↓
Sidekiq 调度执行
  ↓
message.request_response
  ↓
chat.ask_assistant(self)
  ↓
assistant.respond_to(message)
```

### 3.2 助手响应生成核心

`app/models/assistant.rb:19-63`

响应生成分为以下步骤：

1. **创建空的助手消息**：
```ruby
assistant_message = AssistantMessage.new(
  chat: chat,
  content: "",
  ai_model: message.ai_model
)
```

2. **初始化响应器**：
```ruby
responder = Assistant::Responder.new(
  message: message,
  instructions: instructions,
  function_tool_caller: function_tool_caller,
  llm: get_model_provider(message.ai_model)
)
```

3. **注册流式回调**：
```ruby
responder.on(:output_text) do |text|
  if assistant_message.content.blank?
    stop_thinking
    Chat.transaction do
      assistant_message.append_text!(text)
      chat.update_latest_response!(latest_response_id)
    end
  else
    assistant_message.append_text!(text)
  end
end

responder.on(:response) do |data|
  update_thinking("Analyzing your data...")
  # 处理工具调用...
end
```

4. **触发响应**：
```ruby
responder.respond(previous_response_id: latest_response_id)
```

### 3.3 流式响应处理

`app/models/assistant/responder.rb:13-31`

响应器通过事件驱动模式处理 LLM 流式输出：

```ruby
def respond(previous_response_id: nil)
  streamer = proc do |chunk|
    case chunk.type
    when "output_text"
      emit(:output_text, chunk.data)
    when "response"
      response = chunk.data
      if response.function_requests.any?
        handle_follow_up_response(response)
      else
        emit(:response, { id: response.id })
      end
    end
  end

  get_llm_response(streamer: streamer, previous_response_id: previous_response_id)
end
```

### 3.4 LLM 提供商集成

`app/models/provider/openai.rb:41-82`

OpenAI 提供商负责实际的 API 调用和流式解析：

```ruby
def chat_response(prompt, model:, ...)
  stream_proxy = if streamer.present?
    proc do |chunk|
      parsed_chunk = ChatStreamParser.new(chunk).parsed
      unless parsed_chunk.nil?
        streamer.call(parsed_chunk)
        collected_chunks << parsed_chunk
      end
    end
  end

  raw_response = client.responses.create(parameters: {
    model: model,
    input: chat_config.build_input(prompt),
    stream: stream_proxy
  })
end
```

### 3.5 工具调用处理

当 LLM 请求工具调用时，`handle_follow_up_response` 会：

1. 执行工具函数获取结果
2. 将工具调用记录到消息中
3. 发送包含工具结果的 follow-up 请求给 LLM
4. 获取最终的自然语言响应

---

## 四、流式更新前端展示

### 4.1 Turbo Streams 实时通信

系统使用 **Turbo Streams + Action Cable** 实现实时更新，无需自定义 WebSocket 通道。

**服务端广播**：

`app/models/message.rb:13-14`
```ruby
after_create_commit -> { broadcast_append_to chat, target: "messages" }, if: :broadcast?
after_update_commit -> { broadcast_update_to chat }, if: :broadcast?
```

`app/models/chat.rb:45-53`
```ruby
def add_error(e)
  update! error: e.to_json
  broadcast_append target: "messages", partial: "chats/error", locals: { chat: self }
end

def clear_error
  update! error: nil
  broadcast_remove target: "chat-error"
end
```

`app/models/assistant/broadcastable.rb:5-11`
```ruby
def update_thinking(thought)
  chat.broadcast_update target: "thinking-indicator", 
    partial: "chats/thinking_indicator", 
    locals: { chat: chat, message: thought }
end

def stop_thinking
  chat.broadcast_remove target: "thinking-indicator"
end
```

**前端订阅**：

`app/views/chats/show.html.erb:2-3`
```erb
<%= turbo_frame_tag chat_frame do %>
  <%= turbo_stream_from @chat %>
```

### 4.2 流式更新时序

```
1. 思考状态显示
   broadcast_update → "Thinking ..."
   
2. 首个文本块到达
   stop_thinking (broadcast_remove)
   创建 AssistantMessage 记录
   broadcast_append_to chat, target: "messages"
   
3. 后续文本块到达
   assistant_message.append_text!(text)
   broadcast_update_to chat (触发消息局部更新)
   
4. 响应完成
   无额外操作，消息已完整
```

### 4.3 前端自动滚动

`app/javascript/controllers/chat_controller.js:43-59`

使用 MutationObserver 监听 DOM 变化，自动滚动到底部：

```javascript
#configureAutoScroll() {
  this.messagesObserver = new MutationObserver((_mutations) => {
    if (this.hasMessagesTarget) {
      this.#scrollToBottom();
    }
  });

  this.messagesObserver.observe(this.element, {
    childList: true,
    subtree: true,
  });
}

#scrollToBottom = () => {
  this.messagesTarget.scrollTop = this.messagesTarget.scrollHeight;
};
```

### 4.4 Action Cable 配置

`config/cable.yml`
```yaml
development:
  adapter: async

production:
  adapter: redis
  url: <%= ENV.fetch("REDIS_URL") { "redis://localhost:6379/1" } %>
  channel_prefix: maybe_production
```

- 开发环境使用内存异步适配器
- 生产环境使用 Redis 适配器，支持多进程/多服务器部署

---

## 五、多用户、多会话隔离机制

### 5.1 数据模型层级

```
Family (家庭)
  └── User (用户) ── 1:N ── Chat (会话) ── 1:N ── Message (消息)
```

### 5.2 基于 Current 模式的用户上下文

`app/models/current.rb:1-18`
```ruby
class Current < ActiveSupport::CurrentAttributes
  attribute :user_agent, :ip_address
  attribute :session

  delegate :family, to: :user, allow_nil: true

  def user
    impersonated_user || session&.user
  end
end
```

`Current` 是 Rails 提供的线程全局存储，确保：
- 同一请求周期内用户上下文一致
- 支持后台任务中访问当前用户（通过 Job 序列化）

### 5.3 认证与权限隔离

`app/controllers/concerns/authentication.rb:18-28`
```ruby
def authenticate_user!
  if session_record = find_session_by_cookie
    Current.session = session_record
  else
    redirect_to new_session_url
  end
end
```

所有控制器（除明确跳过认证的外）都会先执行 `authenticate_user!`，确保：
- 未登录用户无法访问任何资源
- `Current.user` 始终为有效用户

### 5.4 数据查询隔离

**所有查询都通过 `Current.user` 关联查询**，避免直接访问模型：

`app/controllers/chats_controller.rb:51`
```ruby
def set_chat
  @chat = Current.user.chats.find(params[:id])
end
```

`app/controllers/messages_controller.rb:18`
```ruby
def set_chat
  @chat = Current.user.chats.find(params[:chat_id])
end
```

`app/controllers/chats_controller.rb:8`
```ruby
def index
  @chats = Current.user.chats.order(created_at: :desc)
end
```

### 5.5 Turbo Streams 频道隔离

`turbo_stream_from @chat` 会为每个聊天创建独立的 WebSocket 频道：
- 频道名称基于 `chat_gid`（Global ID），如 `gid://maybe/Chat/123`
- 只有订阅了该聊天的前端才会收到更新
- 即使不同用户打开相同页面，也无法接收他人聊天的更新

### 5.6 数据库层面隔离

`db/schema.rb:174-183`
```ruby
create_table "chats", id: :uuid, ... do |t|
  t.uuid "user_id", null: false  # 每个 chat 明确归属 user
  t.index ["user_id"], name: "index_chats_on_user_id"
end
```

`db/schema.rb:443-455`
```ruby
create_table "messages", id: :uuid, ... do |t|
  t.uuid "chat_id", null: false  # 每个 message 明确归属 chat
  t.index ["chat_id"], name: "index_messages_on_chat_id"
end
```

### 5.7 多会话隔离总结

隔离维度 | 实现方式
--- | ---
**用户隔离** | `Current.user.chats.find(params[:id])` 确保只能访问自己的聊天
**会话隔离** | 每个 `Chat` 有独立的 `id`，消息通过 `chat_id` 关联
**实时通信隔离** | `turbo_stream_from @chat` 为每个聊天创建独立频道
**后台任务隔离** | 任务携带 `message` 对象，内含完整关联链（message → chat → user）
**数据库隔离** | 外键约束 + 索引，确保数据物理隔离

---

## 六、错误处理机制

### 6.1 异常捕获

`app/models/assistant.rb:59-63`
```ruby
responder.respond(previous_response_id: latest_response_id)
rescue => e
  stop_thinking
  chat.add_error(e)
end
```

### 6.2 错误展示与重试

错误通过 Turbo Stream 广播到前端：
- 显示错误消息
- 提供 "Retry" 按钮

`app/controllers/chats_controller.rb:44-47`
```ruby
def retry
  @chat.retry_last_message!
  redirect_to chat_path(@chat, thinking: true)
end
```

`app/models/chat.rb:30-39`
```ruby
def retry_last_message!
  update!(error: nil)
  last_message = conversation_messages.ordered.last
  if last_message.present? && last_message.role == "user"
    ask_assistant_later(last_message)
  end
end
```

---

## 七、关键技术要点总结

### 7.1 设计亮点

1. **回调驱动入队**：使用 `after_create_commit` 自动触发，避免遗漏
2. **事件驱动流式**：Responder 使用 `on(event, &block)` 模式，解耦流式处理
3. **零自定义 WebSocket**：完全基于 Turbo Streams，无需手写通道代码
4. **多层级隔离**：从代码查询到数据库外键，确保数据安全
5. **优雅降级**：错误时显示友好提示并支持重试

### 7.2 关键文件速查

功能 | 文件路径
--- | ---
消息入队 | `app/models/user_message.rb`
任务定义 | `app/jobs/assistant_response_job.rb`
响应生成 | `app/models/assistant.rb`, `app/models/assistant/responder.rb`
流式广播 | `app/models/message.rb`, `app/models/assistant/broadcastable.rb`
前端控制器 | `app/javascript/controllers/chat_controller.js`
多用户隔离 | `app/models/current.rb`, `app/controllers/concerns/authentication.rb`
聊天页面 | `app/views/chats/show.html.erb`

### 7.3 数据流完整路径

```
用户输入
  ↓ [HTTP POST]
messages#create / chats#create
  ↓
UserMessage.create!
  ↓ [after_create_commit]
user_message.request_response_later
  ↓
chat.ask_assistant_later(message)
  ↓
AssistantResponseJob.perform_later(message)
  ↓ [Sidekiq 异步执行]
AssistantResponseJob#perform
  ↓
message.request_response
  ↓
chat.ask_assistant(message)
  ↓
assistant.respond_to(message)
  ├─ 创建空 AssistantMessage
  ├─ 注册 :output_text / :response 回调
  └─ responder.respond
      ├─ llm.chat_response (流式调用 OpenAI API)
      │   └─ stream_proxy 解析每个 chunk
      │       └─ 触发 responder 的 :output_text 事件
      │           ├─ 首个 chunk: 创建消息 + broadcast_append
      │           └─ 后续 chunk: 追加文本 + broadcast_update
      └─ 响应完成
          └─ (可选) 处理工具调用 follow-up
```
