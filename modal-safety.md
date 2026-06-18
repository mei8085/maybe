# Modal Safety Analysis: 删除/撤销确认保护机制

## 概述

本项目的删除/撤销确认保护完全依赖 **Turbo 的 `data-turbo-confirm` 前端钩子**。整个流程是纯前端拦截——如果用户绕过前端（如直接构造 HTTP 请求、禁用 JS、修改 DOM），后端没有任何二次确认保护。

---

## 完整代码路径梳理

### 阶段 1：确认弹窗的触发

#### 1.1 入口：组件层注入 `data-turbo-confirm`

所有危险操作按钮通过两个组件注入确认数据：

- [DS::Button](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/components/DS/button.rb#L28-L30)：`confirm:` 参数 → 合并为 `data-turbo-confirm`
- [DS::MenuItem](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/components/DS/menu_item.rb#L52-L54)：同上逻辑

```ruby
# button.rb / menu_item.rb 中核心代码
if confirm.present?
  data = data.merge(turbo_confirm: confirm.to_data_attribute)
end
```

#### 1.2 确认数据构造：CustomConfirm

[CustomConfirm](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/helpers/custom_confirm.rb) 负责生成弹窗的 title/body/按钮文本/变体：

| 工厂方法 | 用途 | 按钮变体 |
|---|---|---|
| `CustomConfirm.for_resource_deletion(name, high_severity:)` | 资源删除 | `high_severity=true` → `destructive`（红色实心），否则 `outline-destructive`（红色描边） |
| `CustomConfirm.new(...)` | 自定义（如撤销 Import） | 默认 `primary`（蓝色） |

**关键示例：**

- [\_tag.html.erb](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/views/tags/_tag.html.erb#L23-L30)：删除 Tag（无关联交易时），使用 `CustomConfirm.for_resource_deletion`
- [\_menu.html.erb](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/views/accounts/show/_menu.html.erb#L17-L25)：删除 Account，`high_severity: true`
- [\_import.html.erb](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/views/imports/_import.html.erb#L43-L53)：撤销 Import，使用自定义 `CustomConfirm.new`
- [\_selection_bar.html.erb](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/views/entries/_selection_bar.html.erb#L9-L13)：批量删除 Entry

---

### 阶段 2：弹窗渲染与用户交互

#### 2.1 Turbo 钩子接管浏览器原生 confirm

[application.js](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/javascript/controllers/application.js#L9-L17) 覆盖了 Turbo 的默认确认行为：

```javascript
Turbo.config.forms.confirm = (data) => {
  const confirmDialogController = application.getControllerForElementAndIdentifier(
    document.getElementById("confirm-dialog"),
    "confirm-dialog"
  );
  return confirmDialogController.handleConfirm(data);
};
```

**关键点：** 返回的是 `Promise<boolean>`。Turbo 会等待 Promise resolve：
- `true` → 继续提交表单/访问链接
- `false` → 中止操作

#### 2.2 全局共享对话框

唯一的对话框 DOM 定义在 [\_confirm_dialog.html.erb](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/views/layouts/shared/_confirm_dialog.html.erb)：

- 基于原生 `<dialog>` 元素 + Stimulus 控制器 `confirm-dialog`
- 内置 3 个隐藏按钮（`primary` / `outline-destructive` / `destructive`），根据数据显示对应变体
- 提交机制：使用 `<form method="dialog">`，按钮的 `value="confirm"` 或 `value="cancel"` 设置为 `dialog.returnValue`

#### 2.3 弹窗控制器逻辑

[confirm_dialog_controller.js](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/javascript/controllers/confirm_dialog_controller.js)：

```
handleConfirm(rawData)
  → #normalizeRawData()：解析 JSON 或直接用字符串作 title
  → #prepareDialog()：填充 title/subtitle/按钮文本、切换按钮变体
  → this.element.showModal()
  → 返回 Promise，监听 dialog close 事件，检查 returnValue === "confirm"
```

关闭弹窗的方式（均会 resolve Promise）：
1. 点击 Confirm 按钮 → `returnValue = "confirm"` → `resolve(true)`
2. 点击取消按钮（X 图标，`value="cancel"`）→ `resolve(false)`
3. 按 ESC 键（由 `dialog_controller.js` 的 hotkey 绑定 `esc:DS--dialog#close`）
4. 点击遮罩外部区域（`clickOutside` → `close()`）

> **注意 1：** ESC 和点击外部区域 2 种方式的 `returnValue` 是空字符串，不等于 `"confirm"`，因此等同于取消。
> **注意 2：** 用户可以在浏览器 DevTools 中手动执行 `document.getElementById("confirm-dialog").close("confirm")` 来绕过弹窗。

---

### 阶段 3：提交动作（后端处理）

确认通过后，Turbo 正常执行原本的请求。后端完全信任请求，**没有二次校验 token 或确认标志**。

#### 3.1 同步删除（直接 destroy）

| Controller#action | Model 方法 | 错误处理 |
|---|---|---|
| [tags#destroy](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/controllers/tags_controller.rb#L32-L35) | `@tag.destroy!` | 无异常捕获，抛异常由 Rails 全局处理 |
| [transfers#destroy](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/controllers/transfers_controller.rb#L46-L49) | `@transfer.destroy!` | 同上 |
| [rules#destroy](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/controllers/rules_controller.rb#L57-L60) | `@rule.destroy`（无 bang，失败静默） | 无错误反馈，无论成功都显示 "Rule deleted" |
| [imports#destroy](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/controllers/imports_controller.rb#L55-L59) | `@import.destroy` | 同上 |
| [sessions#destroy](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/controllers/sessions_controller.rb#L25-L28) | `@session.destroy` | 登出，正常 |

#### 3.2 异步/延迟删除

| 操作 | 代码路径 |
|---|---|
| 删除 PlaidItem | [plaid_items#destroy](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/controllers/plaid_items_controller.rb#L35-L38) → `@plaid_item.destroy_later` → [plaid_item.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/models/plaid_item.rb#L44-L47) 标记 `scheduled_for_deletion: true` + 入队 `DestroyJob` |
| 撤销 Import | [imports#revert](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/controllers/imports_controller.rb#L41-L44) → `@import.revert_later` → [import.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/models/import.rb#L77-L83) 校验 `revertable?`（`complete? \|\| revert_failed?`）→ 标记 `reverting` + 入队 `RevertImportJob` |
| 发布 Import | [imports#publish](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/controllers/imports_controller.rb#L4-L10) 有 `rescue Import::MaxRowCountExceededError` → 返回 alert |
| 注销用户 | [users#destroy](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/controllers/users_controller.rb#L43-L50) → `@user.deactivate`（软删除，更新 `active: false`）→ 有模型验证 `can_deactivate` 保护 |

---

### 阶段 4：错误反馈

错误反馈有 3 种层次，但**与确认弹窗完全解耦**，确认弹窗无法感知后端错误：

#### 4.1 Flash 消息（重定向场景）

大部分 destroy 动作使用 `redirect_to ..., notice:` 或 `alert:`：

```ruby
# 成功
redirect_to tags_path, notice: t(".deleted")
# 失败（仅 users#destroy 等少数）
redirect_to settings_profile_path, alert: @user.errors.full_messages.to_sentence
```

#### 4.2 Model 验证错误（表单场景）

仅对带有表单的编辑操作生效，删除操作基本不涉及：
- [\_form_errors.html.erb](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/views/shared/_form_errors.html.erb) 渲染 `model.errors.full_messages`

#### 4.3 异常 Rescue（部分关键操作）

- `imports#publish`：rescue `Import::MaxRowCountExceededError` → redirect with alert
- `import#revert_later`：模型层 `raise "Import is not revertable" unless revertable?`，**但 controller 中未 rescue**，异常会冒泡为 500
- `users#destroy`：通过模型验证返回 boolean，controller 分支处理

---

## 整体流程图

```
用户点击删除按钮
    │
    ▼
[View 层] button_to / link_to
    │  data-turbo-confirm="{title, body, confirmText, variant}"
    ▼
[Turbo] Turbo.config.forms.confirm 钩子被触发
    │  调用 confirm-dialog 控制器 handleConfirm(data)
    ▼
[Stimulus] confirm_dialog_controller
    │  1. 解析数据  2. 更新弹窗 DOM  3. showModal()
    ▼
[用户交互] 弹窗显示
    │
    ├─ 取消 / ESC / 点外部 → Promise.resolve(false) → Turbo 中止 ✓
    │
    └─ 确认按钮 → Promise.resolve(true) → Turbo 继续提交
                         │
                         ▼
                   [HTTP 请求] DELETE / PUT
                         │  无任何确认 token 或签名
                         ▼
                   [Backend Controller]
                         │
                         ├─ 同步 destroy / destroy!
                         │    ├─ 成功 → redirect notice
                         │    └─ 失败（仅少数有验证）→ redirect alert
                         │
                         └─ 异步 *_later（入队 Job）
                              ├─ 前置校验（如 revertable?）
                              └─ 立即返回 notice（Job 失败无反馈给用户）
```

---

## 风险与绕过点

| 绕过方式 | 说明 | 防护现状 |
|---|---|---|
| **直接发送 HTTP 请求** | 用 curl/Postman 发 DELETE 请求，或在控制台执行 `fetch('/tags/1', {method: 'DELETE'})` | ❌ 无防护，仅靠 Devise 登录态 |
| **修改 DOM 删除属性** | F12 删除按钮上的 `data-turbo-confirm` 属性 | ❌ 无防护 |
| **禁用 JavaScript** | Turbo 不加载，`button_to` 生成的 form 仍可直接提交（带 `_method=delete`） | ❌ 无防护 |
| **JS 控制台调用 close** | `document.getElementById('confirm-dialog').close('confirm')` 强制返回确认 | ❌ 仅前端防护 |
| **后台 Job 失败无感知** | `RevertImportJob`、`DestroyJob` 失败后，用户看到的是"已提交"，实际状态变为 `revert_failed` | ⚠️ 状态会更新，但无主动通知 |

---

## 已有的保护措施

1. **部分操作前置校验**：
   - `import.revert_later` 检查 `revertable?`
   - `user.deactivate` 检查 `can_deactivate`（管理员+多用户不可删）
   - Tag 删除分场景：有交易的走专门的 `new_tag_deletion_path` 页面流程，无交易的才走快速确认

2. **软删除/延迟删除**：
   - User：软删除（`active: false`）
   - PlaidItem：`scheduled_for_deletion` 标记 + 异步 Job

3. **按钮变体分级**：高严重级别使用 `destructive` 红色实心按钮，视觉警示。
