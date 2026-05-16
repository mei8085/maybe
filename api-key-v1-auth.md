# API Key 与 V1 接口鉴权完整流程说明

## 目录

1. [API Key 数据结构](#1-api-key-数据结构)
2. [API Key 创建流程](#2-api-key-创建流程)
3. [设置页管理流程](#3-设置页管理流程)
4. [V1 接口鉴权流程](#4-v1-接口鉴权流程)
5. [家庭上下文进入请求](#5-家庭上下文进入请求)
6. [错误响应分类](#6-错误响应分类)

---

## 1. API Key 数据结构

### 数据库表结构

文件: `db/migrate/20250613002027_create_api_keys.rb`

```ruby
create_table :api_keys, id: :uuid do |t|
  t.string :key                    # 已废弃，使用 display_key 替代
  t.string :name                   # API Key 名称
  t.references :user, null: false  # 所属用户 (UUID)
  t.json :scopes                   # 权限范围 (数组)
  t.datetime :last_used_at         # 最后使用时间
  t.datetime :expires_at           # 过期时间
  t.datetime :revoked_at           # 撤销时间
  t.timestamps
end
```

### 模型定义

文件: `app/models/api_key.rb`

#### 核心属性

- **`display_key`**: 加密存储的 API Key（确定性加密，可查询）
- **`name`**: 用户自定义名称
- **`scopes`**: 权限范围数组 (支持 `read`, `read_write`)
- **`source`**: 来源 (`web` 或 `mobile`)
- **`revoked_at`**: 撤销时间
- **`expires_at`**: 过期时间

#### 核心方法

```ruby
# 查找并验证活跃的 API Key
def self.find_by_value(plain_key)
  find_by(display_key: plain_key)&.tap do |api_key|
    return api_key if api_key.active?
  end
end

# 生成安全随机 Key
def self.generate_secure_key
  SecureRandom.hex(32)
end

# 检查是否活跃（未撤销且未过期）
def active?
  !revoked? && !expired?
end

# 撤销 Key
def revoke!
  update!(revoked_at: Time.current)
end

# 更新最后使用时间
def update_last_used!
  update_column(:last_used_at, Time.current)
end
```

#### 验证规则

1. **权限验证**: `scopes` 必须包含至少一个权限，且只能有一个权限级别
2. **唯一性验证**: 每个用户每个 `source` 只能有一个活跃的 API Key
3. **格式验证**: 权限值必须是 `read` 或 `read_write`

---

### 🔴 Source 字段完整边界说明

#### 1. 默认值来源

**source 默认值定义在数据库迁移层面，而非模型层面**

文件: `db/migrate/20250618104425_add_source_to_api_keys.rb:3`
```ruby
add_column :api_keys, :source, :string, default: "web"
```

**关键行为**：
- `ApiKey.new.source` → `nil`（新建内存对象时无值）
- 只有调用 `save` 写入数据库时，才会由数据库填充 `"web"`
- 模型层没有 `before_validation` 回调设置默认值

---

#### 2. 模型层单活跃 Key 约束实现

**约束目标**：同一用户 + 同一 source，只能有一个活跃（未撤销、未过期）的 API Key

**三层防护实现**：

| 层级 | 实现位置 | 作用机制 |
|------|---------|---------|
| 数据库索引 | 迁移文件 | `add_index :api_keys, [:user_id, :source]` - 加速查询 |
| 模型验证 | `app/models/api_key.rb:16, 89-93` | 创建时检查是否已有同 source 活跃 Key |
| 业务逻辑 | 设置页控制器 | 创建前临时撤销所有现有 Key |

**模型验证源码**：
文件: `app/models/api_key.rb:16, 89-93`
```ruby
# 仅在创建时触发验证
validate :one_active_key_per_user_per_source, on: :create

def one_active_key_per_user_per_source
  if user&.api_keys&.active&.where(source: source)&.where&.not(id: id)&.exists?
    errors.add(:user, "can only have one active API key per source (#{source})")
  end
end
```

> 注意：`on: :create` 意味着更新操作（如撤销/过期）不会触发此验证。

---

#### 3. 设置页 Create 临时撤销所有 source Key 的原因

**设计决策背景**：

文件: `app/controllers/settings/api_keys_controller.rb:25-34`
```ruby
# 🔴 关键：不区分 source，全部临时撤销！
existing_keys = Current.user.api_keys.active
existing_keys.each { |key| key.update_column(:revoked_at, Time.current) }
```

**为什么要撤销所有 source，而不是只撤销 web source？**

| 原因 | 说明 |
|------|------|
| **设置页定位** | 面向普通用户，假定用户同一时间只需要一个 API Key，不区分端 |
| **简化心智模型** | 对普通用户隐藏 "source" 概念，前端表单没有 source 选择项 |
| **避免竞态条件** | 如果只撤销 web source，而用户恰好有 mobile key，模型验证仍然会失败（因为新 key 最终 source 是 web，但验证时还没到数据库默认值填充） |
| **模型验证时序** | source 默认值由数据库填充，save 前 validation 时 `source == nil`，无法正确过滤 |

> **时序问题详解**：创建时新 key 的 source 还是 nil，此时执行模型验证 `where(source: nil)` 找不到任何现有 key，理论上可以通过。但写入数据库后 source 变为 web，就可能出现重复。所以最稳妥的做法是**先全部撤销，再创建**。

---

#### 4. 创建成功与失败的结果保留

**状态机流程**：

```
开始创建
   │
   ▼
查询所有活跃 Key → existing_keys
   │
   ▼
全部临时撤销（update_column :revoked_at）
   │
   ▼
尝试 save 新 Key
   │
   ├─ ✅ 成功 → 直接跳转，旧 Key 保持撤销状态
   │    │
   │    └─ 最终结果：新 Key 生效，所有旧 Key 失效（跨 source）
   │
   └─ ❌ 失败 → 遍历 existing_keys 恢复 revoked_at = nil
        │
        └─ 最终结果：新 Key 未创建，所有旧 Key 恢复可用
```

**成功与失败对比表**：

| 场景 | 新 Key 状态 | 原有 web Key | 原有 mobile Key |
|------|------------|------------|----------------|
| 创建成功 | ✅ 生效（source = web） | ❌ 永久撤销 | ❌ 永久撤销 |
| 创建失败 | ❌ 回滚未创建 | ✅ 恢复可用 | ✅ 恢复可用 |

**代码证据**：
文件: `app/controllers/settings/api_keys_controller.rb:28-35`
```ruby
if @api_key.save
  # 成功后不做任何处理，旧 key 保持 revoked
  flash[:notice] = "Your API key has been created successfully"
  redirect_to settings_api_key_path
else
  # 失败则恢复所有旧 key（通过 ActiveRecord 脏对象追踪）
  existing_keys.each { |key| key.update_column(:revoked_at, nil) }
  render :new, status: :unprocessable_entity
end
```

> **边界风险**：并发创建请求时，第二个请求会撤销第一个请求刚创建的 Key，造成竞态条件丢失。

---

## 2. API Key 创建流程

### 步骤详解

#### 1. 密钥生成

文件: `app/controllers/settings/api_keys_controller.rb:20-21`

```ruby
# 生成 64 字符的随机十六进制字符串
@plain_key = ApiKey.generate_secure_key  # SecureRandom.hex(32)
@api_key = Current.user.api_keys.build(api_key_params)
@api_key.key = @plain_key  # 临时存储明文用于回调
```

#### 2. 临时撤销现有 Key

文件: `app/controllers/settings/api_keys_controller.rb:25-26`

```ruby
# 为了通过唯一性验证，先临时撤销现有 Key
existing_keys = Current.user.api_keys.active
existing_keys.each { |key| key.update_column(:revoked_at, Time.current) }
```

#### 3. 保存新 Key

文件: `app/controllers/settings/api_keys_controller.rb:28-35`

```ruby
if @api_key.save
  flash[:notice] = "Your API key has been created successfully"
  redirect_to settings_api_key_path
else
  # 保存失败，恢复原有 Key
  existing_keys.each { |key| key.update_column(:revoked_at, nil) }
  render :new, status: :unprocessable_entity
end
```

#### 4. 参数处理

文件: `app/controllers/settings/api_keys_controller.rb:53-60`

```ruby
def api_key_params
  permitted_params = params.require(:api_key).permit(:name, :scopes)
  # 将单个 scope 值转换为数组存储
  if permitted_params[:scopes].present?
    permitted_params[:scopes] = [ permitted_params[:scopes] ]
  end
  permitted_params
end
```

---

## 3. 设置页管理流程

### 路由配置

文件: `config/routes.rb:65`

```ruby
namespace :settings do
  resource :api_key, only: [ :show, :new, :create, :destroy ]
end
```

### 页面流程

#### 1. 显示页面 (show)

文件: `app/controllers/settings/api_keys_controller.rb:8-10`

```ruby
def show
  @current_api_key = @api_key  # @api_key = Current.user.api_keys.active.first
end
```

#### 2. 新建页面 (new)

文件: `app/controllers/settings/api_keys_controller.rb:12-17`

```ruby
def new
  # 如果已有活跃 Key 且不是显式要求重新生成，则重定向
  redirect_to settings_api_key_path if Current.user.api_keys.active.exists? && !params[:regenerate]
  @api_key = ApiKey.new
end
```

#### 3. 撤销 Key (destroy)

文件: `app/controllers/settings/api_keys_controller.rb:38-45`

```ruby
def destroy
  if @api_key&.revoke!
    flash[:notice] = "API key has been revoked successfully"
  else
    flash[:alert] = "Failed to revoke API key"
  end
  redirect_to settings_api_key_path
end
```

---

## 4. V1 接口鉴权流程

### 基础控制器结构

文件: `app/controllers/api/v1/base_controller.rb`

#### 前置过滤器执行顺序

```ruby
before_action :force_json_format              # 1. 强制 JSON 格式
before_action :authenticate_request!          # 2. 鉴权（核心）
before_action :check_api_key_rate_limit       # 3. 速率限制检查
before_action :log_api_access                 # 4. 日志记录
```

### 鉴权核心逻辑

#### authenticate_request! 方法

文件: `app/controllers/api/v1/base_controller.rb:42-46`

```ruby
def authenticate_request!
  return if authenticate_oauth      # 先尝试 OAuth 鉴权
  return if authenticate_api_key    # 再尝试 API Key 鉴权
  render_unauthorized unless performed?  # 都失败则返回 401
end
```

#### OAuth 鉴权流程

文件: `app/controllers/api/v1/base_controller.rb:49-88`

```ruby
def authenticate_oauth
  return false unless request.headers["Authorization"].present?

  # 1. 提取 Token
  token_string = request.authorization&.split(" ")&.last
  access_token = Doorkeeper::AccessToken.by_token(token_string)

  # 2. 验证 Token 有效性和权限
  has_sufficient_scope = access_token&.scopes&.include?("read") || 
                         access_token&.scopes&.include?("read_write")

  unless access_token && !access_token.expired? && has_sufficient_scope
    render_json({ error: "unauthorized", message: "..." }, status: :unauthorized)
    return false
  end

  # 3. 设置当前用户
  @_doorkeeper_token = access_token
  @current_user = User.find_by(id: doorkeeper_token.resource_owner_id)
  
  @authentication_method = :oauth
  setup_current_context_for_api  # 设置上下文
  true
end
```

#### API Key 鉴权流程

文件: `app/controllers/api/v1/base_controller.rb:91-104`

```ruby
def authenticate_api_key
  api_key_value = request.headers["X-Api-Key"]
  return false unless api_key_value

  # 1. 查找并验证 API Key
  @api_key = ApiKey.find_by_value(api_key_value)
  return false unless @api_key && @api_key.active?

  # 2. 设置当前用户
  @current_user = @api_key.user
  @api_key.update_last_used!  # 更新使用时间
  
  @authentication_method = :api_key
  @rate_limiter = ApiRateLimiter.limit(@api_key)  # 初始化速率限制器
  
  setup_current_context_for_api  # 设置上下文
  true
end
```

#### 边界：双 Header 场景分支逻辑

**真实执行路径流程图**：

```
请求到达
   │
   ▼
authenticate_request!
   │
   ├─→ authenticate_oauth
   │    │
   │    ├─ Authorization 不存在？ → return false ←┐
   │    │                                          │
   │    └─ Authorization 存在？
   │         │
   │         ├─ 验证通过 → return true ←───────┐  │
   │         │                                  │  │
   │         └─ 验证失败                        │  │
   │              │                             │  │
   │              ├─ render_json(401) ←─────── 已发送响应！
   │              │                             │  │
   │              └─ return false ←─────────────┘  │
   │                                                │
   └─→ authenticate_api_key  ◄────────── 这里还会执行吗？
        │
        ├─ X-Api-Key 不存在？ → return false
        │
        └─ X-Api-Key 存在且有效？
             ├─ 是 → 设置 @current_user, return true
             └─ 否 → return false
```

**源码证据与真实分支结果**：

文件: `app/controllers/api/v1/base_controller.rb:42-46, 59-62`
```ruby
def authenticate_request!
  return if authenticate_oauth      # 步骤1: 返回 false，不 return
  return if authenticate_api_key    # 步骤2: 还会继续执行！
  render_unauthorized unless performed?
end

# 在 authenticate_oauth 验证失败时：
unless access_token && !access_token.expired? && has_sufficient_scope
  render_json({ error: "unauthorized", message: "..." }, status: :unauthorized)
  return false  # ← 先 render，再 return false
end
```

**关键发现**：当 `Authorization` 存在但验证失败时，`authenticate_oauth` 先调用 `render_json` 再返回 `false`，此时：
1. ✅ `performed?` 已变为 true（响应已写入）
2. ❌ 方法仍然返回 `false`，所以 **会继续执行 `authenticate_api_key`**
3. 但即使 API Key 验证成功，也无法改变已经发送的 401 响应
4. Rails 不会报 DoubleRenderError，因为第二次 render 被跳过

**最终结果矩阵**：

| 场景 | Authorization | X-Api-Key | 真实执行路径 | 最终结果 |
|------|--------------|-----------|------------|---------|
| 1 | ✅ 有效 | 任意 | OAuth 成功，return true → 终止 | 200 成功 |
| 2 | ❌ 无效（Token 存在但有问题） | ✅ 有效 | OAuth render 401 → return false → 执行 API Key 成功但无法改变响应 | **401 Unauthorized** |
| 3 | ❌ 不存在 | ✅ 有效 | OAuth return false → API Key 成功 | 200 成功 |
| 4 | ❌ 不存在 | ❌ 无效 | 两者都失败 → render_unauthorized | 401 Unauthorized |

> **设计意图**：Authorization header 的存在表示客户端**明确意图**使用 OAuth 鉴权，即使失败也不应该静默降级到 API Key。这种设计避免了鉴权方式的隐式切换带来的安全隐患。

### 权限范围检查

#### authorize_scope! 方法

文件: `app/controllers/api/v1/base_controller.rb:174-195`

```ruby
def authorize_scope!(required_scope)
  scopes = current_scopes  # 从 OAuth 或 API Key 获取当前权限

  case required_scope.to_s
  when "read"
    # read 权限允许 read 或 read_write
    has_access = scopes.include?("read") || scopes.include?("read_write")
  when "write"
    # write 权限只允许 read_write
    has_access = scopes.include?("read_write")
  else
    # 其他权限精确匹配
    has_access = scopes.include?(required_scope.to_s)
  end

  unless has_access
    render_json({ error: "insufficient_scope", message: "..." }, status: :forbidden)
    return false
  end
  true
end
```

#### 在子控制器中的使用

文件: `app/controllers/api/v1/transactions_controller.rb:7-8`

```ruby
before_action :ensure_read_scope, only: [ :index, :show ]
before_action :ensure_write_scope, only: [ :create, :update, :destroy ]

def ensure_read_scope
  authorize_scope!(:read)
end

def ensure_write_scope
  authorize_scope!(:write)
end
```

### 速率限制

文件: `app/services/api_rate_limiter.rb`

#### 速率限制级别

```ruby
RATE_LIMITS = {
  standard: 100,      # 标准级：每小时 100 次请求
  premium: 1000,      # 高级：每小时 1000 次请求
  enterprise: 10000   # 企业级：每小时 10000 次请求
}.freeze
```

#### 自托管模式例外

```ruby
def self.limit(api_key)
  if Rails.application.config.app_mode.self_hosted?
    NoopApiRateLimiter.new(api_key)  # 自托管模式无限速
  else
    new(api_key)
  end
end
```

#### 响应头

```
X-RateLimit-Limit: 100          # 总限制
X-RateLimit-Remaining: 99       # 剩余请求数
X-RateLimit-Reset: 3599         # 重置时间（秒）
Retry-After: 3599               # 重试间隔（秒）
```

---

## 5. 家庭上下文进入请求

### Current 上下文模型

文件: `app/models/current.rb`

```ruby
class Current < ActiveSupport::CurrentAttributes
  attribute :user_agent, :ip_address
  attribute :session

  delegate :family, to: :user, allow_nil: true

  def user
    impersonated_user || session&.user
  end
end
```

### API 请求上下文设置

文件: `app/controllers/api/v1/base_controller.rb:253-271`

```ruby
def setup_current_context_for_api
  if @current_user
    # 尝试查找现有 Session，或创建临时 Session
    session = @current_user.sessions.first
    if session
      Current.session = session
    else
      # 创建临时 Session（不持久化）
      session = @current_user.sessions.build(
        user_agent: request.user_agent,
        ip_address: request.ip
      )
      Current.session = session
    end
  end
end
```

### 在业务逻辑中使用家庭上下文

#### 边界：Current.user 与 current_resource_owner 的取值链路差异

**1. 完整时序对比与可获得性窗口**

```
API 请求生命周期时序：

  before_action 1: authenticate_request!
      │
      ├─ authenticate_oauth / authenticate_api_key 成功
      │    └─ @current_user = user  ← @current_user 在此处可用
      │         └─ current_resource_owner 立即可用 ✅
      │
      └─ setup_current_context_for_api 执行
           ├─ 查找现有 Session 或 build 临时 Session
           └─ Current.session = session  ← Current 上下文在此设置
                └─ 此时 Current.user 才可用 ✅

  before_action 2: check_api_key_rate_limit
  before_action 3: log_api_access
  └─ 两个用户对象都已可用

  控制器动作执行
```

**关键差异点**：
| 时间点 | `current_resource_owner` | `Current.user` |
|--------|-------------------------|----------------|
| 鉴权成功后立即 | ✅ 可用 | ❌ 不可用（需等 setup_current_context_for_api） |
| setup 完成后 | ✅ 可用 | ✅ 可用 |

---

**2. 属性对比总表**

| 维度 | `current_resource_owner` | `Current.user` |
|------|-------------------------|----------------|
| 定义位置 | API Base Controller 实例方法 | 全局 CurrentAttributes 单例 |
| 文件位置 | `app/controllers/api/v1/base_controller.rb:156-158` | `app/models/current.rb` |
| 核心代码 | 直接返回 `@current_user` 实例变量 | `impersonated_user || session&.user` |
| 设置时机 | 鉴权成功时直接赋值 | `setup_current_context_for_api` 回调 |
| 生效延迟 | 立即可用 | 需等待 Session 查找/构建完成 |
| 线程安全 | 控制器实例级别，请求隔离 | 全局单例，但基于 RequestStore |

---

**3. 源码实现证据**

文件: `app/controllers/api/v1/base_controller.rb:156-158`
```ruby
# 返回直接持有，无额外间接调用
def current_resource_owner
  @current_user
end
```

文件: `app/models/current.rb:8-14`
```ruby
def user
  # 优先返回被模仿的用户（支持管理员模拟用户场景）
  impersonated_user || session&.user
end

def impersonated_user
  session&.active_impersonator_session&.impersonated
end
```

---

**4. 推荐使用场景矩阵**

| 场景 | 推荐使用 | 原因 |
|------|---------|------|
| API 控制器内获取当前用户 | `current_resource_owner` ✅ | 立即可用，直接可靠 |
| API 控制器内获取家庭 | `current_resource_owner.family` ✅ | 不需要 Current 上下文 |
| Model 层回调/关注点 | `Current.user` ✅ | 全局可访问，脱离控制器 |
| Job/异步任务 | `Current.user` ✅ | 需要通过 Current 传递上下文 |
| Web 控制器（非 API） | `Current.user` ✅ | 由 Session 中间件设置 |
| 跨线程/异步调用 | 需要手动传递 ⚠️ | CurrentAttributes 不跨线程 |

> **API 控制器铁律**：在 API 控制器及其子类中，**始终使用 `current_resource_owner`**。
> - 即使两者指向同一个 User 记录，直接使用 `@current_user` 避免了对 Session 查找的依赖
> - 避免了在 before_action 执行顺序问题导致的 nil 异常
> - 保持 API 鉴权层的独立性，不依赖全局状态

#### 示例 1: 账户查询

文件: `app/controllers/api/v1/accounts_controller.rb:11-12`

```ruby
def index
  family = current_resource_owner.family  # 通过鉴权用户获取家庭
  accounts_query = family.accounts.visible.alphabetically
  # ...
end
```

#### 示例 2: 交易查询

文件: `app/controllers/api/v1/transactions_controller.rb:12, 154-155`

```ruby
def index
  family = current_resource_owner.family
  transactions_query = family.transactions.visible
  # ...
end

def set_transaction
  family = current_resource_owner.family
  @transaction = family.transactions.find(params[:id])  # 限定在家庭范围内查询
  # ...
end
```

#### 跨家庭访问防护

文件: `app/controllers/api/v1/base_controller.rb:235-245`

```ruby
def ensure_current_family_access(resource)
  return unless resource.respond_to?(:family_id)

  unless resource.family_id == current_resource_owner.family_id
    render_json({ error: "forbidden", message: "Access denied to this resource" }, 
                status: :forbidden)
    return false
  end
  true
end
```

---

## 6. 错误响应分类

### 6.1 无效 Key 响应 (401 Unauthorized)

#### 触发场景

1. **缺少认证信息**: 未提供 `Authorization` 或 `X-Api-Key` Header
2. **API Key 无效**: `X-Api-Key` 值不存在
3. **API Key 已撤销**: `revoked_at` 不为空
4. **API Key 已过期**: `expires_at` < 当前时间
5. **OAuth Token 无效**: Token 不存在或已过期
6. **OAuth Token 用户不存在**: `resource_owner_id` 对应的用户已删除

#### 响应格式

```json
{
  "error": "unauthorized",
  "message": "Access token or API key is invalid, expired, or missing"
}
```

#### HTTP 状态码: `401 Unauthorized`

---

### 6.2 权限不足响应 (403 Forbidden)

#### 触发场景

1. **Scope 权限不足**: 操作需要 `write` 权限，但只有 `read` 权限
2. **跨家庭访问**: 尝试访问不属于当前用户家庭的资源
3. **功能未启用**: 例如 AI 功能未启用

#### 响应格式 1: Scope 权限不足

```json
{
  "error": "insufficient_scope",
  "message": "This action requires the 'write' scope"
}
```

#### 响应格式 2: 跨家庭访问

```json
{
  "error": "forbidden",
  "message": "Access denied to this resource"
}
```

#### 响应格式 3: 功能未启用

```json
{
  "error": "feature_disabled",
  "message": "AI features are not enabled for this user"
}
```

#### HTTP 状态码: `403 Forbidden`

---

### 6.3 其他常见错误响应

#### 404 Not Found - 资源不存在

```json
{
  "error": "record_not_found",
  "message": "The requested resource was not found"
}
```

#### 429 Too Many Requests - 速率限制

```json
{
  "error": "rate_limit_exceeded",
  "message": "Rate limit exceeded. Try again in 3600 seconds.",
  "details": {
    "limit": 100,
    "current": 100,
    "reset_in_seconds": 3600
  }
}
```

#### 422 Unprocessable Entity - 验证失败

```json
{
  "error": "validation_failed",
  "message": "Transaction could not be created",
  "errors": ["Account ID is required"]
}
```

#### 400 Bad Request - 参数缺失

```json
{
  "error": "bad_request",
  "message": "Required parameters are missing or invalid"
}
```

---

## 附录: 完整流程时序图

```
用户请求 → [X-Api-Key Header]
    ↓
BaseController#authenticate_request!
    ├─→ authenticate_oauth (优先)
    │    ├─ 检查 Authorization Header
    │    ├─ 查找 Doorkeeper::AccessToken
    │    ├─ 验证 Token 有效性和 Scope
    │    ├─ 设置 @current_user
    │    └─ setup_current_context_for_api
    └─→ authenticate_api_key (备选)
         ├─ 检查 X-Api-Key Header
         ├─ ApiKey.find_by_value(key)
         ├─ 验证 active? (未撤销未过期)
         ├─ 设置 @current_user = @api_key.user
         ├─ @api_key.update_last_used!
         ├─ 初始化 ApiRateLimiter
         └─ setup_current_context_for_api
    ↓
check_api_key_rate_limit
    ├─ 检查是否超过限制
    ├─ 增加计数
    └─ 添加响应头 X-RateLimit-*
    ↓
log_api_access
    ↓
子控制器 before_action (ensure_read_scope / ensure_write_scope)
    └─ authorize_scope!(:read/:write)
         ├─ 获取 current_scopes
         └─ 验证权限层级关系
              read     ← read_write
              write    ← read_write
    ↓
业务逻辑执行
    └─ current_resource_owner.family.xxx
         (所有查询限定在当前家庭范围内)
```
