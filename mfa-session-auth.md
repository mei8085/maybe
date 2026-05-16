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

## 三、会话恢复接入点

### 3.1 Web 登录流程

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

### 3.2 关键接入点代码

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

### 3.3 API 登录流程

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

## 四、失败次数与跳过场景处理

### 4.1 失败次数限制

#### Web 端：无显式 MFA 失败次数限制

当前实现中，MFA 验证失败没有数据库级别的失败次数计数和锁定机制：

```ruby
# mfa_controller.rb:36-38
else
  flash.now[:alert] = t(".invalid_code")
  render :verify, status: :unprocessable_entity
end
```

**保护机制**：
- Rack::Attack 对 `/oauth/token` 有限制：10 次/分钟（IP 级）
- 但 `/mfa/verify_code` 路径未在 Rack::Attack 中单独配置

#### API 端：Rack::Attack 限流

**代码位置**：`config/initializers/rack_attack.rb:8-10`

```ruby
throttle("oauth/token", limit: 10, period: 1.minute) do |request|
  request.ip if request.path == "/oauth/token"
end
```

### 4.2 跳过场景：备份验证码

备份验证码是官方设计的"跳过"正常 TOTP 验证的机制：

**代码位置**：`app/models/user.rb:147-151, 194-206`

```ruby
def verify_otp?(code)
  return false if otp_secret.blank?
  return true if verify_backup_code?(code)  # 优先验证备份码
  totp.verify(code, drift_behind: 15)
end

def verify_backup_code?(code)
  return false if otp_backup_codes.blank?

  # 找到并删除已使用的备份码
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

**备份码特点**：
- 共 8 个，启用 MFA 时生成
- 一次性使用，验证成功后立即从数组中删除
- 优先级高于 TOTP 验证码
- 无时间窗口限制

### 4.3 其他"跳过"机制

根据代码分析，**没有**发现以下跳过机制：
- ❌ 可信设备（Remember Device）跳过
- ❌ IP 白名单跳过
- ❌ 管理员强制跳过
- ❌ 首次登录豁免

---

## 五、完整数据流图

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
│ 用户输入 OTP 验证码                │
└────────┬──────────────────────────┘
         │
         ▼
┌───────────────────────────────────┐
│ MfaController#verify_code         │
│ 1. 验证备份码（优先）              │
│ 2. 验证 TOTP 码                   │
└────────┬──────────────────────────┘
         │
         ├────────── 验证失败 ───────────┐
         │                                │
         ▼                                ▼
┌──────────────────────┐       ┌─────────────────────┐
│ session.delete       │       │ 重新显示验证页       │
│   (:mfa_user_id)     │       │ 显示错误信息         │
└────────┬──────────────┘       └─────────────────────┘
         │
         ▼
┌──────────────────────┐
│ create_session_for   │
│ 生成正式会话         │
└────────┬──────────────┘
         │
         ▼
┌──────────────────────┐
│ redirect root_path   │
└──────────────────────┘
```

---

## 六、关键文件索引

| 文件路径 | 说明 |
|---------|------|
| `app/models/user.rb:124-210` | MFA 配置、验证逻辑 |
| `app/controllers/sessions_controller.rb` | 登录入口，MFA 跳转逻辑 |
| `app/controllers/mfa_controller.rb` | MFA 验证页面与逻辑 |
| `app/controllers/concerns/authentication.rb` | 会话创建、Cookie 管理 |
| `app/controllers/api/v1/auth_controller.rb` | API 登录 MFA 处理 |
| `config/initializers/rack_attack.rb` | 请求限流配置 |
| `db/schema.rb:789-791` | users 表 MFA 字段定义 |

---

## 七、安全特性总结

✅ **HttpOnly Cookie**：会话令牌无法被 JS 读取
✅ **TOTP 时间漂移**：允许 ±15 秒容错（drift_behind: 15）
✅ **备份码一次性**：使用后立即删除
✅ **Rack::Attack 限流**：API 端点速率限制
❌ **MFA 失败无锁定**：Web 端 MFA 验证失败次数无限制
❌ **无 Remember Device**：每次登录都需 MFA 验证
