# 家庭邀请注册链路 - 最终口径校准 (Round 5)

## 核心校准结论

**Round 4 中"邀请 admin 不会接收账单邮件"的表述过于绝对**。被邀请的 admin 在特定条件下可以成为计费主体。本次校准对全文所有绝对化表述进行条件化处理。

### 关键校准点

| 校准项 | Round 4 绝对化表述 | Round 5 条件化表述 |
|--------|-------------------|-------------------|
| 邀请 admin 接收账单 | "❌ 不接收" | "通常不接收，除非满足特定条件" |
| 邀请用户影响计费主体 | "无法影响" | "不直接控制，但作为 admin 加入时可能间接影响" |
| 计费主体选择 | "始终以最早创建的管理员为准" | "优先选择最早创建的 role=admin，无则最早创建的 role=super_admin" |

---

## 一、邀请 admin 成为计费主体的可能性分析

### 1.1 核心逻辑回顾

`app/models/family/subscribeable.rb:8-16`
```ruby
def billing_email
  # 第一优先级：role=admin 中 created_at 最早的
  # 第二优先级：role=super_admin 中 created_at 最早的
  primary_admin = users.admin.order(:created_at).first || users.super_admin.order(:created_at).first
  # ...
end
```

### 1.2 邀请 admin 成为计费主体的条件

**必要且充分条件（同时满足）：**

| 条件 | 说明 | 是否常见 |
|------|------|---------|
| 1. 邀请角色为 `admin` | 邀请时 role 设为 admin 而非 member | 可能 |
| 2. 家庭原 admin 账号已被删除 | 原 admin 用户记录被 destroy | 罕见 |
| 3. 该邀请 admin 是目前 role=admin 中 created_at 最早的 | 没有更早创建的 role=admin 用户 | 在条件2满足时可能 |

**场景示例：**
```
原始状态：
  user_org (created_at: 2025-01-01, role: admin)  ← 计费主体

步骤1：删除 user_org
  家庭现在没有 role=admin 的用户

步骤2：邀请新用户，role=admin
  user_new (created_at: 2025-06-01, role: admin)  ← 现在成为最早的 role=admin

结果：
  billing_email → user_new.email  ✅ 邀请的 admin 成为计费主体
```

### 1.3 其他边缘场景

| 场景 | 邀请 admin 能否成为计费主体 |
|------|---------------------------|
| 家庭已有 role=admin 用户（未删除） | ❌ 不能，原 admin created_at 更早 |
| 家庭只有 role=super_admin 用户，邀请 admin | ✅ 能，成为唯一的 role=admin |
| 邀请时 role=member，后续被提升为 admin | ✅ 可能，如果成为最早的 role=admin |
| 家庭创建者就是被邀请来的（理论不可能） | - | 家庭创建者默认就是 admin |

---

## 二、全文绝对化表述校准

### 2.1 计费主体选择逻辑（已校准）

| 表述类型 | 校准前（绝对化） | 校准后（条件化） |
|----------|------------------|------------------|
| 优先级描述 | "第一优先级：普通管理员 → 第二优先级：超级管理员" | "第一优先级：**role=admin** 中 created_at 最早的 → 第二优先级：**role=super_admin** 中 created_at 最早的" |
| 与邀请成员关系 | "与被邀请成员无关" | "不直接由邀请身份决定，但如果邀请的是 admin 且满足时序条件，可能成为计费主体" |
| 邀请 admin 接收账单 | "❌ 不接收" | "默认不接收，**除非**成为最早创建的 role=admin 用户" |

### 2.2 权限边界表（已校准）

**原表（Round 4）：**
| 操作 | 邀请用户(member) | 邀请用户(admin) | 原家庭管理员 |
|------|------------------|-----------------|------------|
| 接收账单邮件 | ❌ 不接收 | ❌ 不接收 | ✅ 接收（按优先级） |

**校准后（Round 5）：**

| 操作 | 邀请用户(member) | 邀请用户(admin) | 原家庭管理员 |
|------|------------------|-----------------|------------|
| 激活试用 | ✅ 可以 | ✅ 可以 | ✅ 可以 |
| 发起付费升级 | ✅ 可以 | ✅ 可以 | ✅ 可以 |
| 选择计费邮箱 | ❌ 无控制权 | ❌ 无直接控制权 | ❌ 无直接控制权 |
| 接收账单邮件 | ❌ **不可能**接收 | ⚠️ **通常不**接收，特殊场景下可能 | ✅ **通常**接收（按优先级） |

### 2.3 订阅状态矩阵（保持一致，无需修改）

矩阵中的"计费主体确定"列已准确描述调用时机，无需修改。

### 2.4 管理员优先级速查表（已校准）

**权限层级（保持不变）：**
```
super_admin（最高）
    ↓
  admin（家庭管理员）
    ↓
  member（普通成员）
```

**计费主体优先级（校准描述）：**
```
1. role=admin 中 created_at 最早的  ← 第一优先级（无论是否为被邀请用户）
2. role=super_admin 中 created_at 最早的  ← 第二优先级
3. 抛出异常  ← 兜底
```

---

## 三、邀请用户角色与计费主体关系全景

### 3.1 角色决定权限，时序决定计费

```
邀请时指定的 role:
  ├─ member → 普通成员，永远不可能成为计费主体
  └─ admin → 家庭管理员，有可能成为计费主体（取决于 created_at 时序）

计费主体选择与邀请身份无关，仅取决于：
  1. 用户当前的 role 字段值
  2. 用户记录的 created_at 时间
  3. 家庭内其他用户的状态（是否存在更早的同 role 用户）
```

### 3.2 成员生命周期与计费主体变化

```
场景：家庭A，原 admin 离开，邀请新 admin 接替

时间线：
  T1: user_org 创建，role=admin → 计费主体 = user_org
  T2: 邀请 user_new，role=admin → 计费主体仍为 user_org（created_at 更早）
  T3: user_org 被删除 → 计费主体 = user_new（现在是最早的 role=admin）
```

### 3.3 与邀请相关的设计意图

1. **邀请时可以指定 admin 角色**：支持邀请外部管理员加入管理
2. **计费主体按时序自动选择**：确保家庭创建者（最早的 admin）优先成为计费联系人
3. **删除原 admin 时自动转移**：系统自动选择下一个最早的 admin，无需手动指定

---

## 四、billing_email 逻辑的隐式约定

### 4.1 为什么不直接指定计费联系人？

代码中没有"计费联系人"字段，而是通过算法动态计算。这种设计的隐含约定：

1. **避免手动维护**：不需要额外字段记录谁是计费联系人
2. **自动处理人员变动**：管理员离职/被删除时自动转移
3. **规则透明可预测**：按角色+创建时间，结果确定
4. **与邀请解耦**：邀请只是创建用户的一种方式，创建后用户就是普通用户

### 4.2 异常场景的防御性设计

```ruby
unless primary_admin.present?
  raise "No primary admin found for family #{id}.  This is an invalid data state and should never occur."
end
```

**"should never occur" 的含义：**
- 家庭创建时必然创建一个 role=admin 的用户
- 正常流程下不会出现无 admin 的家庭
- 如果出现，说明数据损坏或有 bug
- 抛出异常便于快速发现问题

---

## 五、关键代码位置（与 Round 4 一致，无新增）

| 功能 | 文件位置 | 行号 |
|------|----------|------|
| billing_email 完整实现 | `app/models/family/subscribeable.rb` | 8-16 |
| 角色 enum 定义 | `app/models/user.rb` | 23 |
| admin? 实例方法 | `app/models/user.rb` | 65-67 |
| 注册时设置角色 | `app/controllers/registrations_controller.rb` | 16-24 |
| 邀请角色验证 | `app/models/invitation.rb` | 6 |
| Onboardable 拦截逻辑 | `app/controllers/concerns/onboardable.rb` | 9-21 |

---

## 六、与 Round 4 口径差异对照表

| 内容 | Round 4 表述 | Round 5 校准后 |
|------|-------------|---------------|
| 邀请 admin 接收账单邮件 | "❌ 不接收" | "⚠️ **通常不**接收，成为最早的 role=admin 时可能接收" |
| 邀请用户与计费主体关系 | "无法影响计费主体" | "不直接控制，但邀请 admin 在特定时序下可能成为计费主体" |
| 计费主体优先级描述 | "普通管理员 → 超级管理员" | "**role=admin** 中最早的 → **role=super_admin** 中最早的" |
| 权限边界表"接收账单"列 | 邀请 admin 标记为"❌ 不接收" | 邀请 admin 标记为"⚠️ 通常不，特殊场景下可能" |
| 隐含设计意图 | 未提及 | 补充：计费主体动态计算的设计意图与异常防御 |

---

## 七、结论性速查（最终版）

### 7.1 谁会收到账单邮件？

```ruby
# 伪代码表示精确逻辑
def who_gets_bill_email(family)
  admin_users = family.users.where(role: "admin").order(:created_at)
  return admin_users.first if admin_users.any?

  super_admin_users = family.users.where(role: "super_admin").order(:created_at)
  return super_admin_users.first if super_admin_users.any?

  raise "Invalid data state!"
end
```

**答案：** 最早创建的 `role=admin` 用户，无论是否为被邀请用户。

### 7.2 邀请用户与计费主体关系总结

| 邀请角色 | 能否成为计费主体 | 条件 |
|----------|-----------------|------|
| member | ❌ 绝对不能 | 无 |
| admin | ⚠️ 可能 | 成为家庭内最早创建的 role=admin 用户 |

### 7.3 关键原则（无歧义）

1. **邀请身份不特殊**：被邀请用户注册后与其他用户在系统中无本质区别
2. **角色决定能力**：admin 角色意味着可以管理家庭，包括潜在成为计费联系人
3. **时序决定优先**：创建时间越早，在同角色内优先级越高
4. **动态而非静态**：计费主体随用户增删自动调整，无需手动配置
