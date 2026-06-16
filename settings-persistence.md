# 偏好侧栏持久化链路解析

本文档解析 Maybe 应用中侧栏偏好的持久化机制，涵盖「侧栏展开/折叠」和「侧栏标签页切换」两类偏好的完整数据流。

## 一、两种偏好持久化层级

应用存在两套独立的偏好持久化机制，分别服务于不同粒度的偏好场景：

| 层级 | 存储位置 | 生命周期 | 典型场景 |
|------|----------|----------|----------|
| 用户级（User-level） | `users` 表字段 | 跨会话、跨设备 | 侧栏展开状态、主题、默认周期 |
| 会话级（Session-level） | `sessions.data` JSON 字段 | 当前浏览器会话内 | 标签页选中状态 |

---

## 二、用户级偏好：侧栏展开/折叠

### 2.1 数据模型

偏好存储在 [user.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/2-maybe/app/models/user.rb) 模型对应的 `users` 表中：

- `show_sidebar` (boolean, default: true) — 左侧账户侧栏是否展开，迁移见 [20250212213301_add_user_sidebar_preference.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/2-maybe/db/migrate/20250212213301_add_user_sidebar_preference.rb)
- `show_ai_sidebar` (boolean, default: true) — 右侧 AI 助手侧栏是否展开，在 `create_ai_chats` 迁移中随 AI 聊天功能一同加入，见 [20250319212839_create_ai_chats.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/2-maybe/db/migrate/20250319212839_create_ai_chats.rb) L43
- `ai_enabled` (boolean, default: false) — AI 功能是否启用，同样在 create_ai_chats 迁移中添加，见同上 L44

### 2.2 写入链路（前端 → 后端 → 落库）

**触发点**：用户点击侧栏折叠/展开按钮

```
用户点击按钮
    ↓
Stimulus: app_layout_controller.js #toggleLeftSidebar / #toggleRightSidebar
    ↓
#updateUserPreference(field, value)
    ↓
fetch PATCH /users/:id
    ↓
UsersController#update
    ↓
@user.update!(user_params)
    ↓
users 表落库
```

**关键代码**：

1. 前端控制器 [app_layout_controller.js](file:///d:/fz/0601-2/solo-dogfeeding/code/2-maybe/app/javascript/controllers/app_layout_controller.js)：
   - `toggleLeftSidebar()` (L21-L25)：切换左侧栏，调用 `#updateUserPreference("show_sidebar", !isOpen)`
   - `toggleRightSidebar()` (L27-L31)：切换右侧栏，调用 `#updateUserPreference("show_ai_sidebar", !isOpen)`
   - `#updateUserPreference()` (L43-L55)：发送 PATCH 请求到 `/users/:id`，以 `user[field]=value` 格式提交

2. 后端控制器 [users_controller.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/2-maybe/app/controllers/users_controller.rb)：
   - `update` 动作 (L5-L36)：处理用户属性更新
   - `user_params` (L88-L95)：permit 了 `show_sidebar`、`show_ai_sidebar` 等字段
   - 支持 HTML 和 JSON 两种响应格式（JSON 格式返回 `head :ok`，前端静默更新）

### 2.3 读取链路（全局入口）

**全局读取入口**：[current.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/2-maybe/app/models/current.rb) — `Current.user`

```
请求到达
    ↓
Authentication concern (before_action)
    ↓
find_session_by_cookie → Session.find_by(id: cookie)
    ↓
Current.session = session_record
    ↓
Current.user → session.user（通过 CurrentAttributes 委托）
    ↓
视图/控制器中通过 Current.user.show_sidebar? 读取
```

**关键代码**：

1. [authentication.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/2-maybe/app/controllers/concerns/authentication.rb)：
   - `authenticate_user!` (L18-L28)：从 cookie 读取 session_token，查找 Session 记录，赋值给 `Current.session`
   - `Current` 是 `ActiveSupport::CurrentAttributes`，保证线程安全

2. 视图层 [application.html.erb](file:///d:/fz/0601-2/solo-dogfeeding/code/2-maybe/app/views/layouts/application.html.erb)：
   - 左侧栏 (L75-L79)：通过 `Current.user.show_sidebar?` 决定使用 `expanded_sidebar_class` 还是 `collapsed_sidebar_class`
   - 右侧栏 (L135-L139)：通过 `Current.user.show_ai_sidebar?` 控制显示

---

## 三、会话级偏好：侧栏标签页切换

### 3.1 数据模型

标签页偏好存储在 [session.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/2-maybe/app/models/session.rb) 模型的 `data` JSON 字段中：

```ruby
# sessions.data 结构
{
  "tab_preferences": {
    "account_sidebar_tab": "asset"  // 或 "liability" / "all"
  }
}
```

Session 模型提供两个访问方法：
- `get_preferred_tab(tab_key)` (L13-L15)：从 `data.dig("tab_preferences", tab_key)` 读取
- `set_preferred_tab(tab_key, tab_value)` (L17-L21)：写入并 `save!`

### 3.2 写入链路（组件 → 通道 → 落库）

**触发点**：用户点击侧栏中的标签（Assets / Debts / All）

```
用户点击标签按钮
    ↓
DS::Tabs 组件 Stimulus 控制器: tabs_controller.js #show
    ↓
判断是否有 session_key → 调用 #updateSessionPreference
    ↓
fetch PUT /current_session
    ↓
CurrentSessionsController#update
    ↓
Current.session.set_preferred_tab(tab_key, tab_value)
    ↓
sessions.data 字段更新（JSON 内部）
```

**关键代码**：

1. 组件层：
   - [_account_sidebar_tabs.html.erb](file:///d:/fz/0601-2/solo-dogfeeding/code/2-maybe/app/views/accounts/_account_sidebar_tabs.html.erb) (L24)：渲染 `DS::Tabs` 时传入 `session_key: "account_sidebar_tab"`
   - [tabs.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/2-maybe/app/components/DS/tabs.rb) (L32-L40)：`DS::Tabs` 组件接受 `session_key` 参数
   - [tabs.html.erb](file:///d:/fz/0601-2/solo-dogfeeding/code/2-maybe/app/components/DS/tabs.html.erb) (L5)：将 `session_key` 作为 Stimulus value 传给前端

2. 前端控制器 [tabs_controller.js](file:///d:/fz/0601-2/solo-dogfeeding/code/2-maybe/app/components/DS/tabs_controller.js)：
   - `show(e)` (L9-L41)：处理标签点击，切换 UI 状态
   - 若有 `urlParamKeyValue`，更新 URL 查询参数
   - 若有 `sessionKeyValue` (L38-L40)，调用 `#updateSessionPreference` 持久化
   - `#updateSessionPreference` (L43-L56)：发送 PUT 请求到 `/current_session`，提交 `tab_key` 和 `tab_value`

3. 后端控制器 [current_sessions_controller.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/2-maybe/app/controllers/current_sessions_controller.rb)：
   - `update` 动作 (L2-L8)：接收 `tab_key` 和 `tab_value`，调用 `Current.session.set_preferred_tab`

路由定义在 [routes.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/2-maybe/config/routes.rb) L36：`resource :current_session, only: %i[update]`

### 3.3 读取链路（全局入口）

**全局读取入口**：`RestoreLayoutPreferences` concern + `Current.session`

```
请求到达
    ↓
RestoreLayoutPreferences concern (before_action)
    ↓
restore_active_tabs
    ↓
Current.session.get_preferred_tab("account_sidebar_tab")
    ↓
若无则默认 "asset"
    ↓
@account_group_tab = 最终值（URL 参数优先）
    ↓
视图中作为 active_tab 传给 DS::Tabs 组件
```

**关键代码**：

1. [restore_layout_preferences.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/2-maybe/app/controllers/concerns/restore_layout_preferences.rb)：
   - `before_action :restore_active_tabs` (L5)：每个请求前恢复标签状态
   - `restore_active_tabs` (L9-L13)：
     - 优先读取 `params[:account_sidebar_tab]`（URL 参数）
     - 若无则从 `Current.session.get_preferred_tab("account_sidebar_tab")` 读取
     - 都没有则默认 `"asset"`
     - 结果存入 `@account_group_tab` 实例变量

2. 该 concern 被包含在 [application_controller.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/2-maybe/app/controllers/application_controller.rb) L2，因此所有继承 ApplicationController 的控制器都自动具备标签恢复能力。

3. 视图层使用：
   - 桌面端：[application.html.erb](file:///d:/fz/0601-2/solo-dogfeeding/code/2-maybe/app/views/layouts/application.html.erb) L85 传入 `active_tab: @account_group_tab`
   - 移动端：同上 L29 传入同一变量

---

## 四、两套机制的对比与协同

### 4.1 对比总结

| 维度 | 用户级偏好 (show_sidebar) | 会话级偏好 (account_sidebar_tab) |
|------|---------------------------|----------------------------------|
| 存储表 | `users` | `sessions` |
| 字段类型 | 独立列 (boolean) | JSON 字段内嵌套 |
| 持久化范围 | 跨所有会话/设备 | 当前浏览器会话 |
| 写入接口 | `PATCH /users/:id` | `PUT /current_session` |
| 写入控制器 | `UsersController` | `CurrentSessionsController` |
| 读取入口 | `Current.user` | `Current.session` |
| 恢复机制 | 视图直接读取属性 | `RestoreLayoutPreferences` before_action |
| 前端控制器 | `app_layout_controller.js` | `tabs_controller.js` |

### 4.2 协同关系

两套机制在布局渲染时协同工作：

```
application.html.erb 渲染
    ├─ Current.user.show_sidebar? → 决定侧栏容器是否展开（宽度）
    └─ @account_group_tab → 决定侧栏内哪个标签面板激活
         └─ 来自 RestoreLayoutPreferences，从 Current.session 读取
```

即：**用户级偏好控制侧栏的「可见性」，会话级偏好控制侧栏内的「激活标签」。**

---

## 五、全局读取入口：Current 对象

[Current](file:///d:/fz/0601-2/solo-dogfeeding/code/2-maybe/app/models/current.rb) 是应用的全局状态入口，继承自 `ActiveSupport::CurrentAttributes`。

### 5.1 关键属性

- `session` — 当前会话记录（由 Authentication concern 设置）
- `user` — 当前用户（委托自 `session.user`，支持 impersonation）
- `family` — 当前家庭（委托自 `user.family`）

### 5.2 设置时机

在 `Authentication` concern 的 `before_action :authenticate_user!` 中：
1. 从 `cookies.signed[:session_token]` 读取会话 ID
2. 查询 `Session` 记录
3. 赋值给 `Current.session`
4. `Current.user` 自动通过委托获得

由于 `CurrentAttributes` 是线程安全的，整个请求生命周期内都可以通过 `Current.user` / `Current.session` 访问当前用户和会话。

---

## 六、偏好设置页面

用户也可以在设置页面集中管理偏好，入口为 [preferences/show.html.erb](file:///d:/fz/0601-2/solo-dogfeeding/code/2-maybe/app/views/settings/preferences/show.html.erb)，由 [preferences_controller.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/2-maybe/app/controllers/settings/preferences_controller.rb) 渲染。

该页面使用 `auto-submit-form` Stimulus 控制器，修改后自动提交到 `UsersController#update`，走的是与侧栏按钮相同的用户级偏好持久化通道。

---

## 七、快速索引

| 关注点 | 文件 | 行号 |
|--------|------|------|
| 用户级偏好模型 | [user.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/2-maybe/app/models/user.rb) | L85-L87 |
| 会话级偏好模型 | [session.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/2-maybe/app/models/session.rb) | L13-L21 |
| 全局 Current | [current.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/2-maybe/app/models/current.rb) | 全文 |
| 侧栏展开写入（前端） | [app_layout_controller.js](file:///d:/fz/0601-2/solo-dogfeeding/code/2-maybe/app/javascript/controllers/app_layout_controller.js) | L21-L55 |
| 侧栏展开写入（后端） | [users_controller.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/2-maybe/app/controllers/users_controller.rb) | L5-L36, L88-L95 |
| 标签页写入（前端） | [tabs_controller.js](file:///d:/fz/0601-2/solo-dogfeeding/code/2-maybe/app/components/DS/tabs_controller.js) | L9-L56 |
| 标签页写入（后端） | [current_sessions_controller.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/2-maybe/app/controllers/current_sessions_controller.rb) | L2-L8 |
| 标签页读取恢复 | [restore_layout_preferences.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/2-maybe/app/controllers/concerns/restore_layout_preferences.rb) | L9-L23 |
| 布局渲染入口 | [application.html.erb](file:///d:/fz/0601-2/solo-dogfeeding/code/2-maybe/app/views/layouts/application.html.erb) | L74-L110 (左), L134-L155 (右) |
| 侧栏标签组件 | [_account_sidebar_tabs.html.erb](file:///d:/fz/0601-2/solo-dogfeeding/code/2-maybe/app/views/accounts/_account_sidebar_tabs.html.erb) | L24 |
| Tabs 组件定义 | [tabs.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/2-maybe/app/components/DS/tabs.rb) | L30-L40 |
| 认证与 Current 设置 | [authentication.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/2-maybe/app/controllers/concerns/authentication.rb) | L18-L28 |
