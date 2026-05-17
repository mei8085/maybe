# 家庭邀请注册链路 - 订阅衔接逻辑深度分析 (Round 2)

## 核心结论摘要

**注册成功后不会自动开始试用**。订阅激活是一个独立的阶段，发生在onboarding流程的最后一步，由用户主动点击触发。邀请成员加入现有家庭时，订阅状态通过数据库关联关系隐式继承，无需任何额外处理。

---

## 一、三阶段职责边界

| 阶段 | 触发时机 | 主要职责 | 订阅相关操作 |
|------|----------|----------|--------------|
| **注册阶段** | 用户提交注册表单 | 创建User/Family、建立会话 | ❌ 无任何订阅操作 |
| **Onboarding阶段** | 首次登录后自动跳转 | 收集用户偏好、家庭设置 | ⚠️ 仅判断是否需要订阅，不创建 |
| **订阅开通阶段** | 用户点击"开始试用"按钮 | 创建试用或付费订阅 | ✅ 正式创建Subscription记录 |

---

## 二、注册阶段：纯净的账户创建

### 2.1 代码证据

`app/controllers/registrations_controller.rb:15-33`

```ruby
def create
  if @invitation
    @user.family = @invitation.family
    @user.role = @invitation.role
    @user.email = @invitation.email
  else
    family = Family.new
    @user.family = family
    @user.role = :admin
  end

  if @user.save
    @invitation&.update!(accepted_at: Time.current)
    @session = create_session_for(@user)
    redirect_to root_path, notice: t(".success")
  else
    render :new, status: :unprocessable_entity, alert: t(".failure")
  end
end
```

**关键发现：**
1. 仅创建 `User` 和 `Family`（非邀请场景）
2. 标记邀请为已接受（`accepted_at`）
3. 创建会话 `create_session_for(@user)`
4. **没有任何 `Subscription` 创建逻辑**
5. **没有设置 `onboarded_at`** - 用户处于未完成onboarding状态

### 2.2 注册后重定向

注册成功后 `redirect_to root_path`，但会被 `Onboardable` concern 拦截。

---

## 三、Onboarding阶段：流程控制与状态收集

### 3.1 全局重定向拦截器

`app/controllers/concerns/onboardable.rb:9-21`

```ruby
def require_onboarding_and_upgrade
  return unless Current.user
  return unless redirectable_path?(request.path)

  if Current.user.needs_onboarding?
    redirect_to onboarding_path
  elsif Current.family.needs_subscription?
    redirect_to trial_onboarding_path
  elsif Current.family.upgrade_required?
    redirect_to upgrade_subscription_path
  end
end
```

**判断条件：**
- `needs_onboarding?` → `onboarded_at.blank?`（`app/models/user.rb:162-164`）
- `needs_subscription?` → `subscription.nil? && !self_hoster?`（`app/models/family/subscribeable.rb:44-46`）
- `upgrade_required?` → `!(subscription&.active? || subscription&.trialing?)`（`app/models/family/subscribeable.rb:18-23`）

### 3.2 Onboarding完整流程（新家庭创建者）

```
注册成功 → 访问首页 → Onboardable拦截 → 跳转/onboarding → 完成四步流程
                                                          ↓
                                          ┌───────────────────────────┐
                                          │ 1. Setup (个人资料)       │
                                          │   - first_name, last_name │
                                          │   - avatar                │
                                          │   - family name/country   │
                                          │   → redirect: preferences │
                                          └─────────────┬─────────────┘
                                                        ↓
                                          ┌───────────────────────────┐
                                          │ 2. Preferences (偏好设置) │
                                          │   - theme, locale         │
                                          │   - currency, date_format │
                                          │   → redirect: goals       │
                                          └─────────────┬─────────────┘
                                                        ↓
                                          ┌───────────────────────────┐
                                          │ 3. Goals (目标选择)       │
                                          │   - 选择财务目标          │
                                          │   - 设置 onboarded_at     │
                                          │   → redirect: trial       │
                                          └─────────────┬─────────────┘
                                                        ↓
                                          ┌───────────────────────────┐
                                          │ 4. Trial (试用激活)       │
                                          │   - 用户点击按钮          │
                                          │   - POST /subscriptions   │
                                          │   → 创建订阅记录          │
                                          └───────────────────────────┘
```

**代码证据 - 步骤1 Setup：**
`app/views/onboardings/show.html.erb:21`
```erb
<%= form.hidden_field :redirect_to, value: @invitation ? "home" : "onboarding_preferences" %>
<%= form.hidden_field :onboarded_at, value: Time.current if @invitation %>
```
- 非邀请用户：`redirect_to = "onboarding_preferences"`，继续下一步
- 邀请用户：`redirect_to = "home"`，**直接跳过后续步骤**

**代码证据 - 步骤3 Goals：**
`app/views/onboardings/goals.html.erb:23-25`
```erb
<%= form.hidden_field :redirect_to, value: self_hosted? ? "home" : "trial" %>
<%= form.hidden_field :set_onboarding_goals_at, value: Time.current %>
<%= form.hidden_field :onboarded_at, value: Time.current %>
```
- 自托管用户：跳过试用，直接到首页
- 非自托管用户：进入试用激活页面
- **这里才设置 `onboarded_at`**

### 3.3 邀请用户的Onboarding简化流程

```
注册成功 → 访问首页 → Onboardable拦截 → 跳转/onboarding
                                                          ↓
                                          ┌───────────────────────────┐
                                          │ 1. Setup (仅个人资料)     │
                                          │   - first_name, last_name │
                                          │   - avatar                │
                                          │   - 设置 onboarded_at     │
                                          │   → redirect: home        │
                                          └───────────────────────────┘
                                                          ↓
                                                      完成
```

**关键差异代码：**
`app/views/onboardings/show.html.erb:33-44`
```erb
<% unless @invitation %>
  <div class="space-y-4 mb-4">
    <%= form.fields_for :family do |family_form| %>
      <%= family_form.text_field :name, placeholder: "Household name", label: "Household name" %>
      <%= family_form.select :country, country_options, { label: "Country" }, required: true %>
    <% end %>
  </div>
<% end %>
```
- 邀请用户：隐藏家庭名称和国家选择（已有家庭）
- 邀请用户：`onboarded_at` 在Setup步骤就设置，直接完成onboarding

---

## 四、订阅开通阶段：主动触发的试用激活

### 4.1 试用激活流程

**页面展示：** `app/views/onboardings/trial.html.erb:31-51`
```erb
<% if Current.family.can_start_trial? %>
  <%= render DS::Button.new(
    text: "Try Maybe for 14 days",
    href: subscription_path,
    full_width: true,
    data: { turbo: false }
  ) %>
<% elsif Current.family.trialing? %>
  <%= render DS::Link.new(text: "Continue trial", href: root_path) %>
<% else %>
  <%= render DS::Link.new(text: "Upgrade", href: upgrade_subscription_path) %>
<% end %>
```

**订阅创建：** `app/controllers/subscriptions_controller.rb:30-37`
```ruby
def create
  if Current.family.can_start_trial?
    Current.family.start_trial_subscription!
    redirect_to root_path, notice: "Welcome to Maybe!"
  else
    redirect_to root_path, alert: "You have already started or completed a trial."
  end
end
```

**试用创建实现：** `app/models/family/subscribeable.rb:29-34`
```ruby
def start_trial_subscription!
  create_subscription!(
    status: "trialing",
    trial_ends_at: Subscription.new_trial_ends_at  # 14.days.from_now
  )
end
```

**判断能否试用：** `app/models/family/subscribeable.rb:25-27`
```ruby
def can_start_trial?
  subscription&.trial_ends_at.blank?
end
```
→ **只要曾经创建过订阅（无论状态），就不能再开始新试用**

### 4.2 付费订阅流程

```
用户选择计划 → /subscriptions/new → 创建Stripe checkout session
                                                        ↓
                                          跳转Stripe支付页面
                                                        ↓
                                          支付成功 → /subscriptions/success
                                                        ↓
                                          更新subscription.status = "active"
```

**代码证据：** `app/controllers/subscriptions_controller.rb:49-58`
```ruby
def success
  checkout_result = stripe.get_checkout_result(params[:session_id])
  if checkout_result.success?
    Current.family.start_subscription!(checkout_result.subscription_id)
    redirect_to root_path, notice: "Welcome to Maybe!  Your subscription has been created."
  end
end
```

---

## 五、邀请成员订阅状态继承机制

### 5.1 核心原理：通过关联隐式继承

订阅状态**不需要任何显式的继承代码**，完全通过ActiveRecord关联关系自动实现：

```
User (成员)
  ↓ belongs_to
Family (家庭)
  ↓ has_one
Subscription (订阅)
```

**代码证据 - 模型关联：**
- `app/models/user.rb:4` → `belongs_to :family`
- `app/models/family.rb:2` → `include Family::Subscribeable`
- `app/models/family/subscribeable.rb:5` → `has_one :subscription, dependent: :destroy`

### 5.2 订阅状态查询链路

当任何家庭成员访问需要订阅权限的功能时：

```ruby
# 任何控制器或视图中
Current.family.has_active_subscription?  # 实际调用链
  ↓
family.subscription&.active?
  ↓
# 返回true/false，对所有成员一致
```

**所有成员共享完全相同的订阅状态**，因为：
1. 所有成员的 `family_id` 相同
2. 每个家庭只有一条 `subscription` 记录（`family_id` 唯一索引）
3. 状态查询都通过同一条 `subscription` 记录

### 5.3 唯一性保证

`app/models/subscription.rb:20`
```ruby
validates :family_id, uniqueness: true
```

`db/schema.rb` 中 subscriptions 表的唯一索引确保每个家庭只能有一条订阅记录。

### 5.4 计费主体

`app/models/family/subscribeable.rb:8-16`
```ruby
def billing_email
  primary_admin = users.admin.order(:created_at).first || users.super_admin.order(:created_at).first
  primary_admin.email
end
```
→ **计费始终以家庭最早创建的管理员为准**，与被邀请成员无关

---

## 六、边界场景的订阅状态

### 6.1 邀请流程中的订阅状态

| 场景 | 订阅状态 | 被邀请用户体验 |
|------|----------|----------------|
| 家庭已有活跃订阅 | `active` | 注册→Setup→直接使用，无试用提示 |
| 家庭正在试用中 | `trialing` | 注册→Setup→直接使用，共享试用剩余天数 |
| 家庭订阅已过期 | `paused/canceled` | 注册→Setup→使用时被重定向到升级页面 |
| 家庭无订阅 | `nil` | 注册→Setup→直接使用（非自托管会被拦截升级） |

### 6.2 Onboardable对邀请用户的行为

`app/controllers/concerns/onboardable.rb:14-20`

```ruby
if Current.user.needs_onboarding?
  redirect_to onboarding_path          # 邀请用户需要走Setup
elsif Current.family.needs_subscription?
  redirect_to trial_onboarding_path    # 家庭无订阅才会到这里
elsif Current.family.upgrade_required?
  redirect_to upgrade_subscription_path # 订阅过期才会到这里
end
```

**关键：** 邀请用户完成Setup后 `onboarded_at` 已设置，后续订阅判断完全取决于家庭订阅状态，与邀请身份无关。

---

## 七、关键代码位置汇总

| 功能 | 文件位置 | 行号 |
|------|----------|------|
| 注册创建（无订阅） | `app/controllers/registrations_controller.rb` | 15-33 |
| Onboarding重定向拦截 | `app/controllers/concerns/onboardable.rb` | 9-21 |
| Onboarding Setup页面（邀请分支） | `app/views/onboardings/show.html.erb` | 21, 33-44 |
| Onboarding Goals（设置onboarded_at） | `app/views/onboardings/goals.html.erb` | 23-25 |
| 试用激活页面 | `app/views/onboardings/trial.html.erb` | 31-51 |
| 创建试用订阅 | `app/controllers/subscriptions_controller.rb` | 30-37 |
| 开始试用方法 | `app/models/family/subscribeable.rb` | 29-34 |
| 能否试用判断 | `app/models/family/subscribeable.rb` | 25-27 |
| 订阅状态判断 | `app/models/family/subscribeable.rb` | 18-23, 40-46 |
| 计费邮箱（主管理员） | `app/models/family/subscribeable.rb` | 8-16 |
| User-Family关联 | `app/models/user.rb` | 4 |
| Family-Subscription关联 | `app/models/family/subscribeable.rb` | 5 |
| Subscription family_id唯一性 | `app/models/subscription.rb` | 20 |

---

## 八、修正的流程图（对比Round 1）

### Round 1 错误认知：
```
注册 → 自动创建家庭 → 自动开始试用
```

### Round 2 正确流程：
```
注册 → 创建User/Family → 跳转Onboarding Setup
                              ↓
                    新用户：Setup → Preferences → Goals → Trial(点击激活) → 首页
                              ↓
                    邀请用户：Setup(设置onboarded_at) → 首页
                              ↓
                    订阅状态完全取决于加入的家庭已有状态
```
