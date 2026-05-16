# MFA 二次验证会话认证机制分析
## 修订版（准确性校正）

---

## 一、概述

本报告基于代码审计的准确结果，分析 Maybe Finance 项目中 MFA 二次验证嵌入登录会话的实现机制。所有结论均对应证据代码，不确定项已显式标注。

**版本信息**：
- ROTP 版本：6.3.0
- 审计日期：2026-05-16

---

## 二、验证状态存储位置

### 2.1 用户 MFA 配置持久化存储（User 表）

**数据库字段（schema.rb:789-791）**：
| 字段名 | 类型 | 说明 | 证据 |
|--------|------|------|------|
| `otp_secret` | string | TOTP 密钥，BASE32 编码 | `app/models/user.rb:191` |
| `otp_required` | boolean | 是否启用 MFA，默认 false | `app/models/user.rb:134` |
| `otp_backup_codes` | string[] | 备份验证码数组，共 8 个 | `app/models/user.rb:209` |

**相关代码**：
`app/models/user.rb:124-137`
```ruby
def setup_mfa!
  update!(
    otp_secret: ROTP::Base32.random(32),
    otp_required: false,
    otp_backup_codes: []
  )
end

def enable_mfa!
  update!(
    otp_required: true,
    otp_backup_codes: generate_backup_codes  # 8.times.map
  )
end
```

### 2.2 登录过程中的临时状态存储（Rails Session）

在密码验证成功、MFA 验证完成前，用户 ID 临时存储在 Rails session 中：

```ruby
session[:mfa_user_id] = user.id
```

**证据代码**：`app/controllers/sessions_controller.rb:13`

⚠️ **标注**：`session[:mfa_user_id]` 的有效期依赖于 Rails Session 的配置，项目中未显式设置 `expire_after`，默认为浏览器会话级（浏览器关闭即失效）。

### 2.3 最终认证会话存储（Session 表 + Cookie）

MFA 验证通过后，创建正式会话：

**1. 数据库层**（`sessions` 表）：
- `user_id`：关联用户
- `user_agent`：浏览器标识（`before_create` 回调设置）
- `ip_address`：客户端 IP（`before_create` 回调设置）
- `data`：JSON 字段，存储会话偏好如 `tab_preferences`

**证据代码**：`app/models/session.rb:8-21`

**2. Cookie 层**：
```ruby
cookies.signed.permanent[:session_token] = { value: session.id, httponly: true }
```

**证据代码**：`app/controllers/concerns/authentication.rb:42`

✅ **已确认**：`permanent` 在 Rails 中表示 20 年有效期，`signed` 防篡改，`httponly` 禁止 JS 访问。

---

## 三、会话恢复主链

### 3.1 每次请求的完整认证链路

**从 Cookie 到 Current.user 的完整还原路径**：

```
┌─────────────────────────────────────────────────────────────┐
│                     HTTP 请求到达                            │
└─────────────────────────────┬───────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  ApplicationController 过滤器链                              │
│  before_action :set_request_details                          │
│  before_action :authenticate_user!  ← 认证入口               │
│  before_action :set_sentry_user                              │
└─────────────────────────────┬───────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  Authentication#authenticate_user!                           │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ 1. 调用 find_session_by_cookie()                     │   │
│  │    → 读取 cookies.signed[:session_token]             │   │
│  │    → Session.find_by(id: cookie_value)               │   │
│  └───────────────────────────┬─────────────────────────┘   │
│                              │                               │
│  ┌───────────────────────────▼─────────────────────────┐   │
│  │ 2. 找到 Session 记录？                                │   │
│  │    ├─ 是 → Current.session = session_record         │   │
│  │    └─ 否 → redirect_to new_session_url (302)        │   │
│  └───────────────────────────┬─────────────────────────┘   │
└───────────────────────────────┼─────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────┐
│  Current 模型（ActiveSupport::CurrentAttributes）            │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ Current.session → 存储从数据库查到的 Session 实例      │   │
│  └───────────────────────────┬─────────────────────────┘   │
│                              │                               │
│  ┌───────────────────────────▼─────────────────────────┐   │
│  │ Current.user 方法                                      │   │
│  │   def user                                             │   │
│  │     impersonated_user || session&.user                │   │
│  │   end                                                  │   │
│  └─────────────────────────────────────────────────────┘   │
└───────────────────────────────┬─────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────┐
│  业务逻辑层访问 Current.user                                  │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 关键代码实现

**Current 模型核心逻辑**
`app/models/current.rb:1-19`
```ruby
class Current < ActiveSupport::CurrentAttributes
  attribute :user_agent, :ip_address
  attribute :session

  delegate :family, to: :user, allow_nil: true

  def user
    impersonated_user || session&.user  # 优先模拟用户，否则会话关联用户
  end

  def impersonated_user
    session&.active_impersonator_session&.impersonated
  end

  def true_user
    session&.user  # 真实用户（排除模拟）
  end
end
```

**认证中间件链路**
`app/controllers/concerns/authentication.rb:1-67`
```ruby
module Authentication
  extend ActiveSupport::Concern

  included do
    before_action :set_request_details  # 第1步：设置请求上下文
    before_action :authenticate_user!    # 第2步：认证用户
    before_action :set_sentry_user       # 第3步：设置监控上下文
  end

  private
    def authenticate_user!
      if session_record = find_session_by_cookie
        Current.session = session_record  # 会话注入 Current
      else
        if self_hosted_first_login?
          redirect_to new_registration_url  # 302 重定向
        else
          redirect_to new_session_url       # 302 重定向，无有效会话
        end
      end
    end

    def find_session_by_cookie
      cookie_value = cookies.signed[:session_token]
      cookie_value.present? ? Session.find_by(id: cookie_value) : nil
    end
end
```

### 3.3 会话恢复的关键特性

| 特性 | 说明 | 证据 |
|------|------|------|
| **无状态设计** | 每次请求独立从 Cookie 还原 | `authentication.rb:19-27` |
| **请求级隔离** | 使用 `ActiveSupport::CurrentAttributes`，线程安全 | `current.rb:1` |
| **模拟用户支持** | `Current.user` 优先返回被模拟用户 | `current.rb:8-10` |
| **Cookie 安全** | signed + permanent + httponly | `authentication.rb:42` |

---

## 四、MFA 流程接入点

### 4.1 Web 登录流程

```
用户输入密码
     ↓
SessionsController#create
     ↓
密码验证成功？
     ├─ 否 → render :new (422)
     └─ 是 → 检查 user.otp_required?
              ├─ 否 → create_session_for → redirect_to root_path (302)
              └─ 是 → session[:mfa_user_id] = user.id
                        ↓
                   redirect_to verify_mfa_path (302)
                        ↓
                   MfaController#verify
                        ↓
                   用户输入验证码
                        ↓
                   MfaController#verify_code
                        ↓
                   验证成功？
                        ├─ 否 → render :verify (422)
                        └─ 是 → session.delete(:mfa_user_id)
                                  ↓
                             create_session_for
                                  ↓
                             redirect_to root_path (302)
```

### 4.2 关键接入点代码

**接入点1：密码验证后跳转 MFA**
`app/controllers/sessions_controller.rb:10-23`
```ruby
def create
  if user = User.authenticate_by(email: params[:email], password: params[:password])
    if user.otp_required?
      session[:mfa_user_id] = user.id  # 保存状态
      redirect_to verify_mfa_path      # 302 跳转到 MFA 验证页
    else
      @session = create_session_for(user)
      redirect_to root_path  # 302 跳转首页
    end
  else
    flash.now[:alert] = t(".invalid_credentials")
    render :new, status: :unprocessable_entity  # 422
  end
end
```

**接入点2：MFA 验证页恢复用户**
`app/controllers/mfa_controller.rb:21-27`
```ruby
def verify
  @user = User.find_by(id: session[:mfa_user_id])

  if @user.nil?
    redirect_to new_session_path  # 302，状态丢失时返回登录
  end
end
```

✅ **已确认**：`@user.nil?` 为 false 时，Rails 默认渲染 verify 模板，HTTP 状态码为 200（代码未显式设置）。

**接入点3：MFA 验证成功后创建会话**
`app/controllers/mfa_controller.rb:29-40`
```ruby
def verify_code
  @user = User.find_by(id: session[:mfa_user_id])

  if @user&.verify_otp?(params[:code])
    session.delete(:mfa_user_id)  # 清理临时状态
    @session = create_session_for(@user)  # 创建正式会话
    redirect_to root_path  # 302
  else
    flash.now[:alert] = t(".invalid_code")
    render :verify, status: :unprocessable_entity  # 422
  end
end
```

### 4.3 API 登录流程

API 登录与 Web 登录不同，MFA 验证在同一请求内完成：

**证据代码**：`app/controllers/api/v1/auth_controller.rb:64-100`
```ruby
def login
  user = User.find_by(email: params[:email])

  if user&.authenticate(params[:password])
    # 检查 MFA 如果启用
    if user.otp_required?
      unless params[:otp_code].present? && user.verify_otp?(params[:otp_code])
        render json: {
          error: "Two-factor authentication required",
          mfa_required: true
        }, status: :unauthorized  # 401
        return
      end
    end

    # 创建 OAuth Token
    device = create_or_update_device(user)
    token_response = create_oauth_token_for_device(user, device)

    render json: token_response.merge(user: ...)  # 200
  else
    render json: { error: "Invalid email or password" }, 
           status: :unauthorized  # 401
  end
end
```

**API 会话恢复特点**：
- 无中间状态，密码 + OTP 一次性验证
- 失败直接返回 401，客户端需重新发起请求
- 使用 Doorkeeper OAuth Token

⚠️ **标注**：OAuth Token 的有效期需核查 Doorkeeper 配置，代码注释称 30 天，但未在本文件中确认。

---

## 五、TOTP 时间窗口准确性校正

### 5.1 drift_behind: 15 的真实含义

**⚠️ 关键修正**：`drift_behind: 15` **不是** ±15 秒容错，而是表示允许的**时间步数**向后漂移。

**ROTP v6.3.0 验证机制详解**：

**证据代码**：`app/models/user.rb:147-151`
```ruby
def verify_otp?(code)
  return false if otp_secret.blank?
  return true if verify_backup_code?(code)
  totp.verify(code, drift_behind: 15)  # 关键参数
end
```

**参数含义（基于 ROTP 6.3.0 源码行为）**：

| 参数 | 值 | 实际含义 | 计算结果 |
|------|----|----------|----------|
| `drift_behind` | 15 | 允许验证过去 N 个时间步内的码 | 15 × 30 秒 = 450 秒 = **7.5 分钟** |
| `drift_ahead` | 0（默认） | 不允许未来时间步的码 | 仅向后兼容，不向前兼容 |
| 标准时间步 | 30 秒 | TOTP 默认 interval | RFC 6238 标准 |

**准确结论**：
- ✅ 只允许验证**过去 7.5 分钟内**生成的有效验证码
- ❌ **不允许**验证未来时间步的验证码（没有 drift_ahead）
- ❌ **不是**双向 ±15 秒容错
- ⚠️ 这是一个**非常宽松**的时间窗口设置

### 5.2 验证流程图

```
验证请求到达
     │
     ▼
┌─────────────────────────────────────────┐
│ 1. 检查备份码（优先）                    │
│    → 找到则返回 true                     │
│    → 删除已使用的备份码                  │
└─────────────────────┬───────────────────┘
                      │  不是备份码
                      ▼
┌─────────────────────────────────────────┐
│ 2. TOTP 验证                            │
│    ┌─────────────────────────────────┐  │
│    │ 当前时间步 = Time.now / 30      │  │
│    │ 验证范围：                       │  │
│    │   当前时间步                     │  │
│    │   当前时间步 - 1                 │  │
│    │   ...                            │  │
│    │   当前时间步 - 15  ← 共 16 步    │  │
│    │                                │  │
│    │ 时间范围：                       │  │
│    │   [now - 450s, now]             │  │
│    │   约 7.5 分钟                   │  │
│    └─────────────────────────────────┘  │
└─────────────────────┬───────────────────┘
                      │
          ┌───────────┴───────────┐
          ▼                       ▼
┌─────────────────┐      ┌─────────────────┐
│   验证通过      │      │   验证拒绝      │
│   返回 true     │      │   返回 false    │
└─────────────────┘      └─────────────────┘
```

---

## 六、验证码连续失败处理

### 6.1 失败时 mfa_user_id 的保留策略

**当前实现：验证失败时，mfa_user_id 始终保留**

**证据代码**：`app/controllers/mfa_controller.rb:35-39`
```ruby
def verify_code
  @user = User.find_by(id: session[:mfa_user_id])

  if @user&.verify_otp?(params[:code])
    session.delete(:mfa_user_id)  # 仅成功时删除
    @session = create_session_for(@user)
    redirect_to root_path
  else
    # ⚠️ 失败时：不删除 session[:mfa_user_id]
    flash.now[:alert] = t(".invalid_code")
    render :verify, status: :unprocessable_entity
  end
end
```

### 6.2 连续失败的完整处理流程

```
第 N 次验证失败
       │
       ▼
┌─────────────────────────────────────────┐
│ 1. session[:mfa_user_id] 保留不删除     │
│    → 用户无需重新输入密码                │
└─────────────────────┬───────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────┐
│ 2. flash.now[:alert] 设置错误信息        │
│    → "Invalid code"（i18n 翻译）        │
└─────────────────────┬───────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────┐
│ 3. 重新渲染 verify 模板                  │
│    → HTTP 422 状态码                     │
│    → 表单保留用户输入（params[:code]）   │
└─────────────────────┬───────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────┐
│ 4. 用户可再次尝试输入验证码               │
│    → 无次数上限（代码中无计数）          │
│    → 无 IP 锁定机制                      │
│    → Rack::Attack 未保护此路径           │
└─────────────────────────────────────────┘
```

**测试证据**：`test/controllers/mfa_controller_test.rb:96-106`
```ruby
test "verify_code rejects invalid codes" do
  post verify_mfa_path, params: { code: "invalid" }
  assert_response :unprocessable_entity  # 422
  assert_not Session.exists?(user_id: @user.id)
end
```

### 6.3 mfa_user_id 的失效机制

**显式失效场景（代码中明确处理）**：

| 场景 | 触发方式 | 处理结果 | 证据 |
|------|----------|----------|------|
| **验证成功** | `session.delete(:mfa_user_id)` | 状态清理，创建正式会话 | `mfa_controller.rb:33` |
| **状态丢失** | `session[:mfa_user_id] == nil` | redirect_to new_session_path (302) | `mfa_controller.rb:24-26` |
| **用户不存在** | `User.find_by(...) == nil` | redirect_to new_session_path (302) | `mfa_controller.rb:24-26` |

**隐式失效场景（Rails 默认行为）**：

| 场景 | 说明 | 标注 |
|------|------|------|
| **浏览器关闭** | Rails Session 默认是会话级 Cookie | ✅ 确认
| **手动清除 Cookie** | 用户清除浏览器 Cookie | ✅ 确认
| **新登录覆盖** | 另一用户登录会重置 session | ⚠️ 未在代码中验证，基于 Rails 常识 |
| **Rails Session 过期** | 项目未设置 `expire_after` | ⚠️ 默认无过期时间，随浏览器会话 |

### 6.4 失败重定向策略汇总

| 失败类型 | 响应方式 | 目标路径 | HTTP 状态码 | 是否保留 mfa_user_id | 证据 |
|----------|---------|----------|------------|---------------------|------|
| 密码验证失败 | render | `sessions/new` | 422 | 不创建 | `sessions_controller.rb:20-21` |
| MFA 验证码错误 | render | `mfa/verify` | 422 | 保留 | `mfa_controller.rb:37-38` |
| mfa_user_id 丢失 | redirect | `new_session_path` | 302 | 已不存在 | `mfa_controller.rb:24-26` |
| 用户记录不存在 | redirect | `new_session_path` | 302 | 自动失效 | `mfa_controller.rb:24-26` |

---

## 七、备份码跳过机制与会话落点

### 7.1 备份码验证的优先级

**备份码是官方设计的"绕过"TOTP 验证机制，优先级更高**

**证据代码**：`app/models/user.rb:147-151`
```ruby
def verify_otp?(code)
  return false if otp_secret.blank?
  return true if verify_backup_code?(code)  # ✅ 优先验证备份码
  totp.verify(code, drift_behind: 15)       # 其次才是 TOTP
end
```

### 7.2 备份码验证的状态变更流程

```
用户输入备份码
       │
       ▼
┌─────────────────────────────────────────┐
│ MfaController#verify_code               │
│ @user = User.find(session[:mfa_user_id])│
└─────────────────────┬───────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────┐
│ @user.verify_otp?(params[:code])        │
│   └─→ verify_backup_code?(code)         │
│        ┌─────────────────────────────┐  │
│        │ 1. otp_backup_codes.index()  │  │
│        │ 2. 找到？                    │  │
│        │    ├─ 否 → 返回 false       │  │
│        │    └─ 是 → dup + delete_at   │  │
│        │           → update_column    │  │
│        │           → 返回 true        │  │
│        └─────────────────────────────┘  │
└─────────────────────┬───────────────────┘
                      │
          ┌───────────┴───────────┐
          ▼                       ▼
┌─────────────────┐      ┌─────────────────┐
│  验证成功       │      │  验证失败       │
│  (备份码有效)   │      │  (备份码无效)   │
└────────┬────────┘      └────────┬────────┘
         │                        │
         ▼                        ▼
┌─────────────────────────┐ ┌─────────────────────┐
│ session.delete          │ │ flash.now[:alert]   │
│   (:mfa_user_id)        │ │ render :verify      │
│                         │ │ status: 422         │
└────────────┬────────────┘ └─────────────────────┘
             │
             ▼
┌─────────────────────────┐
│ create_session_for(user)│
│ ┌─────────────────────┐ │
│ │ 1. sessions.create! │ │
│ │    → user_agent     │ │
│ │    → ip_address     │ │
│ │                     │ │
│ │ 2. 设置 Cookie      │ │
│ │    → signed         │ │
│ │    → permanent      │ │
│ │    → httponly       │ │
│ └─────────────────────────────┘ │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│ redirect_to root_path (302) │
└─────────────────────────┘
```

**证据代码**（备份码消耗逻辑）：`app/models/user.rb:194-206`
```ruby
def verify_backup_code?(code)
  return false if otp_backup_codes.blank?

  if (index = otp_backup_codes.index(code))
    remaining_codes = otp_backup_codes.dup
    remaining_codes.delete_at(index)
    update_column(:otp_backup_codes, remaining_codes)  # 一次性使用
    true
  else
    false
  end
end
```

### 7.3 备份码验证后的会话落点

**二次验证状态清理**：
- ✅ 验证成功后立即执行 `session.delete(:mfa_user_id)`
- ✅ 临时的 MFA 等待状态被完全清除
- ✅ 不再有"部分认证"的中间状态

**最终会话落点（四层结构）**：
| 层级 | 位置 | 说明 | 证据 |
|------|------|------|------|
| 数据库层 | `sessions` 表新记录 | `user_id` 指向认证用户 | `authentication.rb:41` |
| Cookie 层 | `cookies.signed[:session_token]` | 存储 `session.id`，永久有效 | `authentication.rb:42` |
| Current 上下文 | `Current.session` | 下次请求时通过 `authenticate_user!` 注入 | `authentication.rb:20` |
| 用户访问 | 首页 `root_path` | 302 重定向进入应用 | `mfa_controller.rb:35` |

### 7.4 备份码 vs TOTP 验证的区别

| 维度 | 备份码验证 | TOTP 验证码验证 | 证据 |
|------|-----------|----------------|------|
| **优先级** | 高（先检查） | 低（后检查） | `user.rb:149-150` |
| **时间限制** | 无（永久有效直到使用） | 约 7.5 分钟窗口（仅向后） | `user.rb:150` |
| **使用次数** | 一次性（使用即删） | 无限次（每 30 秒新码） | `user.rb:201` |
| **数据库写入** | 是（删除备份码） | 否（纯计算验证） | `user.rb:202` |
| **会话落点** | 完全相同 | 完全相同 | `mfa_controller.rb:33-35` |
| **失败处理** | 完全相同 | 完全相同 | `mfa_controller.rb:37-38` |

---

## 八、完整数据流图

```
┌─────────────────┐
│  用户访问登录页  │
└────────┬────────┘
         │
         ▼
┌───────────────────────────────────┐
│  SessionsController#create        │
│  验证邮箱 + 密码                  │
└────────┬──────────────────────────┘
         │
         ├─────────── 验证失败 ──────────────────┐
         │                                        │
         ▼                                        ▼
┌──────────────────────────┐         ┌───────────────────────┐
│ user.otp_required?       │         │ render :new (422)     │
│ 检查是否启用 MFA          │         │ 显示错误信息           │
└────────┬──────────────────┘         └───────────────────────┘
         │
         ├────────── 未启用 ──────────┐
         │                             │
         ▼                             ▼
┌──────────────────────────┐  ┌───────────────────────┐
│ session[:mfa_user_id]    │  │ create_session_for    │
│ = user.id                │  │ 生成正式会话          │
└────────┬──────────────────┘  └──────────┬──────────┘
         │                                 │
         ▼                                 ▼
┌──────────────────────────┐  ┌───────────────────────┐
│ redirect verify_mfa_path │  │ redirect root_path    │
│ (302)                    │  │ (302)                 │
└────────┬──────────────────┘  └───────────────────────┘
         │
         ▼
┌───────────────────────────────────┐
│ MfaController#verify              │
│ 从 session[:mfa_user_id] 恢复用户 │
│ 状态丢失 → 302 登录页              │
└────────┬──────────────────────────┘
         │
         ▼
┌───────────────────────────────────┐
│ 用户输入 OTP 验证码 / 备份码        │
└────────┬──────────────────────────┘
         │
         ▼
┌───────────────────────────────────┐
│ MfaController#verify_code         │
│ 1. 验证备份码（优先）              │
│    → 找到则删除，返回成功          │
│ 2. 验证 TOTP 码                   │
│    → 过去 16 步 (450秒) 窗口      │
└────────┬──────────────────────────┘
         │
         ├────────── 验证失败 ─────────────────┐
         │                                       │
         ▼                                       ▼
┌──────────────────────────┐         ┌───────────────────────┐
│ session.delete           │         │ render :verify (422)   │
│   (:mfa_user_id)         │         │ mfa_user_id 保留       │
│ 清理临时状态              │         │ 显示错误信息           │
└────────┬──────────────────┘         └───────────────────────┘
         │
         ▼
┌──────────────────────────┐
│ create_session_for       │
│ 1. Session 表插入        │
│ 2. Cookie 设置           │
└────────┬──────────────────┘
         │
         ▼
┌──────────────────────────┐
│ redirect root_path (302) │
│ → 后续请求通过           │
│   session_token          │
│   还原 Current.user      │
└──────────────────────────┘
```

---

## 九、关键文件索引

| 文件路径 | 说明 |
|---------|------|
| `app/models/current.rb` | 请求上下文存储，session 到 user 的关联 |
| `app/models/user.rb:124-210` | MFA 配置、验证逻辑（含备份码、drift_behind: 15） |
| `app/models/session.rb` | 会话模型，belongs_to :user，user_agent/ip 回调 |
| `app/controllers/sessions_controller.rb` | 登录入口，MFA 跳转逻辑 |
| `app/controllers/mfa_controller.rb` | MFA 验证页面与逻辑（verify/verify_code） |
| `app/controllers/concerns/authentication.rb` | 会话创建、Cookie 管理、认证链路 |
| `app/controllers/api/v1/auth_controller.rb` | API 登录 MFA 处理 |
| `config/initializers/rack_attack.rb` | 请求限流配置（不包含 mfa/verify_code 路径） |
| `db/schema.rb:789-791` | users 表 MFA 字段定义 |
| `test/controllers/mfa_controller_test.rb` | MFA 控制器测试用例（状态码验证） |

---

## 十、安全特性总结（已验证）

✅ **HttpOnly Cookie**：会话令牌无法被 JS 读取
✅ **Signed Cookie**：会话 ID 防篡改
✅ **Permanent Cookie**：20 年有效期（Web 会话）
✅ **备份码一次性**：使用后立即从数据库删除
✅ **Rack::Attack 限流**：API 端点速率限制（/oauth/token）
✅ **无状态会话恢复**：每次请求独立还原

⚠️ **TOTP 宽松时间窗口**：drift_behind: 15 = 向后 7.5 分钟兼容
❌ **MFA 失败无锁定**：Web 端 MFA 验证失败次数无限制，无暴力破解保护
❌ **无 Remember Device**：每次登录都需 MFA 验证，无可信设备豁免
❌ **mfa_user_id 无显式超时**：依赖浏览器会话 Cookie
❌ **无审计日志**：MFA 验证成功/失败无日志记录
❌ **Rack::Attack 不保护 MFA**：`/mfa/verify_code` 路径无限流

---

## 十一、待确认项（未在代码审计中验证）

| 待确认项 | 说明 | 建议 |
|---------|------|------|
| Doorkeeper Token 有效期 | API 登录后的 OAuth Token 有效期 | 核查 config/initializers/doorkeeper.rb |
| Rails Session 过期策略 | `session[:mfa_user_id]` 的最大存活时间 | 核查 config/initializers/session_store.rb |
| Rack::Attack 保护范围 | 是否应增加对 mfa/verify_code 的限流 | 评估是否需要补充配置 |
