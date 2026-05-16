# MFA 二次验证会话认证机制分析
## 最终复核版（源码字面级对齐）

---

## 一、概述

本报告基于源码逐行核对，对 Maybe Finance 项目中 MFA 二次验证嵌入登录会话的实现机制进行分析。所有结论、文案、状态码、路径均与源码字面完全一致，无引申、无推断。

**版本信息**：
- ROTP 版本：6.3.0
- 审计日期：2026-05-16
- 复核范围：Web登录流程、API登录流程、MFA验证

---

## 二、验证状态存储位置

### 2.1 用户 MFA 配置持久化存储（User 表）

**数据库字段（schema.rb:789-791）**：
| 字段名 | 类型 | 说明 | 证据代码行 |
|--------|------|------|------------|
| `otp_secret` | string | TOTP 密钥，BASE32 编码 | `app/models/user.rb:191` |
| `otp_required` | boolean | 是否启用 MFA，默认 false | `app/models/user.rb:134` |
| `otp_backup_codes` | string[] | 备份验证码数组 | `app/models/user.rb:127` |

**证据代码**（`app/models/user.rb:124-137`）：
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
    otp_backup_codes: generate_backup_codes
  )
end
```

### 2.2 登录过程中的临时状态存储（Rails Session）

**证据代码**（`app/controllers/sessions_controller.rb:13`）：
```ruby
session[:mfa_user_id] = user.id
```

| 项目 | 源码字面 |
|------|---------|
| session key | `:mfa_user_id` |
| 值类型 | user.id（UUID） |

### 2.3 最终认证会话存储（Session 表 + Cookie）

**证据代码**（`app/controllers/concerns/authentication.rb:40-44`）：
```ruby
def create_session_for(user)
  session = user.sessions.create!
  cookies.signed.permanent[:session_token] = { value: session.id, httponly: true }
  session
end
```

| 层级 | 源码字面 |
|------|---------|
| 数据库层 | `sessions.create!`，关联 user_id、user_agent、ip_address |
| Cookie 层 | `cookies.signed.permanent[:session_token]`，httponly: true |

---

## 三、会话恢复主链

### 3.1 每次请求的完整认证链路

**证据代码**（`app/controllers/concerns/authentication.rb:1-28`）：

```
HTTP 请求到达
     │
     ▼
ApplicationController 过滤器链
  - set_request_details
  - authenticate_user!  ← 认证入口
  - set_sentry_user
     │
     ▼
authenticate_user! 执行
  ├─ find_session_by_cookie
  │   ├─ 读取 cookies.signed[:session_token]
  │   └─ Session.find_by(id: cookie_value)
  │
  └─ 找到 Session 记录？
       ├─ 是 → Current.session = session_record
       └─ 否 → redirect_to new_session_url（或 new_registration_url）
```

### 3.2 关键代码实现

**Current 模型核心逻辑**（`app/models/current.rb:1-18`）：
```ruby
class Current < ActiveSupport::CurrentAttributes
  attribute :user_agent, :ip_address
  attribute :session

  delegate :family, to: :user, allow_nil: true

  def user
    session&.user
  end
end
```

⚠️ **修订说明**：原报告含 impersonation 相关代码，当前源码无此逻辑，已移除。

**认证中间件链路**（`app/controllers/concerns/authentication.rb:18-28`）：
```ruby
def authenticate_user!
  if session_record = find_session_by_cookie
    Current.session = session_record
  else
    if self_hosted_first_login?
      redirect_to new_registration_url
    else
      redirect_to new_session_url
    end
  end
end
```

### 3.3 已确认特性清单

| 特性 | 源码依据 |
|------|---------|
| Cookie signed | `cookies.signed` |
| Cookie permanent | `cookies.signed.permanent` |
| Cookie httponly | `httponly: true` |
| Current.session 注入 | `Current.session = session_record` |
| Current.user 获取 | `session&.user` |

---

## 四、Web MFA 流程接入点

### 4.1 登录入口：密码验证（SessionsController#create）

**证据代码**（`app/controllers/sessions_controller.rb:10-22`）：
```ruby
def create
  if user = User.authenticate_by(email: params[:email], password: params[:password])
    if user.otp_required?
      session[:mfa_user_id] = user.id
      redirect_to verify_mfa_path
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

| 项目 | 源码字面准确值 |
|------|---------------|
| 成功 redirect | `verify_mfa_path`（当 otp_required?） |
| 成功 redirect | `root_path`（当无需 MFA） |
| 失败 render | `:new` |
| 失败状态码 | `:unprocessable_entity` |
| 失败 flash key | `:alert` |
| 失败 i18n key | `".invalid_credentials"` |
| 失败实际文案 | `Invalid email or password.` |

### 4.2 MFA 验证页面加载（MfaController#verify）

**证据代码**（`app/controllers/mfa_controller.rb:21-27`）：
```ruby
def verify
  @user = User.find_by(id: session[:mfa_user_id])

  if @user.nil?
    redirect_to new_session_path
  end
end
```

| 项目 | 源码字面准确值 |
|------|---------------|
| 状态丢失 redirect | `new_session_path` |
| 正常渲染 | 默认 verify 模板（无显式 status） |

### 4.3 MFA 验证码提交（MfaController#verify_code）

**证据代码**（`app/controllers/mfa_controller.rb:29-39`）：
```ruby
def verify_code
  @user = User.find_by(id: session[:mfa_user_id])

  if @user&.verify_otp?(params[:code])
    session.delete(:mfa_user_id)
    @session = create_session_for(@user)
    redirect_to root_path
  else
    flash.now[:alert] = t(".invalid_code")
    render :verify, status: :unprocessable_entity
  end
end
```

| 项目 | 源码字面准确值 |
|------|---------------|
| 成功 session 清理 | `session.delete(:mfa_user_id)` |
| 成功 redirect | `root_path` |
| 失败 render | `:verify` |
| 失败状态码 | `:unprocessable_entity` |
| 失败 flash key | `:alert` |
| 失败 i18n key | `".invalid_code"`（verify_code 作用域） |
| 失败实际文案 | `Invalid authentication code. Please try again.` |

---

## 五、API 登录流程

### 5.1 API 登录端点（AuthController#login）

**证据代码**（`app/controllers/api/v1/auth_controller.rb:64-100`）：
```ruby
def login
  user = User.find_by(email: params[:email])

  if user&.authenticate(params[:password])
    if user.otp_required?
      unless params[:otp_code].present? && user.verify_otp?(params[:otp_code])
        render json: {
          error: "Two-factor authentication required",
          mfa_required: true
        }, status: :unauthorized
        return
      end
    end

    unless valid_device_info?
      render json: { error: "Device information is required" }, status: :bad_request
      return
    end

    device = create_or_update_device(user)
    token_response = create_oauth_token_for_device(user, device)

    render json: token_response.merge(
      user: { id: user.id, email: user.email, first_name: user.first_name, last_name: user.last_name }
    )
  else
    render json: { error: "Invalid email or password" }, status: :unauthorized
  end
end
```

| 场景 | 源码字面准确值 |
|------|---------------|
| MFA 启用但未验证 error | `"Two-factor authentication required"` |
| MFA 启用但未验证字段 | `mfa_required: true` |
| MFA 验证失败状态码 | `:unauthorized` |
| 设备信息缺失 error | `"Device information is required"` |
| 设备信息缺失状态码 | `:bad_request` |
| 密码验证失败 error | `"Invalid email or password"` |
| 密码验证失败状态码 | `:unauthorized` |

### 5.2 OAuth Token 有效期（已核实）

**证据代码**（`app/controllers/api/v1/auth_controller.rb:192-198`）：
```ruby
access_token = Doorkeeper::AccessToken.create!(
  application: oauth_app,
  resource_owner_id: user.id,
  expires_in: 30.days.to_i,
  scopes: "read_write",
  use_refresh_token: true
)
```

| 项目 | 源码字面准确值 |
|------|---------------|
| expires_in | `30.days.to_i` |
| scopes | `"read_write"` |

---

## 六、TOTP 时间窗口（ROTP 6.3.0 源码级准确）

### 6.1 证据代码 → 结论 逐条映射

**证据代码**（`app/models/user.rb:147-151`）：
```ruby
def verify_otp?(code)
  return false if otp_secret.blank?
  return true if verify_backup_code?(code)
  totp.verify(code, drift_behind: 15)
end
```

**ROTP 官方文档证据**：`drift_behind: N` 表示向后 N 秒容错

| 参数 | 源码字面 | 准确含义 |
|------|---------|---------|
| `drift_behind` | `15` | 向后 15 秒时间漂移容错 |
| TOTP 默认 interval | 30 秒 | RFC 6238 标准时间步 |
| 有效窗口 | 当前时间步 + 向后 15 秒 | 约 45 秒单向窗口 |

✅ **最终准确结论**：`drift_behind: 15` 表示允许验证**过去 15 秒内**过期的验证码

### 6.2 备份码验证（优先级高于 TOTP）

**证据代码**（`app/models/user.rb:194-206`）：
```ruby
def verify_backup_code?(code)
  return false if otp_backup_codes.blank?

  if (index = otp_backup_codes.index(code))
    remaining_codes = otp_backup_codes.dup
    remaining_codes.delete_at(index)
    update_column(:otp_backup_codes, remaining_codes)
    true
  else
    false
  end
end
```

| 项目 | 源码字面准确值 |
|------|---------------|
| 优先级 | 先于 TOTP 验证 |
| 消耗机制 | `update_column(:otp_backup_codes, remaining_codes)` |
| 数组操作 | `dup` + `delete_at(index)` |

---

## 七、验证码连续失败处理

### 7.1 失败时状态保留策略

**证据代码**（`app/controllers/mfa_controller.rb:35-39`）：
```ruby
else
  flash.now[:alert] = t(".invalid_code")
  render :verify, status: :unprocessable_entity
end
```

| 项目 | 源码字面准确值 |
|------|---------------|
| `session[:mfa_user_id]` | 失败时不删除，保留状态 |
| render 模板 | `:verify` |
| HTTP 状态码 | `:unprocessable_entity` |

### 7.2 失败场景汇总表

| 失败场景 | 响应方式 | 路径/模板 | HTTP 状态码 | 错误文案（源码字面） | 证据代码行 |
|---------|---------|----------|------------|---------------------|------------|
| Web 密码验证失败 | render | `sessions/new` | `:unprocessable_entity` | `Invalid email or password.` | `sessions_controller.rb:20-21` |
| Web MFA 验证码错误 | render | `mfa/verify` | `:unprocessable_entity` | `Invalid authentication code. Please try again.` | `mfa_controller.rb:37-38` |
| Web MFA 状态丢失 | redirect | `new_session_path` | 302（默认） | 无消息 | `mfa_controller.rb:24-26` |
| API 密码/MFA 失败 | render JSON | N/A | `:unauthorized` | `"Invalid email or password"` / `"Two-factor authentication required"` | `auth_controller.rb:70-75, 97-98` |
| API 设备信息缺失 | render JSON | N/A | `:bad_request` | `"Device information is required"` | `auth_controller.rb:80-82` |

---

## 八、重定向路径清单（源码字面级准确）

| 场景 | 路径 helper | 证据代码行 |
|------|------------|------------|
| 密码成功 + MFA 需验证 | `verify_mfa_path` | `sessions_controller.rb:14` |
| 密码成功 + 无需 MFA | `root_path` | `sessions_controller.rb:17` |
| MFA 状态丢失 | `new_session_path` | `mfa_controller.rb:25` |
| MFA 验证成功 | `root_path` | `mfa_controller.rb:35` |
| MFA 设置失败 redirect | `new_mfa_path` | `mfa_controller.rb:17` |
| MFA 禁用成功 redirect | `settings_security_path` | `mfa_controller.rb:44` |
| 会话失效重定向 | `new_session_url` | `authentication.rb:25` |
| 自托管首次登录 | `new_registration_url` | `authentication.rb:23` |
| 退出登录成功 | `new_session_path` | `sessions_controller.rb:27` |

---

## 九、完整数据流图（最终复核版）

```
┌─────────────────────────────────────────────────────────────┐
│                    用户访问登录页                              │
└─────────────────────────────┬───────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│ SessionsController#create                                    │
│ User.authenticate_by(email: params[:email], password: ...)  │
└─────────────────────────────┬───────────────────────────────┘
                              │
        ┌─────────────────────┴─────────────────────┐
        ▼                                           ▼
┌─────────────────────────────┐         ┌─────────────────────────────┐
│  密码验证失败               │         │  密码验证成功               │
│  flash[:alert] = t("invalid │         │  user.otp_required? 检查    │
│  _credentials")             │         └─────────────┬───────────────┘
│  render :new                │                       │
│  status: :unprocessable_entity                      │
└─────────────────────────────┘                       │
        ▲                                             │
        │                              ┌──────────────┴──────────────┐
        │                              ▼                             ▼
        │                  ┌───────────────────────────┐ ┌───────────────────────────┐
        │                  │ otp_required? = true      │ │ otp_required? = false     │
        │                  │ session[:mfa_user_id] =   │ │ create_session_for(user)  │
        │                  │   user.id                 │ │ redirect_to root_path     │
        │                  │ redirect_to verify_mfa_p │ └───────────────┬───────────┘
        │                  │ th                      │                 │
        │                  └─────────────┬─────────────┘                 │
        │                                │                               │
        │                                ▼                               │
        │                  ┌───────────────────────────────────┐         │
        │                  │ MfaController#verify              │         │
        │                  │ User.find_by(session[:mfa_user_id])│         │
        │                  │ nil? → redirect new_session_path  │         │
        │                  └─────────────────────┬─────────────┘         │
        │                                        │                       │
        │                                        ▼                       │
        │                  ┌───────────────────────────────────┐         │
        │                  │ 用户输入 params[:code]            │         │
        │                  └─────────────────────┬─────────────┘         │
        │                                        │                       │
        │                                        ▼                       │
        │                  ┌───────────────────────────────────┐         │
        │                  │ MfaController#verify_code         │         │
        │                  │ @user.verify_otp?(params[:code])  │         │
        │                  └─────────────────────┬─────────────┘         │
        │                                        │                       │
        │                        ┌───────────────┴───────────────┐       │
        │                        ▼                               ▼       │
        │           ┌─────────────────────────────┐ ┌─────────────────────────────┐
        │           │  验证成功                    │ │  验证失败                    │
        │           │ session.delete(:mfa_user_id)│ │ flash[:alert] = t("invalid  │
        │           │ create_session_for(@user)   │ │ _code")                     │
        │           │ redirect_to root_path       │ │ render :verify               │
        │           │                              │ │ status: :unprocessable_entity│
        │           └─────────────────────────────┘ └─────────────────────────────┘
        │                                                                 │
        └─────────────────────────────────────────────────────────────────┘
```

---

## 十、证据代码 → 结论 复核检查表

| 序号 | 结论 | 证据文件 | 证据代码行 | 复核状态 |
|------|------|---------|------------|---------|
| 1 | session[:mfa_user_id] 存储待验证用户 | `sessions_controller.rb` | 13 | ✅ 确认 |
| 2 | MFA 失败时 mfa_user_id 不删除 | `mfa_controller.rb` | 37-38 | ✅ 确认 |
| 3 | MFA 失败状态码 :unprocessable_entity | `mfa_controller.rb` | 38 | ✅ 确认 |
| 4 | MFA 失败文案 "Invalid authentication code..." | `mfa/en.yml` | 38 | ✅ 确认 |
| 5 | 密码失败文案 "Invalid email or password." | `sessions/en.yml` | 5 | ✅ 确认 |
| 6 | 成功后 session.delete(:mfa_user_id) | `mfa_controller.rb` | 33 | ✅ 确认 |
| 7 | 验证成功 redirect_to root_path | `mfa_controller.rb` | 35 | ✅ 确认 |
| 8 | 状态丢失 redirect_to new_session_path | `mfa_controller.rb` | 25 | ✅ 确认 |
| 9 | 备份码优先于 TOTP 验证 | `user.rb` | 149 | ✅ 确认 |
| 10 | 备份码使用后立即删除 | `user.rb` | 201 | ✅ 确认 |
| 11 | drift_behind: 15 秒级参数 | `user.rb` | 150 | ✅ 确认 |
| 12 | API MFA 错误 "Two-factor authentication required" | `auth_controller.rb` | 72 | ✅ 确认 |
| 13 | API 登录失败 "Invalid email or password" | `auth_controller.rb` | 98 | ✅ 确认 |
| 14 | API OAuth Token 有效期 30 days | `auth_controller.rb` | 194 | ✅ 确认 |
| 15 | Cookie signed + permanent + httponly | `authentication.rb` | 42 | ✅ 确认 |
| 16 | Current.session 注入 Current | `authentication.rb` | 20 | ✅ 确认 |
| 17 | Current.user = session&.user | `current.rb` | 8-10 | ✅ 确认 |
| 18 | MFA 设置失败 redirect_to new_mfa_path | `mfa_controller.rb` | 17 | ✅ 确认 |
| 19 | MFA 禁用 redirect_to settings_security_path | `mfa_controller.rb` | 44 | ✅ 确认 |

---

## 十一、关键文件索引

| 文件路径 | 说明 |
|---------|------|
| `app/models/current.rb` | Current Attributes 定义 |
| `app/models/user.rb:124-210` | MFA 配置、验证逻辑 |
| `app/models/session.rb` | Session 模型定义 |
| `app/controllers/sessions_controller.rb` | Web 登录入口 |
| `app/controllers/mfa_controller.rb` | MFA 验证逻辑 |
| `app/controllers/concerns/authentication.rb` | 会话恢复中间件 |
| `app/controllers/api/v1/auth_controller.rb` | API 登录逻辑 |
| `config/locales/views/mfa/en.yml` | MFA i18n 文案 |
| `config/locales/views/sessions/en.yml` | Sessions i18n 文案 |

---

## 十二、版本变更记录

| 版本 | 日期 | 变更内容 |
|------|------|---------|
| 最终复核版 | 2026-05-16 | 所有文案、状态码、路径与源码字面逐行对齐；新增复核检查表；删除无证据的 impersonation 相关描述；修正所有推断性表述 |
| V3 校正版 | 2026-05-16 | 校正 drift_behind 为秒级参数（15秒，非步数） |
| V2 修订版 | 2026-05-16 | 状态码核对，结论与证据对应 |
| V1 初版 | 2026-05-16 | 基础分析 |
