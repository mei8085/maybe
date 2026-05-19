# 邮箱确认与密码重置链路实现详解

## 一、核心机制概述

本项目使用 Rails 7.1+ 内置的 `generates_token_for` 特性来实现安全的签名 token 生成与验证。该机制将**过期时间**和**有效性条件**直接编码到 token 中，无需在数据库存储额外的 token 记录。

---

## 二、邮箱确认链路 (Email Confirmation)

### 2.1 触发场景

⚠️ **重要纠正：注册流程不触发邮箱确认！**

邮箱确认**仅**在用户**修改已有邮箱**时触发（`user.rb:44-58`）。注册时直接保存邮箱到 `email` 字段，不设置 `unconfirmed_email`，也不发送确认邮件。

### 2.2 链路完整流程

```
用户在设置页修改邮箱 → 调用 initiate_email_change → 设置 unconfirmed_email → 生成签名token →
投递邮件队列 → 异步发送邮件 → 用户点击链接 → 验证token → 确认成功
```

### 2.3 Token 定义与签名 (`user.rb:36-38`)

```ruby
generates_token_for :email_confirmation, expires_in: 1.day do
  unconfirmed_email
end
```

**关键设计：**
- **过期时间**：1天（24小时）
- **签名 payload**：块内返回的 `unconfirmed_email` 当前值会被序列化并包含在 token 签名中
- **核心机制**：验证时会**重新执行块**，将当前返回值与 token 中存储的值对比，不一致则验证失败

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

### 2.7 Self-Hosted 模式下的特殊分支

在 `initiate_email_change` 方法（`user.rb:44-58`）中存在一个关键的条件分支：

```ruby
def initiate_email_change(new_email)
  return false if new_email == email
  return false if new_email == unconfirmed_email

  if Rails.application.config.app_mode.self_hosted? && !Setting.require_email_confirmation
    update(email: new_email)  # 直接更新，跳过确认流程
  else
    if update(unconfirmed_email: new_email)
      EmailConfirmationMailer.with(user: self).confirmation_email.deliver_later
      true
    else
      false
    end
  end
end
```

**触发条件（同时满足）：**
1. `app_mode.self_hosted?` → 由环境变量 `SELF_HOSTED=true` 或 `SELF_HOSTING_ENABLED=true` 决定（`application.rb:30`）
2. `!Setting.require_email_confirmation` → 自托管管理员在设置页关闭了邮箱确认要求

**为何不发确认邮件：**
- **设计意图**：自托管场景下，管理员可能完全控制服务器，不需要邮箱验证作为额外的安全层
- **简化流程**：跳过邮件确认可以减少对 SMTP 服务的依赖，适合内网或离线部署
- **风险权衡**：自托管用户通常只有少数几个账号，且管理员对所有账号有完全控制权，邮箱确认的安全收益较低
- **控制器联动**：`users_controller.rb:10-13` 也有对应分支，返回不同的成功提示

### 2.8 链接验证与防御机制

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

**重复使用防御（关键纠正）：**
1. **生成时**：token 签名包含 `unconfirmed_email` 的当前值（如 `"new@example.com"`）
2. **第一次验证**：`unconfirmed_email` 仍为 `"new@example.com"`，验证通过，执行 update
3. **update 后**：`unconfirmed_email` 被设为 `nil`
4. **第二次验证**：重新执行块返回 `nil`，与 token 中存储的 `"new@example.com"` 不匹配
5. **结果**：`find_by_token_for` 返回 `nil`，即使 token 还在 24 小时有效期内

✅ **结论：邮箱确认 token 是一次性的，成功使用后立即失效，无法重复点击。**

---

## 三、密码重置链路 (Password Reset)

### 3.1 触发场景

用户在 "忘记密码" 页面提交邮箱地址（`password_resets_controller.rb:11-19`）

### 3.2 邮箱查询与归一化的行为边界

**关键代码**（`password_resets_controller.rb:12`）：
```ruby
if (user = User.find_by(email: params[:email]))
```

⚠️ **重要边界：控制器直接使用原始邮箱查询，不经过模型归一化！**

#### 3.2.1 模型层的邮箱归一化

模型定义了自动归一化规则（`user.rb:18-19`）：
```ruby
normalizes :email, with: ->(email) { email.strip.downcase }
normalizes :unconfirmed_email, with: ->(email) { email&.strip&.downcase }
```

**归一化触发时机**：
- ✅ 模型 `save` / `update` 时自动调用
- ✅ 创建用户时
- ✅ 修改邮箱时
- ❌ **`find_by` 查询时不会自动应用**

#### 3.2.2 行为边界分析

| 操作 | 邮箱值 | 数据库存储 | 查询结果 |
|------|--------|-----------|---------|
| 注册 "User@Example.com" | 原始输入 | `user@example.com`（归一化后） | N/A |
| 重置密码输入 "User@Example.com" | `"User@Example.com"` | `user@example.com` | ✅ 匹配（PostgreSQL 字符串比较不区分大小写） |
| 重置密码输入 " user@example.com "（带空格） | `" user@example.com "` | `user@example.com` | ❌ **不匹配**（空格导致精确比较失败） |
| 重置密码输入 "USER@EXAMPLE.COM"（全大写） | `"USER@EXAMPLE.COM"` | `user@example.com` | ✅ 匹配（PostgreSQL CI 特性） |

**边界风险**：
用户注册时邮箱前后的空格会被自动去除，但重置密码时输入带空格的邮箱会查询不到用户，且由于统一响应设计，用户得不到任何提示。

**设计权衡**：
- 优点：控制器逻辑简单，不需要复制归一化逻辑
- 缺点：存在边缘 case 导致真实用户收不到重置邮件
- 建议优化：在控制器查询前手动应用归一化：
  ```ruby
  if (user = User.find_by(email: params[:email].to_s.strip.downcase))
  ```

### 3.3 链路完整流程

```
用户提交邮箱 → 查找用户 → 生成签名token → 投递邮件队列 → 异步发送邮件 →
用户点击链接 → 验证token（edit）→ 展示重置表单 → 提交新密码 → 验证token（update）→ 密码更新
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
- **核心机制**：验证时重新执行块，盐值变化则验证失败

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

**重复使用防御（关键行为）：**

| 阶段 | 行为 | `password_salt` | token 有效性 |
|------|------|----------------|-------------|
| 生成 token | 点击"发送重置邮件" | 盐值 A | ✅ 有效 |
| 第一次点击链接（edit） | 访问重置表单页面 | 盐值 A | ✅ 有效 |
| 第二次点击链接（edit） | 再次访问重置表单 | 盐值 A（未变） | ✅ 仍有效 |
| ...（多次点击） | 只要在 15 分钟内 | 盐值 A | ✅ 都有效 |
| 提交新密码（update） | 密码更新成功 | 盐值 B（重新生成） | ❌ 失效 |
| 再次点击链接 | 盐值已变 | 盐值 B | ❌ 失效 |

⚠️ **重要边界：在提交新密码之前，只要在 15 分钟过期时间内，可以重复点击链接多次访问重置页面。只有在密码实际修改后，token 才会失效。**

### 3.8 邮箱存在与否的统一响应设计（防邮箱枚举）

`create` 方法（`password_resets_controller.rb:11-19`）的核心逻辑：

```ruby
def create
  if (user = User.find_by(email: params[:email]))
    PasswordMailer.with(
      user: user,
      token: user.generate_token_for(:password_reset)
    ).password_reset.deliver_later
  end

  redirect_to new_password_reset_path(step: "pending")
end
```

#### 3.8.1 两种场景的完全一致行为

| 场景 | 后端行为 | 前端响应 |
|------|---------|---------|
| **邮箱存在** | 查找用户 → 生成 token → 投递邮件到队列 | `redirect_to /password_reset/new?step=pending` |
| **邮箱不存在** | 不查找用户 → 不生成 token → 不发送邮件 | **完全相同**：`redirect_to /password_reset/new?step=pending` |

**pending 页面展示内容**（`new.html.erb:3-4` + `en.yml:7`）：
```
Please check your email for a link to reset your password.
```

无论邮箱是否注册，用户看到的提示文本完全相同。

#### 3.8.2 安全意义：防止用户名枚举攻击

**什么是用户名枚举？**
攻击者通过提交不同的邮箱，根据服务器响应的差异（文本、状态码、响应时间）判断该邮箱是否已注册，从而获取有效账号列表进行后续攻击。

**本项目的防御机制：**

| 攻击向量 | 无防御的脆弱表现 | 本项目的防御效果 |
|---------|-----------------|----------------|
| **响应文本差异** | 存在→"邮件已发送"<br>不存在→"邮箱未注册" | ✅ 两种情况显示完全相同的文本 |
| **HTTP 状态码** | 存在→302 重定向<br>不存在→422 错误 | ✅ 统一返回 302 重定向 |
| **重定向目标** | 存在→pending 页<br>不存在→原表单页 | ✅ 统一重定向到 `?step=pending` |
| **响应时间差异** | 发送邮件（慢）vs 不发送（快） | ⚠️ 理论上存在时序差异，但 `deliver_later` 异步化后差异极小 |

#### 3.8.3 边界影响与设计权衡

**安全收益：**
- ✅ 有效防止批量邮箱枚举
- ✅ 保护用户隐私（不泄露谁是平台用户）
- ✅ 增加定向攻击的难度（无法预先确认目标账号存在）

**用户体验代价：**
- ❌ 用户输错邮箱时收不到邮件，也得不到任何错误提示
- ❌ 可能产生困惑："我提交了为什么没收到邮件？"
- ❌ 增加客服支持成本（用户可能联系客服反馈"收不到邮件"）

**缓解措施：**
- UI 文案本身已包含隐含提示："Please check **your email**" 暗示"如果这是您的注册邮箱"
- 真实用户通常记得自己的注册邮箱，输错概率较低
- 攻击者无法从大量尝试中筛选有效账号

#### 3.8.4 补充防御：限流与监控

⚠️ **注意**：当前 `rack_attack.rb` 中未对 `/password_reset` 端点配置专门的限流规则。理论上攻击者仍可进行无限次尝试，只是无法从响应中判断结果。

**建议的增强措施（当前未实现）：**
1. 按 IP 对 `POST /password_reset` 限流（如 5 次/分钟）
2. 对异常高频请求进行日志监控
3. 考虑引入简单的人机验证（如 hCaptcha）

---

## 四、`generates_token_for` 工作原理

### 4.1 Token 结构

Rails 生成的 token 结构：
```
[目的标识] + [过期时间] + [用户ID] + [块返回值] → HMAC 签名 → Base64 编码
```

### 4.2 验证流程

调用 `find_by_token_for` 时：
1. 解码 token，提取各字段
2. 检查过期时间是否在有效期内 → 过期则返回 `nil`
3. 根据用户ID查找记录 → 不存在则返回 `nil`
4. **重新执行块**获取当前值，与 token 中存储的值比较 → 不一致则返回 `nil`
5. 验证 HMAC 签名是否有效 → 无效则返回 `nil`
6. 全部通过返回用户对象

---

## 五、两条链路边界条件对比表

| 边界条件 | 邮箱确认链路 | 密码重置链路 |
|---------|-------------|-------------|
| 触发时机 | 仅修改邮箱时 | 忘记密码提交邮箱时 |
| 注册时触发 | ❌ 不触发 | ❌ 不触发 |
| 过期时间 | 24小时 | 15分钟 |
| 签名绑定 | `unconfirmed_email` | `password_salt.last(10)` |
| 能否重复点击 | ❌ 一次性（成功后立即失效） | ⚠️ 提交密码前可重复访问，提交后失效 |
| 失效原因 | `unconfirmed_email` 变为 `nil` | `password_salt` 重新生成 |
| 操作幂等性 | ✅ 多次点击不产生副作用 | ⚠️ edit 幂等，update 非幂等 |
| GET 请求是否修改状态 | ✅ 是（确认邮箱） | ❌ 否（仅展示表单） |
| Self-hosted 跳过分支 | ✅ 支持（关闭确认时直接更新） | ❌ 无特殊分支 |
| 对不存在账号的处理 | N/A | 静默忽略，统一重定向到 pending 页 |
| 防账号枚举 | N/A | ✅ 邮箱存在与否响应完全相同 |
| 专属限流规则 | N/A | ❌ 未配置（仅通用 API 限流） |

---

## 六、安全设计总结

| 安全特性 | 邮箱确认 | 密码重置 | 实现方式 |
|---------|---------|---------|---------|
| 过期时间 | 24小时 | 15分钟 | `expires_in` 参数 |
| 防重复使用 | ✅ 一次性 | ⚠️ 半一次性（update后失效） | 签名绑定可变状态 |
| 防篡改 | ✅ | ✅ | HMAC 签名 |
| 无需DB存储 | ✅ | ✅ | 状态编码到 token |
| 异步发送 | ✅ | ✅ | `deliver_later` |
| 高优先级队列 | ✅ | ✅ | `high_priority` 队列 |
| 账号枚举防御 | N/A | ✅ | 统一响应，不泄露邮箱存在性 |
| 时序攻击防御 | ✅ | ✅ | `secure_compare` 常量时间比较 |
| Self-hosted 灵活配置 | ✅ | ❌ | 可关闭邮箱确认要求 |

### 6.1 为什么这是安全的？

1. **无状态设计**：不需要在数据库创建额外的 token 表，减少了攻击面
2. **自动失效**：通过绑定业务状态（`unconfirmed_email`、`password_salt`）实现一次性使用
3. **时间窗口小**：密码重置仅 15 分钟有效，降低泄露风险
4. **加密签名**：使用应用密钥进行 HMAC 签名，无法伪造
5. **常量时间比较**：Rails 内部使用 `ActiveSupport::SecurityUtils.secure_compare` 防止时序攻击

### 6.2 潜在注意事项

- 应用密钥（`secret_key_base`）泄露会导致 token 可伪造，需妥善保护
- token 仅在 URL 中传输，生产环境已配置 `force_ssl` 确保 HTTPS
- 密码重置的 edit 页面可重复访问，属于设计行为（用户可能多次打开邮件）
- 邮箱确认的 GET 请求会修改状态，需注意邮件服务商的链接预取可能触发确认（但由于需要用户主动点击邮件中的按钮，且确认后会重定向到登录页，实际风险较低）
