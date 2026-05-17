# 家庭邀请码与新用户注册协作链路

## 一、系统架构概览

系统存在两种邀请机制，分别服务于不同场景：

| 机制 | 适用场景 | 有效期 | 使用后行为 |
|------|----------|--------|------------|
| **Invitation (邮件邀请)** | 家庭管理员邀请指定成员加入 | 3天 | 标记为已接受 (accepted_at) |
| **InviteCode (通用邀请码)** | 自托管环境下控制注册入口 | 永不过期 | 直接销毁 |

---

## 二、核心数据模型

### 2.1 Invitation 模型 (`app/models/invitation.rb`)

**字段说明：**

| 字段 | 类型 | 作用 |
|------|------|------|
| `id` | UUID | 主键 |
| `email` | string | 被邀请人邮箱 (family_id 范围内唯一) |
| `role` | string | 授予角色: `admin` / `member` |
| `token` | string | 32字节随机十六进制字符串 (全局唯一) |
| `family_id` | UUID | 目标家庭 |
| `inviter_id` | UUID | 邀请人 (必须是家庭管理员) |
| `accepted_at` | datetime | 接受时间 (nil表示未接受) |
| `expires_at` | datetime | 过期时间 (创建时+3天) |
| `created_at` | datetime | 创建时间 |
| `updated_at` | datetime | 更新时间 |

**关键方法：**
- `pending?` - 判断邀请是否有效（未接受且未过期）
- 作用域: `pending` / `accepted`

### 2.2 InviteCode 模型 (`app/models/invite_code.rb`)

**字段说明：**

| 字段 | 类型 | 作用 |
|------|------|------|
| `id` | UUID | 主键 |
| `token` | string | 4字节随机十六进制字符串 (全局唯一) |
| `created_at` | datetime | 创建时间 |
| `updated_at` | datetime | 更新时间 |

**关键类方法：**
- `generate!` - 生成新邀请码并返回token
- `claim!(token)` - 验证并销毁邀请码，成功返回true

---

## 三、端到端流程

### 3.1 邮件邀请链路 (家庭内部邀请)

```
管理员操作 → 创建邀请 → 发送邮件 → 被邀请人点击 → 跳转注册 → 完成注册 → 加入家庭
     ↓          ↓          ↓           ↓             ↓           ↓
  权限校验   生成token   异步发送   校验token有效性  预填邮箱    设置family_id
                                                           设置role
                                                           标记accepted_at
```

**步骤详解：**

1. **邀请创建** (`invitations#create` - `app/controllers/invitations_controller.rb:7-25`)
   - 权限校验：仅家庭管理员 (`Current.user.admin?`) 可创建
   - 参数：`email` + `role` (admin/member)
   - 自动生成32字节随机token
   - 自动设置过期时间为3天后
   - 唯一性校验：同一邮箱在同一家庭只能被邀请一次
   - 非自托管环境下异步发送邀请邮件

2. **邮件发送** (`InvitationMailer.invite_email`)
   - 包含接受链接：`/invitations/:token/accept`
   - 显示邀请人姓名、家庭名称
   - 提示3天有效期

3. **邀请接受** (`invitations#accept` - `app/controllers/invitations_controller.rb:27-35`)
   - 通过token查找邀请记录
   - 校验邀请状态：必须是 `pending?` (未接受且未过期)
   - 有效则跳转至注册页：`/registrations/new?invitation=:token`
   - 无效则返回404

4. **注册页面** (`registrations#new` - `app/views/registrations/new.html.erb`)
   - 检测到 `invitation` 参数时：
     - 显示邀请信息（邀请人、角色）
     - 邮箱字段预填且禁用
     - 隐藏邀请码输入框（即使开启了邀请码要求）
     - 隐藏字段传递 invitation token

5. **用户注册** (`registrations#create` - `app/controllers/registrations_controller.rb:15-33`)
   - 通过token查找pending状态的邀请
   - 自动关联家庭：`@user.family = @invitation.family`
   - 自动设置角色：`@user.role = @invitation.role`
   - 自动设置邮箱：`@user.email = @invitation.email`
   - 注册成功后标记邀请已接受：`@invitation.update!(accepted_at: Time.current)`
   - 自动创建会话并跳转首页

### 3.2 通用邀请码链路 (自托管注册控制)

```
管理员生成 → 分发邀请码 → 用户注册时输入 → 系统验证 → 完成注册 → 创建新家庭
     ↓           ↓             ↓            ↓          ↓
  仅管理员     线下传递       claim!      销毁code    角色为admin
```

**触发条件：** (`Invitable` concern - `app/controllers/concerns/invitable.rb`)
- 自托管环境：`Setting.require_invite_for_signup` 为true
- 托管环境：`ENV["REQUIRE_INVITE_CODE"] == "true"`
- 注意：如果用户持有有效的invitation token，则跳过邀请码校验

**流程：**
1. 管理员在 `/invite_codes` 页面生成邀请码
2. 用户注册时输入邀请码
3. `InviteCode.claim!(token)` 验证并销毁邀请码
4. 验证成功则继续注册流程，创建新家庭，用户为admin角色

---

## 四、权限与订阅状态衔接

### 4.1 角色体系

```
User.role:
  ├─ member       # 普通成员
  ├─ admin        # 家庭管理员 (可邀请/删除成员)
  └─ super_admin  # 超级管理员 (系统级)
```

**权限控制：**
- 邀请创建：`admin?` (admin或super_admin)
- 邀请撤销：`admin?`
- 家庭成员删除：见 `user.rb:105-109` - 管理员在有其他成员时不能注销自己

### 4.2 订阅状态

`Family::Subscribeable` (`app/models/family/subscribeable.rb`) 提供订阅相关方法：
- `has_active_subscription?` - 是否有活跃订阅
- `trialing?` - 是否在试用期（14天）
- `upgrade_required?` - 是否需要升级

**关键特性：**
- 自托管用户 (`self_hoster?`) 无订阅限制
- 新注册用户会自动创建新家庭并开始试用
- 被邀请用户加入现有家庭，共享订阅状态

---

## 五、边界场景处理

### 5.1 邀请失效场景

| 场景 | 处理方式 | 用户体验 |
|------|----------|----------|
| 邀请过期 (expires_at < now) | `invitations#accept` 返回404 | 看到"页面不存在" |
| 邀请已被接受 (accepted_at present) | `invitations#accept` 返回404 | 看到"页面不存在" |
| 邀请被管理员删除 | token不存在，返回404 | 看到"页面不存在" |
| 注册时邀请已失效 | `@invitation` 为nil，按普通注册流程 | 可能需要邀请码 |

### 5.2 重复邀请场景

| 场景 | 处理方式 | 代码位置 |
|------|----------|----------|
| 同一邮箱重复邀请同一家庭 | 模型校验唯一性，创建失败 | `invitation.rb:8` |
| 同一邮箱邀请不同家庭 | 允许，按不同family_id区分 | - |
| 用户已注册但不在该家庭 | 仍可发送邀请，注册时提示邮箱已存在 | 由User模型唯一性校验捕获 |

### 5.3 其他边界场景

| 场景 | 处理方式 |
|------|----------|
| 非管理员尝试创建邀请 | 权限校验失败，flash提示 |
| 非管理员尝试撤销邀请 | 权限校验失败，flash提示 |
| 邀请码已被使用 | `claim!` 返回false，重定向回注册页并提示 |
| 邀请码无效 | `claim!` 返回false，重定向回注册页并提示 |
| 密码强度不符合要求 | 客户端+服务端双重校验，阻止注册 |

---

## 六、关键代码引用

### 模型层
- 邀请模型: `app/models/invitation.rb:1-37`
- 邀请码模型: `app/models/invite_code.rb:1-25`
- 用户模型: `app/models/user.rb:1-211`
- 家庭订阅: `app/models/family/subscribeable.rb:1-80`

### 控制器层
- 邀请控制器: `app/controllers/invitations_controller.rb:1-60`
- 邀请码控制器: `app/controllers/invite_codes_controller.rb:1-19`
- 注册控制器: `app/controllers/registrations_controller.rb:1-82`
- 邀请相关Concern: `app/controllers/concerns/invitable.rb:1-17`

### 视图层
- 注册页面: `app/views/registrations/new.html.erb:1-95`
- 邀请邮件: `app/views/invitation_mailer/invite_email.html.erb:1-11`

### 数据库表
- invitations: `db/schema.rb:392-407`
- invite_codes: `db/schema.rb:409-414`

---

## 七、路由概览

| HTTP方法 | 路径 | 控制器 | 说明 |
|----------|------|--------|------|
| GET | `/invitations/new` | invitations#new | 新建邀请表单 |
| POST | `/invitations` | invitations#create | 创建邀请 |
| DELETE | `/invitations/:id` | invitations#destroy | 撤销邀请 |
| GET | `/invitations/:id/accept` | invitations#accept | 接受邀请 |
| GET | `/registrations/new` | registrations#new | 注册页面 |
| POST | `/registrations` | registrations#create | 提交注册 |
| GET | `/invite_codes` | invite_codes#index | 邀请码列表（自托管） |
| POST | `/invite_codes` | invite_codes#create | 生成邀请码（自托管） |
