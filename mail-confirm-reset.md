# 邮箱确认与密码重置链路实现详解

## 一、核心机制概述

本项目使用 Rails 7.1+ 内置的 `generates_token_for` 特性来实现安全的签名 token 生成与验证。该机制将**过期时间**和**有效性条件**直接编码到 token 中，无需在数据库存储额外的 token 记录。

---

## 二、邮箱确认链路 (Email Confirmation)

### 2.1 触发场景

邮箱确认在两种情况下触发：

1. **用户修改邮箱**时（`user.rb:44-58`）
2. 注册流程：当前注册成功后直接登录，未强制邮箱确认

### 2.2 链路完整流程

```
用户修改邮箱 → 生成签名token → 投递邮件队列 → 异步发送邮件 → 用户点击链接 → 验证token → 完成确认
```

### 2.3 Token 定义与签名 (`user.rb:36-38`)

```ruby
generates_token_for :email_confirmation, expires_in: 1.day do
  unconfirmed_email
end
```

**关键设计：**
- **过期时间**：1天（24小时）
- **签名 payload**：块内返回的 `unconfirmed_email` 会被包含在签名中
- **作用**：一旦 `unconfirmed_email` 发生变化（如被清空），原有 token 自动失效

### 2.4 Token 生成

在 `EmailConfirmationMailer` 中调用生成：

```ruby
# email_confirmation_mailer.rb:11
@confirmation_url = new_email_confirmation_url(token: @user.generate_token_for(:email_confirmation))
```

### 2.5 队列投递

```ruby
# user.rb:52
EmailConfirmationMailer.with(user: self).confirmation_email.deliver_later
```

**队列配置（生产环境）：**
- 队列名称：`:high_priority`（`production.rb:77`）
- 后端：Sidekiq（通过 Active Job 抽象）
- 优势：异步发送不阻塞用户请求

### 2.6 邮件模板渲染

| 层级 | 文件 | 作用 |
|------|------|------|
| Mailer 类 | `email_confirmation_mailer.rb` | 准备模板变量，构建邮件 |
| HTML 模板 | `views/email_confirmation_mailer/confirmation_email.html.erb` | 邮件内容主体 |
| 布局 | `views/layouts/mailer.html.erb` | 邮件通用布局框架 |
| 发件人配置 | `application_mailer.rb:2` | 统一发件人名称和地址 |

### 2.7 链接验证与防御机制

验证逻辑在 `email_confirmations_controller.rb:5-17`：

```ruby
def new
  @user = User.find_by_token_for(:email_confirmation, params[:token])
  if @user&.unconfirmed_email && @user&.update(email: @user.unconfirmed_email, unconfirmed_email: nil)
    redirect_to new_session_path, notice: t(".success_login")
  else
    redirect_to root_path, alert: t(".invalid_token")
  end
end
```

**过期防御：**
- `find_by_token_for` 自动检查 token 生成时间，超过 1 天返回 `nil`

**重复使用防御：**
1. **签名绑定**：token 签名包含 `unconfirmed_email` 值
2. **状态变更**：确认成功后 `unconfirmed_email` 被设为 `nil`
3. **自动失效**：下次用相同 token 验证时，由于 `unconfirmed_email` 已变，签名验证失败

---

## 三、密码重置链路 (Password Reset)

### 3.1 触发场景

用户在 "忘记密码" 页面提交邮箱地址（`password_resets_controller.rb:11-19`）

### 3.2 链路完整流程

```
用户提交邮箱 → 查找用户 → 生成签名token → 投递邮件队列 → 异步发送邮件 → 用户点击链接 → 验证token → 设置新密码
```

### 3.3 Token 定义与签名 (`user.rb:32-34`)

```ruby
generates_token_for :password_reset, expires_in: 15.minutes do
  password_salt&.last(10)
end
```

**关键设计：**
- **过期时间**：15分钟（安全性要求更高）
- **签名 payload**：`password_salt.last(10)` —— 密码盐值的后10位
- **作用**：密码一旦修改，`password_salt` 随之变化，旧 token 自动失效

### 3.4 Token 生成

在控制器中生成后传递给 Mailer：

```ruby
# password_resets_controller.rb:13-16
PasswordMailer.with(
  user: user,
  token: user.generate_token_for(:password_reset)
).password_reset.deliver_later
```

### 3.5 队列投递

与邮箱确认相同，使用 `deliver_later` 投递到 `:high_priority` 队列。

### 3.6 邮件模板渲染

| 层级 | 文件 |
|------|------|
| Mailer 类 | `password_mailer.rb` |
| HTML 模板 | `views/password_mailer/password_reset.html.erb` |
| 布局 | `views/layouts/mailer.html.erb` |

### 3.7 链接验证与防御机制

验证分两步进行：

**第一步：访问重置页面（`password_resets_controller.rb:36-39`）**
```ruby
def set_user_by_token
  @user = User.find_by_token_for(:password_reset, params[:token])
  redirect_to new_password_reset_path, alert: t("...") unless @user.present?
end
```

**第二步：提交新密码（`password_resets_controller.rb:26-31`）**
```ruby
def update
  if @user.update(password_params)
    redirect_to new_session_path, notice: t(".success")
  else
    render :edit, status: :unprocessable_entity
  end
end
```

**过期防御：**
- `find_by_token_for` 自动检查，超过 15 分钟返回 `nil`

**重复使用防御：**
1. **签名绑定**：token 签名包含 `password_salt.last(10)`
2. **密码变更**：密码更新时，`has_secure_password` 自动重新生成 `password_salt`
3. **自动失效**：盐值变化导致原有 token 签名验证失败，无法重复使用

---

## 四、`generates_token_for` 工作原理

### 4.1 Token 结构

Rails 生成的 token 包含三部分：
```
[目的标识] + [过期时间] + [用户ID] + [块返回值] → HMAC 签名 → Base64 编码
```

### 4.2 验证流程

调用 `find_by_token_for` 时：
1. 解码 token，提取各字段
2. 检查过期时间是否在有效期内
3. 根据用户ID查找记录
4. 重新执行块获取当前值，与 token 中存储的值比较
5. 验证 HMAC 签名是否有效
6. 全部通过返回用户对象，否则返回 `nil`

---

## 五、安全设计总结

| 安全特性 | 邮箱确认 | 密码重置 | 实现方式 |
|---------|---------|---------|---------|
| 过期时间 | 24小时 | 15分钟 | `expires_in` 参数 |
| 防重复使用 | ✅ | ✅ | 签名绑定可变状态 |
| 防篡改 | ✅ | ✅ | HMAC 签名 |
| 无需DB存储 | ✅ | ✅ | 状态编码到 token |
| 异步发送 | ✅ | ✅ | `deliver_later` |
| 高优先级队列 | ✅ | ✅ | `high_priority` 队列 |

### 5.1 为什么这是安全的？

1. **无状态设计**：不需要在数据库创建额外的 token 表，减少了攻击面
2. **自动失效**：通过绑定业务状态（`unconfirmed_email`、`password_salt`）实现一次性使用
3. **时间窗口小**：密码重置仅 15 分钟有效，降低泄露风险
4. **加密签名**：使用应用密钥进行 HMAC 签名，无法伪造
5. **常量时间比较**：Rails 内部使用 `ActiveSupport::SecurityUtils.secure_compare` 防止时序攻击

### 5.2 潜在注意事项

- 应用密钥（`secret_key_base`）泄露会导致 token 可伪造，需妥善保护
- token 仅在 URL 中传输，建议确保 HTTPS 已启用（生产环境已配置 `force_ssl`）
- 邮件服务商可能会预取链接进行安全扫描，这不会触发实际操作（因为需要 POST 提交新密码）
