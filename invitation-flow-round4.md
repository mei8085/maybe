# 家庭邀请注册链路 - 计费主体与管理员优先级深度修订 (Round 4)

## 核心修正结论

**Round 3 中"计费始终以家庭最早创建的管理员为准"的表述不够精确**。实际计费主体选择遵循严格的两级兜底机制，且 Rails enum scope 与 `admin?` 实例方法存在关键行为差异。

### 关键发现汇总

| 发现点 | 详细说明 |
|--------|----------|
| **两级兜底机制** | 第一优先级：普通管理员(role=admin) → 第二优先级：超级管理员(role=super_admin) |
| **scope vs 实例方法差异** | `users.admin` scope 只包含 role=admin，**不包含** super_admin；但 `user.admin?` 方法对 super_admin 也返回 true |
| **异常分支** | 两级兜底都失败时抛出 RuntimeError，明确标记为无效数据状态 |
| **与邀请成员无关** | 计费主体选择完全基于家庭现有管理员，与被邀请用户的角色/身份无关 |

---

## 一、billing_email 完整实现分析

### 1.1 源码与注释

`app/models/family/subscribeable.rb:8-16`

```ruby
def billing_email
  # 第一优先级：普通管理员（role=admin）中最早创建的
  # 第二优先级：超级管理员（role=super_admin）中最早创建的
  primary_admin = users.admin.order(:created_at).first || users.super_admin.order(:created_at).first

  # 异常分支：两级兜底都失败时抛出
  unless primary_admin.present?
    raise "No primary admin found for family #{id}.  This is an invalid data state and should never occur."
  end

  primary_admin.email
end
```

### 1.2 执行流程图

```
调用 billing_email
   ↓
┌─────────────────────────────────────┐
│ 第一优先级：users.admin.order(:created_at).first │
│ （只匹配 role="admin"，不包含 super_admin）     │
└───────────────────┬─────────────────────────┘
                    ↓
              找到？→ 是 → 返回该用户邮箱
                    ↓ 否
┌─────────────────────────────────────┐
│ 第二优先级：users.super_admin.order(:created_at).first │
│ （只匹配 role="super_admin"）                      │
└───────────────────┬─────────────────────────┘
                    ↓
              找到？→ 是 → 返回该用户邮箱
                    ↓ 否
┌─────────────────────────────────────┐
│ 抛出 RuntimeError                    │
│ "No primary admin found for family..." │
└─────────────────────────────────────┘
```

---

## 二、Rails enum scope 与 admin? 方法的关键差异

### 2.1 角色定义

`app/models/user.rb:23`
```ruby
enum :role, { member: "member", admin: "admin", super_admin: "super_admin" }, validate: true
```

Rails 自动生成的 scope（精确匹配 role 字段值）：
- `User.member` → `where(role: "member")`
- `User.admin` → `where(role: "admin")`
- `User.super_admin` → `where(role: "super_admin")`

### 2.2 admin? 实例方法

`app/models/user.rb:65-67`
```ruby
def admin?
  super_admin? || role == "admin"
end
```

### 2.3 差异对比表

| 场景 | `users.admin` scope 结果 | `user.admin?` 方法结果 |
|------|--------------------------|------------------------|
| user.role = "admin" | ✅ 包含 | ✅ true |
| user.role = "super_admin" | ❌ **不包含** | ✅ true |
| user.role = "member" | ❌ 不包含 | ❌ false |

**重要结论：** 在 `billing_email` 中使用 `users.admin` scope 时，超级管理员**不会**被纳入第一优先级选择范围，必须通过第二优先级的 `users.super_admin` scope 单独查找。

---

## 三、计费主体选择场景分析

### 3.1 场景1：同时存在普通管理员和超级管理员

测试数据示例（来自 `test/fixtures/users.yml`）：
```yaml
empty:
  family: empty
  email: user1@email.com
  role: admin        # 普通管理员，先创建

maybe_support_staff:
  family: empty
  email: support@maybefinance.com
  role: super_admin  # 超级管理员，后创建
```

**计费邮箱选择结果：** `user1@email.com`（普通管理员优先）

### 3.2 场景2：只有超级管理员

```yaml
family_with_only_super_admin:
  user_a:
    role: super_admin  # 创建时间早
  user_b:
    role: super_admin  # 创建时间晚
```

**计费邮箱选择结果：** `user_a.email`（第一优先级无结果，使用第二优先级）

### 3.3 场景3：只有普通管理员

```yaml
family_with_only_admin:
  user_x:
    role: admin  # 创建时间早
  user_y:
    role: admin  # 创建时间晚
```

**计费邮箱选择结果：** `user_x.email`（第一优先级命中）

### 3.4 场景4：无任何管理员（异常场景）

```yaml
family_without_admins:
  user_m:
    role: member
  user_n:
    role: member
```

**计费邮箱选择结果：** 抛出 `RuntimeError`

---

## 四、billing_email 调用链路

### 4.1 唯一调用点

`app/controllers/subscriptions_controller.rb:15-27`
```ruby
def new
  checkout_session = stripe.create_checkout_session(
    plan: params[:plan],
    family_id: Current.family.id,
    family_email: Current.family.billing_email,  # ← 此处调用
    success_url: success_subscription_url + "?session_id={CHECKOUT_SESSION_ID}",
    cancel_url: upgrade_subscription_url
  )
  # ...
end
```

### 4.2 触发时机

只有在**创建付费订阅**（跳转 Stripe 支付页面）时才会调用 `billing_email`。激活试用（`start_trial_subscription!`）不会调用此方法。

### 4.3 权限上下文

调用 `billing_email` 时，当前用户必须是已登录状态，但：
- 不需要管理员权限（任何成员点击升级都会触发）
- 计费邮箱选择与当前用户无关，仅基于家庭数据

---

## 五、订阅状态矩阵（完整修订版）

### 5.1 非自托管环境

| 家庭订阅状态 | needs_subscription? | upgrade_required? | 跳转目标 | 计费主体确定 | 能否直接使用 |
|--------------|---------------------|-------------------|----------|--------------|-------------|
| 无订阅 (nil) | ✅ true | ✅ true | `/onboarding/trial` | 未调用 billing_email | ❌ 需激活试用 |
| 试用中 (trialing) | ❌ false | ❌ false | 无（正常访问） | 未调用 billing_email | ✅ 可以 |
| 已激活 (active) | ❌ false | ❌ false | 无（正常访问） | 未调用 billing_email | ✅ 可以 |
| 已过期 (paused) | ❌ false | ✅ true | `/subscriptions/upgrade` | 升级时调用 | ❌ 需升级 |
| 已取消 (canceled) | ❌ false | ✅ true | `/subscriptions/upgrade` | 升级时调用 | ❌ 需升级 |

### 5.2 自托管环境

| 家庭订阅状态 | needs_subscription? | upgrade_required? | 跳转目标 | 计费主体确定 | 能否直接使用 |
|--------------|---------------------|-------------------|----------|--------------|-------------|
| 任何状态 | ❌ false | ❌ false | 无（正常访问） | **永不调用 billing_email** | ✅ 可以 |

---

## 六、邀请用户权限与计费主体关系

### 6.1 权限边界总结

| 操作 | 邀请用户(member) | 邀请用户(admin) | 原家庭管理员 |
|------|------------------|-----------------|------------|
| 激活试用 | ✅ 可以 | ✅ 可以 | ✅ 可以 |
| 发起付费升级 | ✅ 可以 | ✅ 可以 | ✅ 可以 |
| 选择计费邮箱 | ❌ 无控制权 | ❌ 无控制权 | ❌ 无控制权 |
| 接收账单邮件 | ❌ 不接收 | ❌ 不接收 | ✅ 接收（按优先级） |

### 6.2 关键说明

1. **计费主体选择是纯数据驱动**：完全基于家庭现有用户的 `role` 字段和 `created_at` 时间，与谁发起操作无关
2. **被邀请用户无法影响计费主体**：即使被邀请的是 admin 角色，只要不是最早创建的 admin，就不会成为计费主体
3. **异常场景理论上不会发生**：家庭创建者默认就是 admin 角色，因此至少会有一个管理员

---

## 七、Onboarding 流程对比（保持与 Round 3 一致）

### 7.1 新家庭创建者（非自托管）

```
注册 → Setup(个人+家庭信息) → Preferences(偏好) → Goals(目标) → Trial(激活试用) → 首页
   ↓                ↓                  ↓                ↓
onboarded_at     未设置             未设置         设置onboarded_at
在Goals步骤设置
```

### 7.2 邀请用户（非自托管，家庭无订阅）

```
注册 → Setup(仅个人信息) → 跳转root → Onboardable拦截 → Trial(激活试用) → 首页
   ↓                ↓
onboarded_at     设置onboarded_at
在Setup步骤设置
```

---

## 八、关键代码位置汇总（完整修订版）

| 功能 | 文件位置 | 行号 |
|------|----------|------|
| **billing_email 完整实现** | `app/models/family/subscribeable.rb` | 8-16 |
| 角色 enum 定义 | `app/models/user.rb` | 23 |
| admin? 实例方法（含 super_admin） | `app/models/user.rb` | 65-67 |
| Onboardable 拦截逻辑 | `app/controllers/concerns/onboardable.rb` | 9-21 |
| needs_subscription? 定义 | `app/models/family/subscribeable.rb` | 44-46 |
| upgrade_required? 定义 | `app/models/family/subscribeable.rb` | 18-23 |
| can_start_trial? 定义 | `app/models/family/subscribeable.rb` | 25-27 |
| 邀请用户 Setup 分支 | `app/views/onboardings/show.html.erb` | 21, 33-44 |
| 试用激活页面逻辑 | `app/views/onboardings/trial.html.erb` | 31-51 |
| 创建试用订阅 | `app/controllers/subscriptions_controller.rb` | 30-37 |
| **billing_email 唯一调用点** | `app/controllers/subscriptions_controller.rb` | 15-27 |

---

## 九、与 Round 3 口径差异对照表

| 内容 | Round 3 表述 | Round 4 修正后 |
|------|-------------|---------------|
| 计费主体选择逻辑 | "计费始终以家庭最早创建的管理员为准" | "**两级兜底机制**：<br>1. 先找 role=admin 中最早创建的<br>2. 没有则找 role=super_admin 中最早创建的<br>3. 都没有则抛出异常" |
| admin scope 行为 | 未提及 | 明确说明：`users.admin` scope **不包含** super_admin，与 `admin?` 方法行为不同 |
| 异常分支处理 | 未提及 | 完整说明异常触发条件和错误信息 |
| billing_email 调用时机 | "无论谁激活试用" | 修正：**仅在创建付费订阅时调用**，激活试用不调用 |
| 邀请用户与计费主体关系 | "与被邀请成员无关" | 补充：即使被邀请用户是 admin 角色，也不影响计费主体选择 |
| 订阅状态矩阵中的计费列 | 未包含 | 新增"计费主体确定"列，明确各状态下是否调用 billing_email |

---

## 十、管理员角色优先级速查表

### 10.1 角色权限层级

```
super_admin（最高）
    ↓
  admin（家庭管理员）
    ↓
  member（普通成员）
```

### 10.2 计费主体优先级（与权限层级不同！）

```
1. role=admin 中 created_at 最早的  ← 第一优先级
2. role=super_admin 中 created_at 最早的  ← 第二优先级
3. 抛出异常  ← 兜底
```

**重要提示：** 计费优先级与权限层级**不一致**。超级管理员虽然权限更高，但在计费主体选择时优先级低于普通管理员。这是设计决策，确保家庭创建者（普通admin）始终是默认计费联系人。
