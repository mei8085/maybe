# Modal Safety Analysis: 删除/撤销确认保护机制

## 概述

本项目的删除/撤销确认保护完全依赖 **Turbo 的 `data-turbo-confirm` 前端钩子**。整个流程是纯前端拦截——如果用户绕过前端（如直接构造 HTTP 请求、禁用 JS、修改 DOM），后端没有任何二次确认保护。

部分高风险操作（有依赖关系的分类/标签删除）使用了**独立的删除页面流程**（DeletionsController），但那是业务流程上的二次确认，而非安全防护。

---

## 保护强度分级标准

### 确认保护强度（5 级）

| 级别 | 名称 | 说明 |
|---|---|---|
| **Level 4** | 独立删除页面流程 | 独立的删除确认页面 + 可选替换资源 + 后端业务校验 |
| **Level 3** | 前端确认 + 后端校验 | 前端弹窗确认 + 后端有业务逻辑校验（状态/关联/数量） |
| **Level 2** | 完整前端确认 | 前端弹窗确认（CustomConfirm 完整配置）+ 后端无业务校验 |
| **Level 1** | 简单前端确认 | `turbo_confirm: true` 或字符串（无 CustomConfirm 结构化配置）+ 后端无业务校验 |
| **Level 0** | 无保护 | 无任何确认属性，点击即删 |

### 后端校验类型说明

后端校验分为 **3 类**，其中仅"业务逻辑校验"算真正的删除保护，另外两类是通用权限控制：

| 类型 | 说明 | 举例 |
|---|---|---|
| **业务逻辑校验** | 针对删除操作本身的状态/关联/数量校验 | `revertable?`、`linked?`、`can_deactivate`、`row_count_exceeded?` |
| **管理员权限校验** | 检查 `Current.user.admin?` 角色权限 | invitations#destroy、settings/profiles#destroy、users#reset |
| **资源归属校验** | 通过 `Current.family.xxx` 或 `Current.user.xxx` 查找资源，确保只能操作自己/家庭的资源 | api_keys#destroy、大部分 destroy action |

> **注意**：管理员权限校验和资源归属校验是应用的通用权限机制，不是专门为删除确认设计的保护。所有操作都隐式或显式地有资源归属校验。

### 异步任务反馈强度（4 级）

| 级别 | 名称 | 说明 |
|---|---|---|
| **Level 3** | 完整反馈 | 状态字段 + 错误消息存储 + 视图显示错误 |
| **Level 2** | 状态 + 错误存储 | 状态字段更新 + 错误消息存储到数据库 |
| **Level 1** | 仅状态重置 | 失败后重置状态，但不保存错误信息 |
| **Level 0** | 无反馈 | Job 无 rescue，失败无任何痕迹 |

---

## 阶段 1：确认弹窗的触发

### 1.1 三种确认数据注入方式

删除操作的确认数据注入有 **3 种层级**，对应不同的保护强度：

| 注入方式 | 保护级别 | 代码位置 |
|---|---|---|
| **`confirm: CustomConfirm.new(...)`** | Level 2 / Level 3 | [DS::Button#merged_opts](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/components/DS/button.rb#L28-L30)、[DS::MenuItem#merged_opts](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/components/DS/menu_item.rb#L52-L54) |
| **`data: { turbo_confirm: true }`** | Level 1 | 直接写在 view 中的 `button_to` |
| **无确认（直接提交）** | Level 0 | 部分 `_category.html.erb` 等 |

```ruby
# Level 2/3：组件层注入（推荐）
DS::Button.new(confirm: CustomConfirm.for_resource_deletion("account", high_severity: true))

# Level 1：原生 turbo_confirm 属性（简略）
button_to "Delete", path, method: :delete, data: { turbo_confirm: true }

# Level 0：无确认（危险）
menu.with_item(variant: "button", text: "Delete", href: category_path(category), method: :delete)
```

### 1.2 确认数据构造：CustomConfirm

[CustomConfirm](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/helpers/custom_confirm.rb) 负责生成弹窗的 title/body/按钮文本/变体：

| 工厂方法 | 用途 | 按钮变体 |
|---|---|---|
| `CustomConfirm.for_resource_deletion(name, high_severity:)` | 通用资源删除 | `high_severity=true` → `destructive`（红色实心），否则 `outline-destructive`（红色描边） |
| `CustomConfirm.new(title:, body:, btn_text:, destructive:, high_severity:)` | 自定义（撤销、禁用 MFA 等） | 默认 `primary`（蓝色），`destructive: true` 时为红色 |

---

## 阶段 2：所有删除/撤销入口点汇总（按保护强度分级）

### Level 4：独立删除页面流程（最高保护）

| 操作 | 触发条件 | 删除页面 | 后端 |
|---|---|---|---|
| 删除分类（有交易时） | `category.transactions.any?` | [category/deletions/new.html.erb](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/views/category/deletions/new.html.erb) | [Category::DeletionsController](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/controllers/category/deletions_controller.rb) |
| 删除标签（有交易时） | `tag.transactions.any?` | [tag/deletions/new.html.erb](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/views/tag/deletions/new.html.erb) | [Tag::DeletionsController](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/controllers/tag/deletions_controller.rb) |

**流程特点**：
- 用户选择替换资源后才提交
- 按钮无二次确认弹窗（页面本身就是确认）
- 后端 `replace_and_destroy!` 方法确保数据完整性

---

### Level 3：前端确认 + 后端业务校验

| 操作 | 视图入口 | 确认方式 | 后端 Action | 后端业务校验 |
|---|---|---|---|---|
| 删除账户（手动账户） | [\_menu.html.erb](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/views/accounts/show/_menu.html.erb#L17-L25) | `CustomConfirm.for_resource_deletion("account", high_severity: true)` | [accounts#destroy](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/controllers/accounts_controller.rb#L56-L63) | `linked?` 检查（已关联账户不可删） |
| 撤销 Import | [\_import.html.erb](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/views/imports/_import.html.erb#L43-L53) | `CustomConfirm.new(title: "Revert import?", ...)` | [imports#revert](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/controllers/imports_controller.rb#L41-L44) | `revertable?` 校验（`complete? \|\| revert_failed?`，模型层 raise） |
| 注销用户 | [show.html.erb](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/views/settings/profiles/show.html.erb#L157-L166) | `CustomConfirm.new(title: "Reset account?", ...)` | [users#destroy](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/controllers/users_controller.rb#L43-L50) | `can_deactivate` 模型验证（管理员+多用户不可删） |
| 删除团队成员 | [show.html.erb](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/views/settings/profiles/show.html.erb#L54-L60) | `CustomConfirm.for_resource_deletion(user.display_name, high_severity: true)` | [settings/profiles#destroy](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/controllers/settings/profiles_controller.rb#L10-L17) | 管理员权限 + 不能删除自己 |
| 撤销邀请 | [show.html.erb](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/views/settings/profiles/show.html.erb#L101-L107) | `CustomConfirm.for_resource_deletion(invitation.email, high_severity: true)` | [invitations#destroy](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/controllers/invitations_controller.rb#L37-L44) | 管理员权限校验 |

> **排除说明**：以下两项曾误入 Level 3，实际不属于此级别：
> - **Import 发布**：按钮无 `confirm` 属性（见下方"无前端确认的操作"小节），不属于任何前端确认级别。
> - **撤销 API Key**：前端仅字符串 `turbo_confirm`（Level 1 确认），后端仅资源归属范围（`Current.user.api_keys.active.first`），无业务校验也无管理员校验，已归入 Level 1。

---

### 无前端确认的操作（仅有后端业务校验）

| 操作 | 视图入口 | 前端确认 | 后端 Action | 后端业务校验 |
|---|---|---|---|---|
| 发布 Import | [\_ready.html.erb](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/views/imports/_ready.html.erb#L38) | ❌ **无**（`DS::Button.new(text: "Publish import", ...)`，无 confirm 参数） | [imports#publish](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/controllers/imports_controller.rb#L4-L10) | `row_count_exceeded?`（MaxRowCountExceededError）+ `publishable?` |
| Import 发布重试 | [\_failure.html.erb](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/views/imports/_failure.html.erb#L14) | ❌ **无**（`DS::Button.new(text: "Try again", ...)`，无 confirm 参数） | 同上 | 同上 |

---

### Level 2：完整前端确认 + 后端无校验

| 操作 | 视图入口 | 确认方式 | 后端 Action |
|---|---|---|---|
| 删除 Plaid 连接 | [\_plaid_item.html.erb](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/views/plaid_items/_plaid_item.html.erb#L66-L75) | `CustomConfirm.for_resource_deletion(plaid_item.name, high_severity: true)` | [plaid_items#destroy](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/controllers/plaid_items_controller.rb#L35-L38) |
| ❌ **禁用 MFA** | [show.html.erb](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/views/settings/securities/show.html.erb#L24-L35) | `CustomConfirm.new(destructive: true)` | [mfa#disable](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/controllers/mfa_controller.rb#L42-L45) |
| 删除 Import | [\_import.html.erb](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/views/imports/_import.html.erb#L56-L63) | `CustomConfirm.for_resource_deletion("import")` | [imports#destroy](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/controllers/imports_controller.rb#L55-L59) |
| 删除交易（详情页） | [show.html.erb](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/views/transactions/show.html.erb#L160-L167) | `CustomConfirm.for_resource_deletion("transaction")` | entries#destroy（单条） |
| 删除 Valuation | [show.html.erb](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/views/valuations/show.html.erb#L66-L70) | `CustomConfirm.for_resource_deletion("value update")` | entries#destroy |
| 删除规则 | [\_rule.html.erb](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/views/rules/_rule.html.erb#L66-L72) | `CustomConfirm.for_resource_deletion("rule")` | [rules#destroy](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/controllers/rules_controller.rb#L57-L60) |
| 删除聊天 | [\_chat.html.erb](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/views/chats/_chat.html.erb#L22-L28)、[\_chat_nav.html.erb](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/views/chats/_chat_nav.html.erb#L32-L38) | `CustomConfirm.for_resource_deletion("chat")` | [chats#destroy](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/controllers/chats_controller.rb#L37-L42) |
| 删除标签（无交易时） | [\_tag.html.erb](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/views/tags/_tag.html.erb#L23-L30) | `CustomConfirm.for_resource_deletion(tag.name)` | [tags#destroy](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/controllers/tags_controller.rb#L32-L35) |
| 删除 Family Merchant | [\_family_merchant.html.erb](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/views/family_merchants/_family_merchant.html.erb#L20-L26) | `CustomConfirm.for_resource_deletion(family_merchant.name)` | family_merchants#destroy |
| 删除全部分类 | [index.html.erb](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/views/categories/index.html.erb#L6-L12) | `CustomConfirm.for_resource_deletion("all categories", high_severity: true)` | [categories#destroy_all](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/controllers/categories_controller.rb#L59-L62) |
| 删除全部标签 | [index.html.erb](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/views/tags/index.html.erb#L6-L12) | `CustomConfirm.for_resource_deletion("all tags", high_severity: true)` | [tags#destroy_all](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/controllers/tags_controller.rb#L37-L40) |
| 删除全部规则 | [index.html.erb](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/views/rules/index.html.erb#L6-L12) | `CustomConfirm.for_resource_deletion("all rules", high_severity: true)` | [rules#destroy_all](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/controllers/rules_controller.rb#L62-L65) |

> **⚠️ MFA 禁用的特殊风险**：禁用 MFA 是高敏感操作，但 [mfa#disable](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/controllers/mfa_controller.rb#L42-L45) **完全没有后端二次验证**，只有前端弹窗确认。用户绕过前端即可直接禁用 MFA。

---

### Level 1：简单前端确认 + 后端无业务校验

| 操作 | 视图入口 | 确认方式 | 后端 Action |
|---|---|---|---|
| 批量删除交易 | [entries/_selection_bar.html.erb](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/views/entries/_selection_bar.html.erb#L9-L13)、[transactions/_selection_bar.html.erb](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/views/transactions/_selection_bar.html.erb#L17-L21) | `turbo_confirm: true`（仅默认文案） | [transactions/bulk_deletions#create](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/controllers/transactions/bulk_deletions_controller.rb#L2-L6) |
| 删除 Transfer | [show.html.erb](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/views/transfers/show.html.erb#L94-L99) | `turbo_confirm: true`（仅默认文案） | [transfers#destroy](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/controllers/transfers_controller.rb#L46-L49) |
| 删除 Trade | [show.html.erb](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/views/trades/show.html.erb#L88-L93) | `turbo_confirm: true`（仅默认文案） | entries#destroy |
| 删除 Holding | [show.html.erb](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/views/holdings/show.html.erb#L91-L95) | `turbo_confirm: true`（仅默认文案） | holdings#destroy |
| 撤销 API Key | [show.html.erb](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/views/settings/api_keys/show.html.erb#L132-L140) | `data: { turbo_confirm: "Are you sure you want to revoke this API key?" }`（字符串，非 CustomConfirm） | [settings/api_keys#destroy](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/controllers/settings/api_keys_controller.rb#L38-L45) |

> **API Key 撤销说明**：按钮使用字符串形式的 `turbo_confirm`，而非结构化的 `CustomConfirm`，属于 Level 1 简单确认。后端仅有资源归属校验（`Current.user.api_keys.active.first`），无业务逻辑校验。 |

---

### Level 0：无保护（点击即删）

| 操作 | 视图入口 | 说明 |
|---|---|---|
| ⚠️ 删除分类（无交易时） | [\_category.html.erb](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/views/categories/_category.html.erb#L18-L22) | 当 `category.transactions.none?` 时，菜单项删除按钮无 `confirm` 属性 |
| 删除 Session | 列表页菜单 | `method: :delete` 无确认 |

```erb
# app/views/categories/_category.html.erb:18-22
<% if category.transactions.any? %>
  <% menu.with_item(variant: "link", text: t(".delete"), href: new_category_deletion_path(category), ...) %>
<% else %>
  <% menu.with_item(variant: "button", text: t(".delete"), href: category_path(category), method: :delete) %>
  <%# ⚠️ 无 confirm 属性！直接删除！ %>
<% end %>
```

---

## 阶段 3：弹窗渲染与用户交互

### 3.1 Turbo 钩子接管浏览器原生 confirm

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

**关键点**：返回 `Promise<boolean>`。Turbo 等待 Promise resolve：
- `true` → 继续提交
- `false` → 中止操作

### 3.2 全局共享对话框

唯一的对话框 DOM 定义在 [\_confirm_dialog.html.erb](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/views/layouts/shared/_confirm_dialog.html.erb)：

- 基于原生 `<dialog>` 元素 + Stimulus 控制器 `confirm-dialog`
- 内置 3 个隐藏按钮（`primary` / `outline-destructive` / `destructive`），根据数据显示对应变体
- 提交机制：`<form method="dialog">`，按钮的 `value="confirm"` / `value="cancel"` 设置 `dialog.returnValue`

### 3.3 弹窗控制器逻辑

[confirm_dialog_controller.js](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/javascript/controllers/confirm_dialog_controller.js)：

```
handleConfirm(rawData)
  → #normalizeRawData()：JSON 解析 / 字符串 / boolean true → 转换为配置对象
  → #prepareDialog()：填充 title/subtitle/按钮文本、切换按钮变体
  → this.element.showModal()
  → 返回 Promise，监听 dialog close 事件，检查 returnValue === "confirm"
```

关闭弹窗的方式（均会 resolve Promise）：
1. 点击 Confirm 按钮 → `returnValue = "confirm"` → `resolve(true)`
2. 点击取消按钮（X 图标，`value="cancel"`）→ `resolve(false)`
3. 按 ESC 键（由 `dialog_controller.js` 的 hotkey 绑定 `esc:DS--dialog#close`）
4. 点击遮罩外部区域（`clickOutside` → `close()`）

> **注意 1：** ESC 和点击外部区域的 `returnValue` 是空字符串，不等于 `"confirm"`，等同于取消。
> **注意 2：** 用户可在 DevTools 执行 `document.getElementById("confirm-dialog").close("confirm")` 强制绕过。

---

## 阶段 4：后端提交动作

确认通过后，Turbo 正常执行请求。后端完全信任请求，**没有二次校验 token 或确认标志**。

### 4.1 同步删除（直接 destroy）

| Controller#action | Model 方法 | 错误处理 | 成功反馈 |
|---|---|---|---|
| [categories#destroy](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/controllers/categories_controller.rb#L53-L57) | `@category.destroy` | 无 | notice: t(".success") |
| [categories#destroy_all](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/controllers/categories_controller.rb#L59-L62) | `Current.family.categories.destroy_all` | 无 | notice: "All categories deleted" |
| [tags#destroy](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/controllers/tags_controller.rb#L32-L35) | `@tag.destroy!` | 无（异常冒泡 500） | notice: t(".deleted") |
| [tags#destroy_all](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/controllers/tags_controller.rb#L37-L40) | `Current.family.tags.destroy_all` | 无 | notice: "All tags deleted" |
| [rules#destroy](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/controllers/rules_controller.rb#L57-L60) | `@rule.destroy`（无 bang，静默失败） | 无 | notice: "Rule deleted" |
| [rules#destroy_all](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/controllers/rules_controller.rb#L62-L65) | `Current.family.rules.destroy_all` | 无 | notice: "All rules deleted" |
| [chats#destroy](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/controllers/chats_controller.rb#L37-L42) | `@chat.destroy` | 无 | notice: "Chat was successfully deleted" |
| [transfers#destroy](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/controllers/transfers_controller.rb#L46-L49) | `@transfer.destroy!` | 无（异常冒泡 500） | notice: t(".success") |
| [imports#destroy](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/controllers/imports_controller.rb#L55-L59) | `@import.destroy` | 无 | notice: "Your import has been deleted." |
| [mfa#disable](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/controllers/mfa_controller.rb#L42-L45) | `Current.user.disable_mfa!` | 无 | notice: t(".success") |
| [sessions#destroy](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/controllers/sessions_controller.rb#L25-L28) | `@session.destroy` | 无 | notice: t(".logout_successful") |
| [transactions/bulk_deletions#create](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/controllers/transactions/bulk_deletions_controller.rb#L2-L6) | `destroy_by(id: ...)` | 无 | notice: "N transactions deleted" |
| [settings/api_keys#destroy](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/controllers/settings/api_keys_controller.rb#L38-L45) | `@api_key.revoke!` | 失败 → alert | notice: "API key has been revoked successfully" |
| [invitations#destroy](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/controllers/invitations_controller.rb#L37-L44) | `@invitation.destroy` | 非管理员 → alert | 无显式 notice |

---

### 4.2 异步/延迟删除（Job 队列）

#### 账户删除（Account）

```
accounts#destroy
  → @account.linked? 检查
    → 失败 → redirect alert: "Cannot delete a linked account"
    → 成功 → @account.destroy_later
        → [account.rb#L91-L94](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/models/account.rb#L91-L94)
            → mark_for_deletion!（AASM 状态：pending_deletion）
            → DestroyJob.perform_later(self)
```

**Account 删除失败恢复**（[account.rb#L97-L104](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/models/account.rb#L97-L104)）：
```ruby
def destroy
  super
rescue => e
  # Account 重写了 destroy 方法，失败时恢复到 disabled 状态
  disable! if may_disable?
  raise e  # 重新抛出，由上层处理
end
```

#### PlaidItem 删除

```
plaid_items#destroy
  → @plaid_item.destroy_later
      → [plaid_item.rb#L44-L47](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/models/plaid_item.rb#L44-L47)
          → update!(scheduled_for_deletion: true)  # 布尔字段
          → DestroyJob.perform_later(self)
```

#### Import 发布/撤销

```
imports#publish → @import.publish_later
  → 校验 row_count_exceeded?（raise MaxRowCountExceededError，controller 层 rescue）
  → 校验 publishable?（raise）
  → update! status: :importing
  → ImportJob.perform_later(self)

imports#revert → @import.revert_later
  → [import.rb#L77-L83](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/models/import.rb#L77-L83)
      → 校验 revertable?（complete? || revert_failed?，raise）
      → update! status: :reverting
      → RevertImportJob.perform_later(self)
```

#### 用户注销

```
users#destroy → @user.deactivate
  → [user.rb#L101-L103](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/models/user.rb#L101-L103)
      → update active: false, email: deactivated_email
      → after_update_commit :purge_later
          → UserPurgeJob.perform_later(self)
```

#### 家庭重置

```
users#reset → FamilyResetJob.perform_later(Current.family)
```

---

### 4.3 DestroyJob 的 BUG

[DestroyJob](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/jobs/destroy_job.rb) 有一个潜在问题：

```ruby
def perform(model)
  model.destroy
rescue => e
  model.update!(scheduled_for_deletion: false)  # ⚠️ 问题在这里！
end
```

- 对 **PlaidItem**（有 `scheduled_for_deletion` 字段）：✅ 正常工作，失败后重置状态
- 对 **Account**（**没有** `scheduled_for_deletion` 字段，用 AASM `pending_deletion` 状态）：❌ 会抛出 `ActiveModel::MissingAttributeError`！

**Account 的恢复流程**：Account 自己的 `destroy` 方法已经处理了状态恢复（`disable!`），但 DestroyJob 的 rescue 会在这之后尝试更新不存在的字段，导致二次异常。

---

## 阶段 5：异步任务失败反馈（按反馈强度分级）

### Level 3：完整反馈（状态 + 错误 + 视图显示）

| 操作 | Job | 失败处理 | 错误存储 | 视图反馈 |
|---|---|---|---|---|
| Import 发布 | [ImportJob](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/jobs/import_job.rb) | `import.publish` 内部 rescue | ✅ `status: :failed`, `error: message` | Import 列表显示失败状态 |
| Import 撤销 | [RevertImportJob](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/jobs/revert_import_job.rb) | `import.revert` 内部 rescue | ✅ `status: :revert_failed`, `error: message` | Import 列表显示失败状态和重试按钮 |

**Import 内部的 rescue 逻辑**（[import.rb#L73-L75, L94-L96](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/models/import.rb#L73-L96)）：
```ruby
def publish
  # ... 业务逻辑 ...
  update! status: :complete
rescue => error
  update! status: :failed, error: error.message  # ✅ 保存错误信息
end

def revert
  # ... 业务逻辑 ...
  update! status: :pending
rescue => error
  update! status: :revert_failed, error: error.message  # ✅ 保存错误信息
end
```

---

### Level 2：状态重置 + 无错误存储

| 操作 | Job | 失败处理 | 错误存储 | 视图反馈 |
|---|---|---|---|---|
| PlaidItem 删除 | [DestroyJob](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/jobs/destroy_job.rb) | Job 层 rescue，`update!(scheduled_for_deletion: false)` | ❌ 不保存错误信息 | 删除动画消失，按钮恢复可点击，但无失败提示 |

---

### Level 1：状态恢复（Account 特殊处理）

| 操作 | Job | 失败处理 | 错误存储 | 视图反馈 |
|---|---|---|---|---|
| Account 删除 | [DestroyJob](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/jobs/destroy_job.rb) | Account.destroy 重写 rescue，`disable!` 恢复状态 | ❌ 不保存错误信息 | pending_deletion 动画消失，账户恢复 disabled 状态，但无失败提示 |

---

### Level 0：无反馈（完全无痕迹）

| 操作 | Job | 失败处理 | 错误存储 | 视图反馈 |
|---|---|---|---|---|
| 用户数据清理 | [UserPurgeJob](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/jobs/user_purge_job.rb) | ❌ 无 rescue | ❌ 无 | 完全无反馈，失败后用户已被 deactivate 但数据未清理 |
| 家庭重置 | [FamilyResetJob](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/jobs/family_reset_job.rb) | ❌ 无 rescue | ❌ 无 | 完全无反馈，失败后数据可能部分删除 |

---

## 阶段 6：错误反馈（同步操作）

错误反馈有 4 种层次，**均与确认弹窗完全解耦**，确认弹窗无法感知后端错误：

### 6.1 Flash 消息（重定向场景）

大部分 destroy 动作使用 `redirect_to ..., notice:` 或 `alert:`。成功时显示 notice，失败时（少数有校验的）显示 alert。

### 6.2 Model 验证错误（表单场景）

仅对带有表单的编辑操作生效，标准删除操作基本不涉及：
- [\_form_errors.html.erb](file:///d:/fz/0601-2/solo-dogfeeding/code/43-maybe/app/views/shared/_form_errors.html.erb) 渲染 `model.errors.full_messages`

### 6.3 状态字段更新（异步 Job 场景）

异步操作通过模型状态字段反馈，用户需要刷新页面才能看到：
- Import: `revert_failed` / `failed` 状态
- PlaidItem: `scheduled_for_deletion` 标记，失败后重置为 false
- Account: `pending_deletion` 状态，失败后恢复为 `disabled`
- 视图中显示 `(deletion in progress...)` 动画

### 6.4 异常冒泡

以下操作失败时直接抛异常，由 Rails 全局错误处理（500 页面）：
- `tags#destroy`（`destroy!` 抛异常）
- `transfers#destroy`（`destroy!` 抛异常）
- `imports#revert`（`revert_later` 的 `revertable?` 校验失败抛异常，controller 未 rescue）

---

## 整体流程图

### 弹窗确认流程（快速删除）

```
用户点击删除按钮
    │
    ▼
[View 层] button_to / link_to
    │  data-turbo-confirm="{title, body, confirmText, variant}" 或 true 或 无
    │
    ├─ 有 turbo_confirm 属性 → Turbo 拦截
    │       │
    │       ▼
    │ [Turbo] Turbo.config.forms.confirm 钩子
    │       │  调用 confirm-dialog 控制器 handleConfirm(data)
    │       ▼
    │ [Stimulus] confirm_dialog_controller
    │       │  1. 解析数据  2. 更新弹窗 DOM  3. showModal()
    │       ▼
    │ [用户交互] 弹窗显示
    │       │
    │       ├─ 取消 / ESC / 点外部 → Promise.resolve(false) → Turbo 中止 ✓
    │       │
    │       └─ 确认按钮 → Promise.resolve(true) → Turbo 继续提交
    │                            │
    └────────────────────────────┘
                        │
                        ▼
              [HTTP 请求] DELETE / PUT / POST
                    │  无任何确认 token 或签名
                    ▼
              [Backend Controller]
                    │
                    ├─ 同步 destroy / destroy!
                    │    ├─ 成功 → redirect notice
                    │    └─ 失败 → 500 / alert（少数有验证）
                    │
                    └─ 异步 *_later（入队 Job）
                         ├─ 前置校验（如 revertable? / linked?）
                         └─ 立即返回 notice（Job 失败无主动反馈）
```

### 独立删除页面流程（有依赖关系的资源）

```
用户点击删除按钮（有依赖关系时）
    │
    ▼
[GET] /xxx/:id/deletions/new（modal/turbo_frame）
    │
    ▼
显示删除确认页面
    ├─ 显示影响范围说明
    ├─ 可选替换资源（replacement_id）
    └─ 提交按钮（无二次确认弹窗）
    │
    ▼
[POST] /xxx_deletions
    │
    ▼
replace_and_destroy!(replacement)
    │
    ▼
redirect + notice
```

---

## 风险与绕过点

| 绕过方式 | 说明 | 防护现状 |
|---|---|---|
| **直接发送 HTTP 请求** | curl/Postman 发 DELETE 请求，或控制台 `fetch('/tags/1', {method: 'DELETE'})` | ❌ 无防护，仅靠 Devise 登录态和资源归属校验 |
| **修改 DOM 删除属性** | F12 删除按钮上的 `data-turbo-confirm` 属性 | ❌ 无防护 |
| **禁用 JavaScript** | Turbo 不加载，`button_to` 生成的 form 仍可直接提交（带 `_method=delete`） | ❌ 无防护 |
| **JS 控制台调用 close** | `document.getElementById('confirm-dialog').close('confirm')` 强制返回确认 | ❌ 仅前端防护 |
| **无确认的删除点** | 无交易的分类删除（`_category.html.erb`）等少数入口没有 confirm | ❌ 完全无确认 |
| **后台 Job 失败无感知** | `UserPurgeJob`、`FamilyResetJob` 失败后完全无反馈；`DestroyJob` 无错误信息 | ⚠️ 部分有状态重置，但无失败原因 |
| **批量操作风险放大** | `destroy_all` 系列操作一次删除全部资源，仅一次弹窗确认 | ⚠️ 有确认但单次确认即全删 |
| **高敏感操作无后端校验** | 禁用 MFA（`mfa#disable`）只有前端弹窗，无后端二次验证 | ❌ 高风险 |
| **DestroyJob 兼容性 BUG** | Account 删除失败时，DestroyJob 尝试更新不存在的 `scheduled_for_deletion` 字段 | ⚠️ 状态已由 Account.destroy 恢复，但 Job 会抛二次异常 |

---

## 已有的保护措施

### 业务流程保护

1. **分层确认流程**：
   - 无依赖资源：弹窗快速确认（Level 1-3）
   - 有依赖资源（分类/标签）：跳转独立删除页面，可选择替换资源（Level 4）

2. **部分操作前置校验**（Level 3）：
   - `import.revert_later` 检查 `revertable?`
   - `user.deactivate` 检查 `can_deactivate`（管理员+多用户不可删）
   - `account.destroy` 检查 `linked?`（已关联账户不可删）
   - API Key 撤销、邀请删除等有管理员权限校验

### 数据保护

3. **软删除/延迟删除**：
   - User：软删除（`active: false`）+ 异步 purge
   - Account / PlaidItem：状态标记 + 异步 `DestroyJob`
   - Account 销毁失败自动恢复到 `disabled` 状态

4. **专门的 DeletionsController**：
   - 有依赖关系的分类/标签删除走独立页面流程
   - 提供替换资源选项，降低数据丢失风险

### 视觉保护

5. **按钮变体分级**：
   - 高严重级别使用 `destructive` 红色实心按钮
   - 普通删除使用 `outline-destructive` 红色描边按钮
   - 视觉警示分级

### 异步反馈保护

6. **Import 完整反馈链**（Level 3）：
   - `publish` 和 `revert` 方法内部 rescue
   - 状态字段 + 错误消息存储 + 视图显示
   - `revert_failed` 状态可重试
