# 后台任务失败重试逻辑与用户提示关联分析

## 1. 基础重试机制（Sidekiq + ActiveJob）

### 1.1 全局 Job 基类配置

[application_job.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/22-maybe/app/jobs/application_job.rb)

```ruby
class ApplicationJob < ActiveJob::Base
  retry_on ActiveRecord::Deadlocked
  discard_on ActiveJob::DeserializationError
  queue_as :low_priority # default queue
end
```

**全局重试策略：**
- `retry_on ActiveRecord::Deadlocked`：数据库死锁时自动重试（Sidekiq 默认指数退避算法）
- `discard_on ActiveJob::DeserializationError`：反序列化失败时直接丢弃（如关联记录已删除），不重试

### 1.2 Sidekiq 队列优先级配置

[sidekiq.yml](file:///d:/fz/0601-2/solo-dogfeeding/code/22-maybe/config/sidekiq.yml)

| 队列名称 | 优先级权重 | 典型用途 |
|---------|----------|--------|
| scheduled | 10 | 定时任务（每日市场数据同步等） |
| high_priority | 4 | Import、Sync、AssistantResponse |
| medium_priority | 2 | Rule、RevertImport、AutoCategorize |
| low_priority | 1 | Destroy、FamilyReset、默认队列 |
| default | 1 | FamilyDataExport |

### 1.3 定时清理机制

[sync_cleaner_job.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/22-maybe/app/jobs/sync_cleaner_job.rb)
- `Sync.clean` 定期清理超过 24 小时未完成的 sync，标记为 `stale`
- 防止因服务重启、代码部署导致的"僵尸"同步任务

---

## 2. 核心后台任务类型及错误处理

### 2.1 ImportJob（导入任务）- high_priority

**执行流程：** [import_job.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/22-maybe/app/jobs/import_job.rb) → `import.publish`

**错误处理位置：** [import.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/22-maybe/app/models/import.rb#L65-L75)

```ruby
def publish
  import!
  family.sync_later
  update! status: :complete
rescue => error
  update! status: :failed, error: error.message  # 异常捕获+状态持久化+错误信息存储
end
```

**Import 状态机：** [import.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/22-maybe/app/models/import.rb#L24-L31)

| 状态 | 说明 | 触发时机 |
|-----|------|--------|
| pending | 待处理 | 默认初始状态，revert 成功后回到此状态 |
| importing | 导入中 | `publish_later` 调用时 |
| complete | 完成 | publish 方法成功执行 |
| failed | 失败 | publish 方法异常（错误信息存 error 字段） |
| reverting | 回滚中 | `revert_later` 调用时 |
| revert_failed | 回滚失败 | revert 方法异常 |

**回滚错误处理：** [import.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/22-maybe/app/models/import.rb#L85-L96)

```ruby
def revert
  Import.transaction do
    accounts.destroy_all
    entries.destroy_all
  end
  family.sync_later
  update! status: :pending
rescue => error
  update! status: :revert_failed, error: error.message
end
```

---

### 2.2 SyncJob（同步任务）- high_priority

**执行流程：** [sync_job.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/22-maybe/app/jobs/sync_job.rb) → `sync.perform`

**状态机（AASM）：** [sync.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/22-maybe/app/models/sync.rb#L27-L52)

| 状态 | 说明 | 事件 |
|-----|------|------|
| pending | 待同步（初始） | - |
| syncing | 同步中 | `start!` |
| completed | 同步完成 | `complete!` |
| failed | 同步失败 | `fail!`（异常时触发，错误存 error 字段） |
| stale | 超时失效 | `mark_stale!`（超过 24h 未完成） |

**错误处理核心逻辑：** [sync.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/22-maybe/app/models/sync.rb#L60-L79)

```ruby
def perform
  unless may_start?       # 状态校验：防止重复/无效启动
    Rails.logger.warn(...)
    return
  end

  start!

  begin
    syncable.perform_sync(self)
  rescue => e
    fail!                 # 状态流转为 failed
    update(error: e.message)  # 记录错误信息
    report_error(e)       # 上报 Sentry
  ensure
    finalize_if_all_children_finalized  # 父子同步联动
  end
end
```

**父子同步联动机制：** [sync.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/22-maybe/app/models/sync.rb#L83-L104)
- 支持父子层级结构（如 Family → Account 的同步）
- 父同步完成条件：所有子同步均已结束（`all_children_finalized?`）
- 子同步失败 → 父同步自动标记为 `failed`（`has_failed_children?`）
- 完成后触发 `perform_post_sync` + `broadcast_sync_complete`（Turbo Stream 广播）

**Syncable Concern（同步能力混入）：** [syncable.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/22-maybe/app/models/concerns/syncable.rb)

核心方法：
- `sync_later`：创建同步任务，若已有进行中的同步则扩展时间窗口而非重复创建
- `syncing?`：`syncs.visible.any?`（5 分钟内未完成的同步视为可见）
- `sync_error`：获取最新同步的错误信息（包括子同步的错误）
- `last_synced_at`：最新同步完成时间
- `broadcast_sync_complete`：同步完成后的 Turbo Stream 广播入口

---

### 2.3 AssistantResponseJob（AI 助手响应）- high_priority

**执行流程：** [assistant_response_job.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/22-maybe/app/jobs/assistant_response_job.rb) → `message.request_response`

**Chat 模型错误处理：** [chat.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/22-maybe/app/models/chat.rb#L45-L53)

```ruby
def add_error(e)
  update! error: e.to_json                      # 序列化存储错误
  broadcast_append target: "messages",          # Turbo Stream 实时追加错误UI
    partial: "chats/error", locals: { chat: self }
end

def clear_error
  update! error: nil
  broadcast_remove target: "chat-error"         # 移除错误UI
end
```

**重试机制：** [chat.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/22-maybe/app/models/chat.rb#L30-L39)

```ruby
def retry_last_message!
  update!(error: nil)                            # 清除错误
  last_message = conversation_messages.ordered.last
  if last_message.present? && last_message.role == "user"
    ask_assistant_later(last_message)            # 重新入队
  end
end
```

---

### 2.4 FamilyDataExportJob（数据导出）- default

**错误处理：** [family_data_export_job.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/22-maybe/app/jobs/family_data_export_job.rb#L5-L21)

```ruby
def perform(family_export)
  family_export.update!(status: :processing)
  # ... 导出逻辑 ...
  family_export.update!(status: :completed)
rescue => e
  Rails.logger.error "Family export failed: #{e.message}"
  Rails.logger.error e.backtrace.join("\n")
  family_export.update!(status: :failed)        # 无详细错误信息存储
end
```

**FamilyExport 状态：** [family_export.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/22-maybe/app/models/family_export.rb#L6-L11)

| 状态 | 说明 |
|-----|------|
| pending | 等待处理（默认） |
| processing | 导出中 |
| completed | 完成，`downloadable?` 判定可下载 |
| failed | 失败 |

---

### 2.5 DestroyJob（资源删除）- low_priority

**错误处理：** [destroy_job.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/22-maybe/app/jobs/destroy_job.rb#L4-L8)

```ruby
def perform(model)
  model.destroy
rescue => e
  model.update!(scheduled_for_deletion: false)  # 重置状态，允许用户重试
end
```

**Account 模型层兜底处理：** [account.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/22-maybe/app/models/account.rb#L97-L104)

```ruby
def destroy
  super
rescue => e
  disable! if may_disable?  # 删除失败时降级为禁用状态
  raise e
end
```

---

### 2.6 RuleJob / AutoCategorizeJob / RevertImportJob

- **RuleJob**：调用 `rule.apply`，无显式异常捕获，失败依赖 Sidekiq 全局重试
- **AutoCategorizeJob**：调用 `family.auto_categorize_transactions`，无显式异常捕获
- **RevertImportJob**：调用 `import.revert`，异常处理在 Import 模型层（见 2.1）

---

## 3. 面板状态来源与用户提示

### 3.1 Import（导入）面板状态

**列表页状态徽章：** [_import.html.erb](file:///d:/fz/0601-2/solo-dogfeeding/code/22-maybe/app/views/imports/_import.html.erb#L12-L36)

| 状态 | UI 样式 |
|-----|--------|
| pending | 灰色 badge - "In Progress" |
| importing | 橙色 badge + 脉冲动画 - "Uploading" |
| failed | 红色 badge - "Failed" |
| reverting | 橙色 badge - "Reverting" |
| revert_failed | 红色 badge - "Revert Failed" |
| complete | 绿色 badge - "Complete" |

**详情页视图路由：** [show.html.erb](file:///d:/fz/0601-2/solo-dogfeeding/code/22-maybe/app/views/imports/show.html.erb#L7-L17)

| 状态 | 渲染 Partial | 用户提示 | 操作按钮 |
|-----|------------|---------|---------|
| importing | _importing.html.erb | "Import in progress" + 进度说明 | Check Status / Back to Dashboard |
| complete | _success.html.erb | "Import successful" + 数据就绪说明 | Back to Dashboard |
| failed | _failure.html.erb | "Import failed" + 检查格式说明 | **Try again**（重新 publish） |
| revert_failed | _revert_failure.html.erb | "Reverting import failed" + 联系支持 | **Try again**（重新 revert） |
| 其他 | _ready.html.erb | 预览 dry_run 数据 + 说明 | Publish Import |

---

### 3.2 Sync（账户同步）状态展示

**PlaidItem 层面板：** [_plaid_item.html.erb](file:///d:/fz/0601-2/solo-dogfeeding/code/22-maybe/app/views/plaid_items/_plaid_item.html.erb#L26-L45)

状态优先级（从上到下匹配）：
1. `syncing?` → loader 图标 + "Syncing"（脉冲动画）
2. `requires_update?` → 警告三角 + "Requires Update"（黄色，显示 Update 按钮）
3. `sync_error.present?` → 错误图标 + "Error"（红色）
4. 正常 → "Last synced X ago" 或 "Never synced"

**账户卡片状态：** [_account.html.erb](file:///d:/fz/0601-2/solo-dogfeeding/code/22-maybe/app/views/accounts/_account.html.erb#L33-L41)

- `syncing?` 时：余额区域替换为骨架屏 `bg-loader rounded-full animate-pulse`
- `pending_deletion?` 时：名称旁显示红色脉冲 "(deletion in progress...)"

**账户详情页头部：** [show/_header.html.erb](file:///d:/fz/0601-2/solo-dogfeeding/code/22-maybe/app/views/accounts/show/_header.html.erb#L11-L39)

- `syncing?` 时：标题添加 `animate-pulse` 类，刷新按钮 `disabled: true`

**账户分组侧栏：** [_accountable_group.html.erb](file:///d:/fz/0601-2/solo-dogfeeding/code/22-maybe/app/views/accounts/_accountable_group.html.erb#L9-L36)

- `account_group.syncing?` → 组名脉冲动画
- `account.syncing?` → 账户名脉冲动画

**账户列表页头部：** [index.html.erb](file:///d:/fz/0601-2/solo-dogfeeding/code/22-maybe/app/views/accounts/index.html.erb#L5-L12)

- `Current.family.syncing?` → "Sync All" 按钮 `disabled: true`

---

### 3.3 Chat（AI 对话）错误提示

**错误 UI Partial：** [_error.html.erb](file:///d:/fz/0601-2/solo-dogfeeding/code/22-maybe/app/views/chats/_error.html.erb#L3-L17)

- debug_mode? 时：显示详细错误信息 `<code><%= chat.error %></code>`
- 普通模式："Failed to generate response. Please try again."
- **Retry 按钮**：链接到 `retry_chat_path(chat)`

**控制器重试入口：** [chats_controller.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/22-maybe/app/controllers/chats_controller.rb#L44-L47)

```ruby
def retry
  @chat.retry_last_message!
  redirect_to chat_path(@chat, thinking: true)  # 显示思考中指示器
end
```

---

### 3.4 FamilyExport（数据导出）状态

**导出列表（Turbo 自动刷新）：** [_list.html.erb](file:///d:/fz/0601-2/solo-dogfeeding/code/22-maybe/app/views/family_exports/_list.html.erb#L1-L39)

| 状态 | UI 表现 | 自动刷新 |
|-----|--------|---------|
| pending / processing | loading spinner + "Exporting..." | ✅ 每 3 秒刷新（turbo_refresh_interval） |
| completed | Download 链接（downloadable? 判定） | ❌ 停止刷新 |
| failed | 红色警告图标 + "Failed" | ❌ 停止刷新（无重试按钮，需重新创建导出） |

---

## 4. 重试条件与页面操作影响矩阵

### 4.1 操作可用性条件

| 操作 | 触发条件/前置状态 | 不可用状态 | 代码位置 |
|-----|-----------------|-----------|---------|
| Import.publish_later | `publishable?`（cleaned? + all mappings valid） + 未超行数 | pending 以外其他状态（除 failed 后重试） | [import.rb#L56-L63](file:///d:/fz/0601-2/solo-dogfeeding/code/22-maybe/app/models/import.rb#L56-L63) |
| Import.revert_later | `revertable?` = complete? \|\| revert_failed? | pending / importing / reverting / failed | [import.rb#L77-L83](file:///d:/fz/0601-2/solo-dogfeeding/code/22-maybe/app/models/import.rb#L77-L83) |
| Sync.start! | `may_start?` = 当前状态为 pending | syncing / completed / failed / stale | [sync.rb#L36-L38](file:///d:/fz/0601-2/solo-dogfeeding/code/22-maybe/app/models/sync.rb#L36-L38) |
| Account 手动同步 | `!account.syncing?` | 同步中自动跳过 | [accounts_controller.rb#L28-L34](file:///d:/fz/0601-2/solo-dogfeeding/code/22-maybe/app/controllers/accounts_controller.rb#L28-L34) |
| Family 全量同步 | `!Current.family.syncing?` | 同步中按钮 disabled | [accounts/index.html.erb#L10](file:///d:/fz/0601-2/solo-dogfeeding/code/22-maybe/app/views/accounts/index.html.erb#L10) |
| Chat.retry | `error.present? && needs_assistant_response?` | 无错误或最后一条非 user 消息 | [chats/_error.html.erb#L13-L16](file:///d:/fz/0601-2/solo-dogfeeding/code/22-maybe/app/views/chats/_error.html.erb#L13-L16) |
| FamilyExport 下载 | `downloadable?` = completed? + export_file.attached? | pending / processing / failed | [family_exports_controller.rb#L28-L33](file:///d:/fz/0601-2/solo-dogfeeding/code/22-maybe/app/controllers/family_exports_controller.rb#L28-L33) |

### 4.2 重试触发方式汇总

| 任务类型 | 系统自动重试 | 用户手动重试 | 重试入口 |
|---------|------------|------------|---------|
| ActiveRecord::Deadlocked | ✅ Sidekiq retry_on | - | - |
| Import 失败 | ❌ 无自动重试 | ✅ Try again 按钮 | publish_import_path |
| Import 回滚失败 | ❌ 无自动重试 | ✅ Try again 按钮 | revert_import_path |
| Sync 失败（stale） | ✅ 下次 sync_later 会新建 | ✅ 刷新按钮（开发/自托管） | sync_account_path |
| AI 响应失败 | ❌ 无自动重试 | ✅ Retry 按钮 | retry_chat_path |
| Account 删除失败 | ❌ 重置状态 | ✅ 再次操作 | 菜单删除按钮 |
| 数据导出失败 | ❌ 无重试 | ❌ 仅重新创建 | New Export 按钮 |
| Rule 执行失败 | ✅ Sidekiq 默认重试 | - | - |

### 4.3 Turbo Stream 实时更新机制

**Sync 完成广播链路：**
1. Sync.finalize_if_all_children_finalized → `perform_post_sync`
2. `syncable.broadcast_sync_complete` → `SyncCompleteEvent#broadcast`

**Account 维度广播：** [account/sync_complete_event.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/22-maybe/app/models/account/sync_complete_event.rb#L10-L37)
- 替换账户列表中的单条账户行（`broadcast_replace_to`）
- 替换侧栏账户分组（desktop + mobile 各两个 tab）
- 非 Plaid 账户触发 Family 维度广播
- `broadcast_refresh` 刷新当前账户详情页

**Family 维度广播：** [family/sync_complete_event.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/22-maybe/app/models/family/sync_complete_event.rb#L8-L20)
- 替换资产负债表 `#balance-sheet`
- 替换净值走势图 `#net-worth-chart`

**Chat 维度广播：**
- 错误追加：`broadcast_append target: "messages"`
- 错误清除：`broadcast_remove target: "chat-error"`
- 页面订阅：`turbo_stream_from @chat`（[show.html.erb#L3](file:///d:/fz/0601-2/solo-dogfeeding/code/22-maybe/app/views/chats/show.html.erb#L3)）

---

## 5. 关键数据流总结

### 5.1 Import 失败重试流程

```
用户点击 Publish
  → import.publish_later (status: importing)
  → ImportJob.perform_later
  → Sidekiq 执行失败
    → import.publish rescue
      → update!(status: :failed, error: "xxx")
用户刷新/访问详情页
  → render _failure.html.erb
  → 显示 "Try again" 按钮
用户点击 Try again
  → publish_import_path
  → imports_controller#publish
  → @import.publish_later  ← 重新入队
```

### 5.2 Sync 失败 → 页面状态传播

```
SyncJob 执行异常
  → sync.fail! + update(error: msg)
  → finalize_if_all_children_finalized
    → perform_post_sync
      → broadcast_sync_complete
        → Account::SyncCompleteEvent
          → broadcast_replace 账户行（余额从骨架屏→真实值，但错误信息不直接展示）
用户查看 PlaidItem 卡片
  → plaid_item.sync_error 从最新 sync/子 sync 获取
  → 显示红色 "Error" 状态
用户点击刷新按钮（开发/自托管）
  → sync_plaid_item_path
  → plaid_item.sync_later  ← 新建 Sync 记录（失败的旧记录保留历史）
```

### 5.3 Chat AI 响应失败重试

```
UserMessage 创建
  → after_create_commit :request_response_later
  → chat.ask_assistant_later → clear_error + AssistantResponseJob.perform_later
Job 执行失败（在 assistant.respond_to 内部）
  → chat.add_error(exception)
    → update!(error: e.to_json)
    → broadcast_append "messages" 区域追加 _error.html.erb
用户看到 Retry 按钮
  → retry_chat_path
  → chat.retry_last_message!
    → clear_error（broadcast_remove 移除错误条）
    → ask_assistant_later（重新入队）
  → redirect thinking: true（显示思考中指示器）
```

---

## 6. 代码设计模式总结

1. **状态机分离：** Sync 用 AASM，Import 用 enum（无事件机制），设计不统一
2. **错误持久化：** 关键模型（Import、Sync、Chat）均有独立 `error` 字段存储
3. **两层异常处理：** Job 层（少量）+ Model 层（业务逻辑内 rescue），主要在 Model 层
4. **UI 状态驱动：** 视图完全依赖模型 status 枚举分支渲染，无独立的 ViewModel 层
5. **实时更新差异：** Sync/Account 用 Turbo Stream 广播，FamilyExport 用定时轮询
6. **重试按钮前置条件：** 失败视图中统一提供重试按钮，无次数限制
