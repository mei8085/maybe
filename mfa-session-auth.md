# MFA 二次验证会话认证机制分析

## 一、概述

本报告分析 Maybe Finance 项目中 MFA（多因素认证）二次验证嵌入登录会话的实现机制，重点说明验证状态存储位置、会话恢复接入点、失败次数处理及跳过场景。

---

## 二、验证状态存储位置

### 2.1 用户 MFA 配置持久化存储（User 表）

MFA 配置存储在 `users` 表的以下字段中：

| 字段名 | 类型 | 说明 |
|--------|------|------|
| `otp_secret` | string | TOTP 密钥，BASE32 编码 |
| `otp_required` | boolean | 是否启用 MFA，默认 false |
| `otp_backup_codes` | string[] | 备份验证码数组，共 8 个 |

**相关代码位置**：`app/models/user.rb:124-210`

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
    otp_backup_codes: generate_backup_codes  # 生成8个备份码
  )
end
```

### 2.2 登录过程中的临时状态存储（Session）

在密码验证成功、MFA 验证完成前，用户 ID 临时存储在 Rails session 中：

```ruby
session[:mfa_user_id] = user.id
```

**代码位置**：`app/controllers/sessions_controller.rb:12-14`

### 2.3 最终认证会话存储（Session 表 + Cookie）

MFA 验证通过后，创建正式会话：

1. **数据库层**：`sessions` 表记录，关联 user_id、user_agent、ip_address
2. **客户端层**：`cookies.signed[:session_token]` 存储会话 ID，永久有效、HttpOnly

**代码位置**：`app/controllers/concerns/authentication.rb:40-44`

```ruby
def create_session_for(user)
  session = user.sessions.create!
  cookies.signed.permanent[:session_token] = { value: session.id, httponly: true }
  session
end
```

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
│  │    └─ 否 → 重定向到登录页                             │   │
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
│  │                                                        │   │
│  │   → 通过 Current.session.user 关联到 User 记录         │   │
│  └─────────────────────────────────────────────────────┘   │
└───────────────────────────────┬─────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────┐
│  业务逻辑层访问 Current.user                                  │
│  例如：set_default_chat、权限检查、数据过滤等                  │
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
        redirect_to new_session_url       # 无有效会话，登录
      end
    end

    def find_session_by_cookie
      cookie_value = cookies.signed[:session_token]
      cookie_value.present? ? Session.find_by(id: cookie_value) : nil
    end
end
```

### 3.3 会话恢复的关键特性

| 特性 | 说明 |
|------|------|
| **无状态设计** | 每次请求独立从 Cookie 还原，不依赖服务器内存状态 |
| **请求级隔离** | 使用 `ActiveSupport::CurrentAttributes`，线程安全 |
| **模拟用户支持** | `Current.user` 优先返回被模拟用户，`true_user` 返回真实用户 |
| **Cookie 安全** | signed + permanent + httponly 三重保护 |

---

## 四、MFA 流程接入点

### 4.1 Web 登录流程

```
用户输入密码
     ↓
SessionsController#create
     ↓
密码验证成功？
     ├─ 否 → 返回登录页，显示错误
     └─ 是 → 检查 user.otp_required?
              ├─ 否 → 直接 create_session_for，完成登录
              └─ 是 → session[:mfa_user_id] = user.id
                        ↓
                   重定向到 /mfa/verify
                        ↓
                   MfaController#verify
                        ↓
                   用户输入验证码
                        ↓
                   MfaController#verify_code
                        ↓
                   验证成功？
                        ├─ 否 → 重新显示验证页
                        └─ 是 → session.delete(:mfa_user_id)
                                  ↓
                             create_session_for
                                  ↓
                             重定向到首页
```

### 4.2 关键接入点代码

**接入点1：密码验证后跳转 MFA**
`app/controllers/sessions_controller.rb:10-18`

```ruby
def create
  if user = User.authenticate_by(email: params[:email], password: params[:password])
    if user.otp_required?
      session[:mfa_user_id] = user.id  # 保存状态
      redirect_to verify_mfa_path      # 跳转到MFA验证页
    else
      @session = create_session_for(user)
      redirect_to root_path
    end
  else
    flash.now[:alert] = t(".invalid_credentials")
    render :new, status: :unprocessable_entity
  end
end
```

**接入点2：MFA 验证页恢复用户**
`app/controllers/mfa_controller.rb:21-27`

```ruby
def verify
  @user = User.find_by(id: session[:mfa_user_id])

  if @user.nil?
    redirect_to new_session_path  # 状态丢失，返回登录
  end
end
```

**接入点3：MFA 验证成功后创建会话**
`app/controllers/mfa_controller.rb:29-40`

```ruby
def verify_code
  @user = User.find_by(id: session[:mfa_user_id])

  if @user&.verify_otp?(params[:code])
    session.delete(:mfa_user_id)  # 清理临时状态
    @session = create_session_for(@user)  # 创建正式会话
    redirect_to root_path
  else
    flash.now[:alert] = t(".invalid_code")
    render :verify, status: :unprocessable_entity
  end
end
```

### 4.3 API 登录流程

API 登录与 Web 登录不同，MFA 验证在**同一请求**内完成：

**代码位置**：`app/controllers/api/v1/auth_controller.rb:64-100`

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
        }, status: :unauthorized
        return
      end
    end

    # 创建 OAuth Token
    device = create_or_update_device(user)
    token_response = create_oauth_token_for_device(user, device)

    render json: token_response.merge(user: ...)
  else
    render json: { error: "Invalid email or password" }, status: :unauthorized
  end
end
```

**API 会话恢复特点**：
- 无中间状态，密码 + OTP 一次性验证
- 失败直接返回 401，客户端需重新发起请求
- 使用 Doorkeeper OAuth Token，有效期 30 天

---

## 五、验证码连续失败处理

### 5.1 失败时 mfa_user_id 的保留策略

**当前实现：验证失败时，mfa_user_id 始终保留**

`app/controllers/mfa_controller.rb:35-39`

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

### 5.2 连续失败的完整处理流程

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
│    → "Invalid code"                     │
└─────────────────────┬───────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────┐
│ 3. 重新渲染 verify 模板                  │
│    → HTTP 422 状态码                     │
│    → 表单保留用户输入（非密码，是 OTP）  │
└─────────────────────┬───────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────┐
│ 4. 用户可再次尝试输入验证码               │
│    → 无次数上限                          │
│    → 无锁定机制                          │
│    → 无超时失效                          │
└─────────────────────────────────────────┘
```

### 5.3 mfa_user_id 的失效机制

**显式失效场景**：

| 场景 | 触发方式 | 处理结果 |
|------|----------|----------|
| **验证成功** | `session.delete(:mfa_user_id)` | 状态清理，创建正式会话 |
| **状态丢失** | `session[:mfa_user_id] == nil` | 重定向到登录页 `new_session_path` |
| **用户不存在** | `User.find_by(...) == nil` | 重定向到登录页 |

**隐式失效场景**（Rails Session 默认行为）：

| 场景 | 说明 |
|------|------|
| **Session 过期** | Rails 默认 `:expire_after` 未设置（随浏览器会话） |
| **Cookie 被清除** | 用户手动清除浏览器 Cookie |
| **Session 重置** | 新登录流程覆盖 session |

**状态丢失保护代码**：
`app/controllers/mfa_controller.rb:24-26`

```ruby
def verify
  @user = User.find_by(id: session[:mfa_user_id])

  if @user.nil?
    redirect_to new_session_path  # 状态丢失时强制重新登录
  end
end
```

### 5.4 失败重定向策略汇总

| 失败类型 | 重定向目标 | 是否保留 mfa_user_id | HTTP 状态码 |
|----------|-----------|---------------------|------------|
| 密码验证失败 | 登录页（render） | 不创建 | 422 |
| MFA 验证码错误 | MFA 验证页（render） | 保留 | 422 |
| mfa_user_id 丢失 | 登录页（redirect） | 已不存在 | 302 |
| 用户记录不存在 | 登录页（redirect） | 自动失效 | 302 |

---

## 六、备份码跳过机制与会话落点

### 6.1 备份码验证的优先级

**备份码是官方设计的"跳过"TOTP 验证机制，优先级更高**

`app/models/user.rb:147-151`

```ruby
def verify_otp?(code)
  return false if otp_secret.blank?
  return true if verify_backup_code?(code)  # ✅ 优先验证备份码
  totp.verify(code, drift_behind: 15)       # 其次才是 TOTP
end
```

### 6.2 备份码验证的状态变更流程

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
│        │ 1. 查找 code 在数组中位置   │  │
│        │ 2. 找到？                    │  │
│        │    ├─ 否 → 返回 false       │  │
│        │    └─ 是 → 从数组删除 code  │  │
│        │           → update_column   │  │
│        │           → 返回 true       │  │
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
│                         │ │ 状态保留，可重试    │
└────────────┬────────────┘ └─────────────────────┘
             │
             ▼
┌─────────────────────────┐
│ create_session_for(user)│
│ ┌─────────────────────┐ │
│ │ 1. sessions.create! │ │
│ │    → 写入 sessions  │ │
│ │    → user_id 关联   │ │
│ │    → user_agent     │ │
│ │    → ip_address     │ │
│ │                     │ │
│ │ 2. 设置 Cookie      │ │
│ │    → signed         │ │
│ │    → permanent      │ │
│ │    → httponly       │ │
│ └─────────────────────┘ │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│ redirect_to root_path   │
│ → 登录完成，进入应用    │
└─────────────────────────┘
```

### 6.3 备份码验证后的会话落点

**二次验证状态清理**：
- 验证成功后立即执行 `session.delete(:mfa_user_id)`
- 临时的 MFA 等待状态被完全清除
- 不再有"部分认证"的中间状态

**最终会话落点**：
| 会话层级 | 落点位置 | 说明 |
|---------|---------|------|
| **数据库层** | `sessions` 表新记录 | `user_id` 指向认证用户 |
| **Cookie 层** | `cookies.signed[:session_token]` | 存储 `session.id`，永久有效 |
| **Current 上下文** | `Current.session` | 下次请求时通过 `authenticate_user!` 注入 |
| **用户访问** | 首页 `root_path` | 重定向进入应用主界面 |

### 6.4 备份码 vs TOTP 验证的区别

| 维度 | 备份码验证 | TOTP 验证码验证 |
|------|-----------|----------------|
| **优先级** | 高（先检查） | 低（后检查） |
| **时间限制** | 无（永久有效直到使用） | 30秒窗口（±15秒容错） |
| **使用次数** | 一次性（使用即删） | 无限次（可重复使用不同码） |
| **数据库写入** | 是（删除备份码） | 否（纯计算验证） |
| **会话落点** | 完全相同 | 完全相同 |
| **失败处理** | 完全相同 | 完全相同 |

---

## 七、完整数据流图

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
         ├─────────── 验证失败 ────────────┐
         │                                  │
         ▼                                  ▼
┌──────────────────────┐         ┌───────────────────┐
│ user.otp_required?   │         │ 返回登录页         │
│ 检查是否启用 MFA     │         │ 显示错误信息       │
└────────┬─────────────┘         └───────────────────┘
         │
         ├────────── 未启用 ────────┐
         │                           │
         ▼                           ▼
┌──────────────────────┐  ┌─────────────────────┐
│ session[:mfa_user_id]│  │ create_session_for  │
│ = user.id            │  │ 生成正式会话        │
└────────┬─────────────┘  └──────────┬──────────┘
         │                           │
         ▼                           ▼
┌──────────────────────┐  ┌─────────────────────┐
│ redirect /mfa/verify │  │  redirect root_path │
└────────┬─────────────┘  └─────────────────────┘
         │
         ▼
┌───────────────────────────────────┐
│ MfaController#verify              │
│ 从 session[:mfa_user_id] 恢复用户 │
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
│    → 30秒窗口，±15秒漂移          │
└────────┬──────────────────────────┘
         │
         ├────────── 验证失败 ───────────┐
         │                                │
         ▼                                ▼
┌──────────────────────┐       ┌─────────────────────┐
│ session.delete       │       │ 重新显示验证页       │
│   (:mfa_user_id)     │       │ mfa_user_id 保留    │
│ 清理临时状态         │       │ 显示错误信息         │
└────────┬──────────────┘       └─────────────────────┘
         │
         ▼
┌──────────────────────┐
│ create_session_for   │
│ 1. Session 表插入    │
│ 2. Cookie 设置       │
└────────┬──────────────┘
         │
         ▼
┌──────────────────────┐
│ redirect root_path   │
│ → 后续请求通过        │
│   session_token      │
│   还原 Current.user  │
└──────────────────────┘
```

---

## 八、关键文件索引

| 文件路径 | 说明 |
|---------|------|
| `app/models/current.rb` | 请求上下文存储，session 到 user 的关联 |
| `app/models/user.rb:124-210` | MFA 配置、验证逻辑（含备份码） |
| `app/models/session.rb` | 会话模型，belongs_to :user |
| `app/controllers/sessions_controller.rb` | 登录入口，MFA 跳转逻辑 |
| `app/controllers/mfa_controller.rb` | MFA 验证页面与逻辑 |
| `app/controllers/concerns/authentication.rb` | 会话创建、Cookie 管理、认证链路 |
| `app/controllers/api/v1/auth_controller.rb` | API 登录 MFA 处理 |
| `config/initializers/rack_attack.rb` | 请求限流配置 |
| `db/schema.rb:789-791` | users 表 MFA 字段定义 |

---

## 九、安全特性总结

✅ **HttpOnly Cookie**：会话令牌无法被 JS 读取
✅ **Signed Cookie**：会话 ID 防篡改
✅ **TOTP 时间漂移**：允许 ±15 秒容错（drift_behind: 15）
✅ **备份码一次性**：使用后立即从数据库删除
✅ **Rack::Attack 限流**：API 端点速率限制
✅ **无状态会话恢复**：每次请求独立还原，不依赖服务器状态

❌ **MFA 失败无锁定**：Web 端 MFA 验证失败次数无限制，无暴力破解保护
❌ **无 Remember Device**：每次登录都需 MFA 验证，无可信设备豁免
❌ **mfa_user_id 无超时**：MFA 中间状态永久有效直到成功/丢失
❌ **无审计日志**：MFA 验证成功/失败无日志记录
