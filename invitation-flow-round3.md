# 家庭邀请注册链路 - 订阅边界结论修订 (Round 3)

## 核心修正结论

**Round 2 中"家庭无订阅时被邀请用户可直接使用"的表述不准确**。实际行为取决于部署模式：

| 部署模式 | 家庭无订阅时邀请用户体验 |
|----------|--------------------------|
| **自托管 (self_hosted?)** | ✅ 完成Setup后可直接使用 |
| **非自托管 (托管/云服务)** | ❌ 完成Setup后被强制跳转到试用激活页面，必须激活试用才能使用 |

**关键发现：** 被邀请用户在非自托管环境下，**有权限为无订阅的家庭激活试用**。

---

## 一、Onboardable 全局拦截器的精确逻辑

### 1.1 判断优先级与跳转路径

`app/controllers/concerns/onboardable.rb:14-20`

```ruby
# 优先级从高到低，命中即跳转
if Current.user.needs_onboarding?
  redirect_to onboarding_path                          # 1. 未完成引导
elsif Current.family.needs_subscription?
  redirect_to trial_onboarding_path                    # 2. 从未有过订阅（非自托管）
elsif Current.family.upgrade_required?
  redirect_to upgrade_subscription_path                # 3. 订阅已过期/取消（非自托管）
end
```

### 1.2 判断条件精确定义

| 方法 | 代码定义 | 精确逻辑 |
|------|----------|----------|
| `needs_onboarding?` | `onboarded_at.blank?` | 用户未完成引导流程 |
| `needs_subscription?` | `subscription.nil? && !self_hoster?` | 家庭**从未创建过**订阅记录，且非自托管 |
| `upgrade_required?` | `!self_hoster? && !(subscription&.active? \|\| subscription&.trialing?)` | 非自托管，且订阅不活跃、不在试用中 |

### 1.3 排除路径（不触发拦截）

`app/controllers/concerns/onboardable.rb:23-36`

```ruby
def redirectable_path?(path)
  return false if path.starts_with?("/settings")
  return false if path.starts_with?("/subscription")
  return false if path.starts_with?("/onboarding")
  return false if path.starts_with?("/users")
  return false if path.starts_with?("/api")
  # 认证页面也排除
  [new_registration_path, new_session_path, ...].exclude?(path)
end
```

---

## 二、邀请用户的完整跳转路径（分场景）

### 2.1 通用前置流程（所有场景一致）

```
注册成功
   ↓
创建 User（关联 invitation.family，设置 role）
   ↓
标记 invitation.accepted_at = Time.current
   ↓
创建会话，跳转 root_path
   ↓
Onboardable 拦截：needs_onboarding? = true（onboarded_at 为 nil）
   ↓
跳转 /onboarding（Setup 页面）
   ↓
用户填写 first_name、last_name、头像（邀请用户不显示家庭名称/国家选择）
   ↓
提交表单，设置 onboarded_at = Time.current（因为 @invitation 存在）
   ↓
redirect_to = "home" → 跳转 root_path
   ↓
[分支开始：根据家庭订阅状态和部署模式分流]
```

### 2.2 场景A：自托管环境（任何订阅状态）

```
跳转 root_path
   ↓
Onboardable 检查：
  needs_onboarding? = false
  needs_subscription? = false（因为 self_hoster? = true）
  upgrade_required? = false（因为 self_hoster? = true）
   ↓
✅ 正常访问首页，可直接使用
```

**代码证据：**
`app/models/family/subscribeable.rb:44-46`
```ruby
def needs_subscription?
  subscription.nil? && !self_hoster?  # 自托管时 !self_hoster? = false
end
```

### 2.3 场景B：非自托管，家庭无订阅（subscription = nil）

```
跳转 root_path
   ↓
Onboardable 检查：
  needs_onboarding? = false
  needs_subscription? = true（subscription.nil? && !self_hoster?）
   ↓
跳转 /onboarding/trial（试用激活页面）
   ↓
用户点击 "Try Maybe for 14 days" 按钮
   ↓
POST /subscriptions → start_trial_subscription!
   ↓
创建 subscription（status = "trialing", trial_ends_at = 14天后）
   ↓
跳转 root_path
   ↓
✅ 正常使用
```

**关键代码 - 试用激活判断：**
`app/views/onboardings/trial.html.erb:31-38`
```erb
<% if Current.family.can_start_trial? %>
  <%= render DS::Button.new(text: "Try Maybe for 14 days", href: subscription_path) %>
```

`app/models/family/subscribeable.rb:25-27`
```ruby
def can_start_trial?
  subscription&.trial_ends_at.blank?
end
```
→ 当 `subscription.nil?` 时，`nil&.trial_ends_at.blank?` = `nil.blank?` = **true**
→ **被邀请用户可以为无订阅家庭激活试用！**

### 2.4 场景C：非自托管，家庭订阅已过期/取消（subscription.status = paused/canceled）

```
跳转 root_path
   ↓
Onboardable 检查：
  needs_onboarding? = false
  needs_subscription? = false（subscription 不为 nil）
  upgrade_required? = true（!self_hoster? && !(active? || trialing?)）
   ↓
跳转 /subscriptions/upgrade（升级页面）
   ↓
用户选择计划，跳转 Stripe 支付
   ↓
支付成功 → 更新 subscription.status = "active"
   ↓
跳转 root_path
   ↓
✅ 正常使用
```

### 2.5 场景D：非自托管，家庭正在试用中（subscription.status = trialing）

```
跳转 root_path
   ↓
Onboardable 检查：
  needs_onboarding? = false
  needs_subscription? = false
  upgrade_required? = false（trialing? = true）
   ↓
✅ 正常访问首页，共享剩余试用天数
```

### 2.6 场景E：非自托管，家庭有活跃订阅（subscription.status = active）

```
跳转 root_path
   ↓
Onboardable 检查：
  needs_onboarding? = false
  needs_subscription? = false
  upgrade_required? = false（active? = true）
   ↓
✅ 正常访问首页，共享订阅权益
```

---

## 三、订阅状态矩阵（邀请用户视角）

### 3.1 非自托管环境

| 家庭订阅状态 | needs_subscription? | upgrade_required? | 跳转目标 | 能否直接使用 |
|--------------|---------------------|-------------------|----------|-------------|
| 无订阅 (nil) | ✅ true | ✅ true | `/onboarding/trial` | ❌ 需激活试用 |
| 试用中 (trialing) | ❌ false | ❌ false | 无（正常访问） | ✅ 可以 |
| 已激活 (active) | ❌ false | ❌ false | 无（正常访问） | ✅ 可以 |
| 已过期 (paused) | ❌ false | ✅ true | `/subscriptions/upgrade` | ❌ 需升级 |
| 已取消 (canceled) | ❌ false | ✅ true | `/subscriptions/upgrade` | ❌ 需升级 |

### 3.2 自托管环境

| 家庭订阅状态 | needs_subscription? | upgrade_required? | 跳转目标 | 能否直接使用 |
|--------------|---------------------|-------------------|----------|-------------|
| 无订阅 (nil) | ❌ false | ❌ false | 无（正常访问） | ✅ 可以 |
| 试用中 (trialing) | ❌ false | ❌ false | 无（正常访问） | ✅ 可以 |
| 已激活 (active) | ❌ false | ❌ false | 无（正常访问） | ✅ 可以 |
| 已过期 (paused) | ❌ false | ❌ false | 无（正常访问） | ✅ 可以 |
| 已取消 (canceled) | ❌ false | ❌ false | 无（正常访问） | ✅ 可以 |

---

## 四、Onboarding 流程对比（修正版）

### 4.1 新家庭创建者（非自托管）

```
注册 → Setup(个人+家庭信息) → Preferences(偏好) → Goals(目标) → Trial(激活试用) → 首页
   ↓                ↓                  ↓                ↓
onboarded_at     未设置             未设置         设置onboarded_at
在Goals步骤设置
```

### 4.2 邀请用户（非自托管，家庭无订阅）

```
注册 → Setup(仅个人信息) → 跳转root → Onboardable拦截 → Trial(激活试用) → 首页
   ↓                ↓
onboarded_at     设置onboarded_at
在Setup步骤设置
```

### 4.3 邀请用户（非自托管，家庭有有效订阅）

```
注册 → Setup(仅个人信息) → 跳转root → 正常访问首页
   ↓                ↓
onboarded_at     设置onboarded_at
在Setup步骤设置
```

### 4.4 邀请用户（自托管，任何订阅状态）

```
注册 → Setup(仅个人信息) → 跳转root → 正常访问首页
   ↓                ↓
onboarded_at     设置onboarded_at
在Setup步骤设置
```

---

## 五、关键代码位置（与Round 2一致，无新增）

| 功能 | 文件位置 | 行号 |
|------|----------|------|
| Onboardable 拦截逻辑 | `app/controllers/concerns/onboardable.rb` | 9-21 |
| needs_subscription? 定义 | `app/models/family/subscribeable.rb` | 44-46 |
| upgrade_required? 定义 | `app/models/family/subscribeable.rb` | 18-23 |
| can_start_trial? 定义 | `app/models/family/subscribeable.rb` | 25-27 |
| 邀请用户Setup分支 | `app/views/onboardings/show.html.erb` | 21, 33-44 |
| 试用激活页面逻辑 | `app/views/onboardings/trial.html.erb` | 31-51 |
| 创建试用订阅 | `app/controllers/subscriptions_controller.rb` | 30-37 |

---

## 六、与 Round 2 口径差异对照表

| 内容 | Round 2 表述 | Round 3 修正后 |
|------|-------------|---------------|
| 家庭无订阅时邀请用户体验 | "可直接使用" | 区分自托管/非自托管：<br>✅ 自托管：可直接使用<br>❌ 非自托管：跳转试用激活页面 |
| 订阅跳转逻辑图示 | 未体现场景差异 | 增加6种场景的完整跳转路径 |
| 订阅状态矩阵 | 不准确 | 分自托管/非自托管两个维度完整呈现 |
| 邀请用户激活试用权限 | 未提及 | 明确指出：被邀请用户可以为无订阅家庭激活试用 |
| needs_subscription? 触发条件 | `subscription.nil?` | 补充完整：`subscription.nil? && !self_hoster?` |
| upgrade_required? 触发条件 | 未详细说明 | 明确：仅非自托管环境下，订阅不活跃且不在试用中 |

---

## 七、权限边界说明

### 7.1 被邀请用户可以激活试用

**代码证据：** `app/models/family/subscribeable.rb:25-27`
```ruby
def can_start_trial?
  subscription&.trial_ends_at.blank?  # subscription.nil? 时返回 true
end
```

`app/controllers/subscriptions_controller.rb:30-37`
```ruby
def create
  if Current.family.can_start_trial?
    Current.family.start_trial_subscription!  # 无权限校验
    redirect_to root_path
  end
end
```

**结论：** 订阅控制器的 `create` 动作没有校验用户角色，任何家庭成员（包括被邀请的 member）都可以为家庭激活试用。

### 7.2 计费主体不变

无论谁激活试用，计费邮箱始终是家庭最早创建的管理员：
`app/models/family/subscribeable.rb:8-16`
```ruby
def billing_email
  primary_admin = users.admin.order(:created_at).first
  primary_admin.email
end
```
