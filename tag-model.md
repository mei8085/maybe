# 标签系统完整脉络

## 一、数据模型层

### 1.1 核心模型

**`Tag` 模型** (`app/models/tag.rb:1`)

```ruby
class Tag < ApplicationRecord
  belongs_to :family
  has_many :taggings, dependent: :destroy
  has_many :transactions, through: :taggings, source: :taggable, source_type: "Transaction"
  has_many :import_mappings, as: :mappable, dependent: :destroy, class_name: "Import::Mapping"

  validates :name, presence: true, uniqueness: { scope: :family }

  COLORS = %w[#e99537 #4da568 #6471eb #db5a54 #df4e92 #c44fe9 #eb5429 #61c9ea #805dee #6ad28a]
  UNCATEGORIZED_COLOR = "#737373"
end
```

**`Tagging` 关联模型** (`app/models/tagging.rb:1`)

```ruby
class Tagging < ApplicationRecord
  belongs_to :tag
  belongs_to :taggable, polymorphic: true
end
```

### 1.2 数据库表结构

**`tags` 表** (`db/schema.rb:719`)

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | uuid | 主键 |
| `name` | string | 标签名称，家庭范围内唯一 |
| `color` | string | 标签颜色，默认 `#e99537` |
| `family_id` | uuid | 外键，关联家庭 |
| `created_at` / `updated_at` | datetime | 时间戳 |

**`taggings` 表** (`db/schema.rb:709`)

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | uuid | 主键 |
| `tag_id` | uuid | 外键，关联标签 |
| `taggable_type` | string | 多态类型，如 "Transaction" |
| `taggable_id` | uuid | 多态 ID |
| `created_at` / `updated_at` | datetime | 时间戳 |

---

## 二、归属关系：家庭级多归属

### 2.1 归属模型

标签采用**家庭级归属**设计，而非用户级：

```ruby
# app/models/family.rb:29
class Family < ApplicationRecord
  has_many :tags, dependent: :destroy
end

# app/models/tag.rb:2
class Tag < ApplicationRecord
  belongs_to :family
end
```

**设计意图**：
- 标签是家庭共享资源，所有家庭成员使用同一套标签体系
- 通过 `uniqueness: { scope: :family }` 确保同一家庭内标签名称唯一
- 家庭删除时级联删除所有标签

### 2.2 与用户的关系

标签没有直接的 `user_id` 外键，用户通过家庭间接关联标签：

```
User → belongs_to :family → has_many :tags
```

控制器中通过 `Current.family` 获取当前用户的家庭上下文：

```ruby
# app/controllers/tags_controller.rb:5
def index
  @tags = Current.family.tags.alphabetically
end
```

---

## 三、与交易的关联：多态标签机制

### 3.1 关联方式

采用**多态关联**设计，通过中间表 `taggings` 实现标签与交易的多对多关系：

```ruby
# app/models/transaction.rb:7-10
class Transaction < ApplicationRecord
  has_many :taggings, as: :taggable, dependent: :destroy
  has_many :tags, through: :taggings
  accepts_nested_attributes_for :taggings, allow_destroy: true
end
```

**关联链路**：
```
Transaction ←→ Tagging (polymorphic) ←→ Tag
```

### 3.2 操作方式

**赋值标签**：通过 `tag_ids` 数组直接操作：

```ruby
# app/controllers/transactions_controller.rb:131
entryable_attributes: [ :id, :category_id, :merchant_id, :kind, { tag_ids: [] } ]
```

**查询筛选**：通过 `joins(:tags)` 进行筛选：

```ruby
# app/models/transaction/search.rb:139-142
def apply_tag_filter(query, tags)
  return query unless tags.present?
  query.joins(:tags).where(tags: { name: tags })
end
```

**API 输出**：序列化为嵌套数组：

```ruby
# app/views/api/v1/transactions/_transaction.json.jbuilder:42-46
json.tags transaction.tags do |tag|
  json.id tag.id
  json.name tag.name
  json.color tag.color
end
```

### 3.3 特殊机制：标签锁定

当交易被同步或锁定时，标签也会被锁定防止修改：

```ruby
# app/models/entry.rb:92
entry.entryable.lock_attr!(:tag_ids) if entry.transaction? && entry.transaction.tags.any?
```

---

## 四、颜色等元信息的承载方式

### 4.1 颜色系统

**颜色定义** (`app/models/tag.rb:11-13`)：
```ruby
COLORS = %w[#e99537 #4da568 #6471eb #db5a54 #df4e92 #c44fe9 #eb5429 #61c9ea #805dee #6ad28a]
UNCATEGORIZED_COLOR = "#737373"
```

**颜色承载**：直接存储在 `tags.color` 字段中，创建时随机分配：

```ruby
# app/controllers/tags_controller.rb:11
def new
  @tag = Current.family.tags.new color: Tag::COLORS.sample
end
```

### 4.2 颜色在视图中的渲染

#### 4.2.1 颜色头像 Partial

**位置**：`app/views/shared/_color_avatar.html.erb`

```erb
<%# locals: (name: nil, color: "#000") %>
<% letter = name&.first || "?" %>
<% background_color = "color-mix(in srgb, #{color} 5%, white)" %>
<% border_color = "color-mix(in srgb, #{color} 10%, white)" %>
<span data-color-avatar-target="avatar"
      class="w-8 h-8 flex items-center justify-center rounded-full"
      style="background-color: <%= background_color %>; border-color: <%= border_color %>; color: <%= color %>">
  <%= letter.upcase %>
</span>
```

**使用场景**：
1. **标签列表项** (`app/views/tags/_tag.html.erb:5`) - 在标签管理页面展示每个标签的颜色头像
2. **标签编辑表单** (`app/views/tags/_form.html.erb:5`) - 新建/编辑标签时的颜色预览

**技术要点**：
- 展示标签名称首字母的圆形头像
- 使用 CSS `color-mix()` 函数实现颜色透明化
- 背景色：颜色 + 5% 透明度混合白色
- 边框色：颜色 + 10% 透明度混合白色
- 文字色：直接使用标签颜色
- 作为 Stimulus `color-avatar` 控制器的目标元素

#### 4.2.2 DS::FilledIcon 组件

**位置**：`app/components/DS/filled_icon.rb`

这是一个通用的设计系统组件，用于在标签筛选器等场景展示带颜色的图标/文字：

```ruby
def container_styles
  <<~STYLE.strip
    background-color: #{transparent_bg_color};
    border-color: #{transparent_border_color};
    color: #{custom_fg_color};
  STYLE
end

def transparent_bg_color
  "color-mix(in oklab, #{custom_fg_color} 10%, transparent)"
end

def transparent_border_color
  "color-mix(in oklab, #{custom_fg_color} 10%, transparent)"
end
```

**使用场景**：标签筛选器 (`app/views/transactions/searches/filters/_tag_filter.html.erb:19-25`)

```erb
<%= render DS::FilledIcon.new(
  variant: :text,
  hex_color: tag.color || Tag::UNCATEGORIZED_COLOR,
  text: tag.name,
  size: "sm",
  rounded: true
) %>
```

**技术要点**：
- 使用 `oklab` 色彩空间进行颜色混合（而非 `srgb`）
- 背景透明度为 10%
- 支持多种尺寸变体（sm/md/lg）

#### 4.2.3 标签徽章 Partial

**位置**：`app/views/tags/_badge.html.erb`

```erb
<%# locals: (tag:) %>
<span class="border text-sm font-medium px-2.5 py-1 rounded-full content-center"
      style="
        background-color: color-mix(in srgb, <%= tag.color %> 5%, white);
        border-color: color-mix(in srgb, <%= tag.color %> 10%, white);
        color: <%= tag.color %>;">
  <%= tag.name %>
</span>
```

---

## 五、UI 选择器实现

### 5.1 表单多选器

**交易表单中的标签选择** (`app/views/transactions/_form.html.erb:34-41`)：
```erb
<%= ef.select :tag_ids,
              Current.family.tags.alphabetically.pluck(:name, :id),
              {
                include_blank: t(".none"),
                multiple: true,
                label:    t(".tags_label"),
                container_class: "h-40"
              } %>
```

**自动提交**：在交易详情抽屉中配合 Stimulus 自动提交：
```erb
# app/views/transactions/show.html.erb:79-87
<%= ef.select :tag_ids,
      Current.family.tags.alphabetically.pluck(:name, :id),
      {
        include_blank: t(".none"),
        multiple: true,
        label:    t(".tags_label"),
        container_class: "h-40"
      },
      { "data-auto-submit-form-target": "auto" } %>
```

### 5.2 颜色选择器

**标签编辑表单** (`app/views/tags/_form.html.erb:1`)：
```erb
<div data-controller="color-avatar">
  <%= styled_form_with model: tag, class: "space-y-4", data: { turbo_frame: :_top } do |f| %>
    <div class="w-fit m-auto">
      <%= render partial: "shared/color_avatar", locals: { name: tag.name, color: tag.color } %>
    </div>
    <div class="flex gap-2 items-center justify-center">
      <% Tag::COLORS.each do |color| %>
        <label class="relative">
          <%= f.radio_button :color, color, class: "sr-only peer", data: { action: "change->color-avatar#handleColorChange" } %>
          <div class="w-6 h-6 rounded-full cursor-pointer peer-checked:ring-2 peer-checked:ring-offset-2 peer-checked:ring-blue-500" style="background-color: <%= color %>"></div>
        </label>
      <% end %>
    </div>
  <% end %>
</div>
```

---

## 六、列表筛选实现

### 6.1 搜索模型

**交易搜索类** (`app/models/transaction/search.rb:15,34,139-142`)：
```ruby
class Transaction::Search
  attribute :tags, array: true

  def transactions_scope
    query = apply_tag_filter(query, tags)
  end

  def apply_tag_filter(query, tags)
    return query unless tags.present?
    query.joins(:tags).where(tags: { name: tags })
  end
end
```

#### 6.1.1 查询层匹配语义

**核心逻辑**：`joins(:tags).where(tags: { name: tags })`

**SQL 语义解析**：
```sql
SELECT transactions.*
FROM transactions
INNER JOIN taggings ON taggings.taggable_id = transactions.id 
                   AND taggings.taggable_type = 'Transaction'
INNER JOIN tags ON tags.id = taggings.tag_id
WHERE tags.name IN ('标签A', '标签B', ...)
```

**匹配规则**：
- 使用 `INNER JOIN`（通过 `joins` 方法）而非 `LEFT JOIN`
- **多标签命中方式**：`OR` 语义（只要命中任意一个选中的标签即可）
- **结果影响**：
  - 无标签的交易**不会**出现在结果中（因为 INNER JOIN 排除了无匹配的行）
  - 如果一个交易同时有多个选中的标签，会出现**重复行**（需要后续去重）
  - 没有处理重复问题，依赖上层查询或分页逻辑隐式去重

**与其他筛选器的对比**：
| 筛选器 | JOIN 类型 | 特殊处理 |
|--------|----------|----------|
| 标签筛选 | `INNER JOIN` (`joins`) | 无标签交易被排除 |
| 分类筛选 | `LEFT JOIN` (`left_joins`) | 特别处理 "Uncategorized" 情况 |
| 商家筛选 | `INNER JOIN` (`joins`) | 无商家交易被排除 |

### 6.2 筛选器 UI

**标签筛选组件** (`app/views/transactions/searches/filters/_tag_filter.html.erb:1`)：
```erb
<div data-controller="list-filter">
  <div class="relative">
    <input type="search" placeholder="Filter tags"
           data-list-filter-target="input"
           data-action="input->list-filter#filter">
    <%= icon("search") %>
  </div>
  <div data-list-filter-target="list">
    <% Current.family.tags.alphabetically.each do |tag| %>
      <div class="filterable-item" data-filter-name="<%= tag.name %>">
        <%= form.check_box :tags, { multiple: true, checked: @q[:tags]&.include?(tag.name) }, tag.name, nil %>
        <%= form.label :tags, value: tag.name do %>
          <%= render DS::FilledIcon.new(
            variant: :text,
            hex_color: tag.color || Tag::UNCATEGORIZED_COLOR,
            text: tag.name,
            size: "sm",
            rounded: true
          ) %>
          <%= tag.name %>
        <% end %>
      </div>
    <% end %>
  </div>
</div>
```

**关键点**：
- 筛选器中使用的是 `DS::FilledIcon` 组件，不是颜色头像 partial
- `DS::FilledIcon` 是设计系统通用组件，使用 `oklab` 色彩空间混合
- 颜色头像 partial 仅用于标签管理页面和标签编辑表单

**筛选徽章展示**：选中的标签以徽章形式显示在搜索结果顶部：
```erb
# app/views/transactions/searches/_search.html.erb:19-21
<% Array(param_value).each do |value| %>
  <%= render partial: "transactions/searches/filters/badge", locals: { param_key: param_key, param_value: value } %>
<% end %>
```

---

## 七、Hotwire 与 Stimulus 控件消费方式

### 7.1 路径一：标签创建

**完整链路**：
```
用户点击"新建标签"按钮 (tags/index.html.erb:15-21)
  ↓
Turbo Frame 加载模态框 (data: { turbo_frame: "modal" })
  ↓
渲染 tags/new.html.erb → 嵌套 _form.html.erb
  ↓
color-avatar Stimulus 控制器激活 (data-controller="color-avatar")
  ↓
用户选择颜色 → handleColorChange() 实时更新头像预览
  ↓
用户输入名称 → 绑定到 color-avatar 控制器的 name target
  ↓
提交表单 (data: { turbo_frame: :_top })
  ↓
TagsController#create → 保存到数据库
  ↓
整页重定向到标签列表 (redirect_to tags_path)
```

**Hotwire 角色**：
- `turbo_frame: "modal"`：通过 Turbo Frame 加载模态框内容，无需整页刷新
- `turbo_frame: :_top`：表单提交后整页导航，确保页面状态同步

**Stimulus 角色**：
- `color-avatar` 控制器：提供颜色和名称的实时预览，无需服务器往返

### 7.2 路径二：交易打标

**完整链路**：
```
用户点击交易条目 (transactions/_transaction.html.erb:49-57)
  ↓
Turbo Frame 抽屉加载 (data: { turbo_frame: "drawer" })
  ↓
渲染 transactions/show.html.erb 抽屉界面
  ↓
auto-submit-form Stimulus 控制器激活 (data: { controller: "auto-submit-form" })
  ↓
标签多选器绑定 (data-auto-submit-form-target="auto")
  ↓
用户选择标签 → 控制器自动触发表单提交
  ↓
TransactionsController#update → 更新 tag_ids 关联
  ↓
Turbo Stream 响应 (app/controllers/transactions_controller.rb:94-104)
  ↓
局部替换：交易头部 + 交易条目 + Flash 通知
```

**Hotwire 角色**：
- `turbo_frame: "drawer"`：通过 Turbo Frame 加载抽屉详情页
- `turbo_stream.replace`：更新后局部替换 DOM，无需整页刷新
- `turbo_stream.replace(@entry)`：替换交易列表中的对应条目

**Stimulus 角色**：
- `auto-submit-form` 控制器：监听表单字段变化，自动提交保存，实现无感体验

### 7.3 路径三：列表筛选

**完整链路**：
```
打开交易列表页面 (transactions/index.html.erb:53)
  ↓
渲染搜索表单 (transactions/searches/_form.html.erb:1)
  ↓
auto-submit-form 控制器激活 (data: { controller: "auto-submit-form" })
  ↓
点击筛选按钮 → 弹出筛选菜单
  ↓
选择标签筛选 → 渲染 _tag_filter.html.erb
  ↓
list-filter 控制器激活 (data-controller="list-filter")
  ↓
用户搜索标签 → filter() 方法前端实时过滤
  ↓
用户勾选标签 → auto-submit-form 自动提交表单 (GET 请求)
  ↓
TransactionsController#index → Transaction::Search 应用筛选
  ↓
整页刷新，展示筛选结果
  ↓
顶部显示已选标签徽章 (_search.html.erb:19-21)
```

**Hotwire 角色**：
- 常规 GET 请求刷新页面（不是 Turbo Stream）
- 筛选表单使用 `method: :get`，通过 URL 参数传递筛选条件

**Stimulus 角色**：
- `list-filter` 控制器：在标签筛选列表中提供实时搜索过滤，纯前端操作
- `auto-submit-form` 控制器：筛选条件变化时自动提交搜索表单

---

## 八、完整数据流

### 8.1 标签创建流程

```
用户点击"新建标签" → Turbo Frame 加载模态框 → 表单展示 (color-avatar 控制器)
    → 选择颜色 (Stimulus 实时预览) → 输入名称 → 提交表单
    → TagsController#create → 保存到数据库 → 整页重定向到标签列表
```

### 8.2 交易打标签流程

```
打开交易详情 → Turbo Frame 抽屉加载 → 标签多选器展示家庭所有标签
    → 用户选择标签 → auto-submit-form 控制器自动提交
    → TransactionsController#update → 更新 tag_ids 关联
    → Turbo Stream 局部替换 → 交易列表实时更新
```

### 8.3 标签筛选交易流程

```
打开交易搜索页面 → 展开标签筛选器 → list-filter 控制器实时过滤标签列表
    → 用户勾选标签 → auto-submit-form 自动提交 GET 请求
    → Transaction::Search 应用标签筛选 (joins(:tags).where)
    → 整页刷新 → 顶部显示已选标签徽章
```

---

## 九、关键设计特点

| 设计点 | 说明 |
|--------|------|
| **家庭级归属** | 标签是家庭共享资源，而非用户私有 |
| **多态关联** | 通过 `taggings` 中间表支持未来扩展到其他模型 |
| **颜色双轨渲染** | 颜色头像 partial 使用 `srgb`，DS::FilledIcon 使用 `oklab` |
| **颜色承载** | 颜色直接存储在标签表，渲染时通过 CSS `color-mix()` 动态计算 |
| **实时交互** | Stimulus 控制器实现前端实时过滤和预览，无需服务器往返 |
| **渐进式增强** | 标签创建用整页刷新，交易打标用 Turbo Stream 局部更新，筛选用 GET 刷新 |
| **自动保存** | 配合 `auto-submit-form` 实现无感保存体验 |
