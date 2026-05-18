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

**标签徽章** (`app/views/tags/_badge.html.erb:1`)：
```erb
<span class="border text-sm font-medium px-2.5 py-1 rounded-full content-center"
      style="
        background-color: color-mix(in srgb, <%= tag.color %> 5%, white);
        border-color: color-mix(in srgb, <%= tag.color %> 10%, white);
        color: <%= tag.color %>;">
  <%= tag.name %>
</span>
```

**技术要点**：
- 使用 CSS `color-mix()` 函数实现颜色透明化
- 背景色：颜色 + 5% 透明度混合白色
- 边框色：颜色 + 10% 透明度混合白色
- 文字色：直接使用标签颜色

**颜色头像** (`app/views/shared/color_avatar.html.erb`)：
在标签列表和筛选器中使用，展示颜色预览。

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
  <% Tag::COLORS.each do |color| %>
    <label class="relative">
      <%= f.radio_button :color, color, class: "sr-only peer", data: { action: "change->color-avatar#handleColorChange" } %>
      <div class="w-6 h-6 rounded-full cursor-pointer peer-checked:ring-2 peer-checked:ring-offset-2 peer-checked:ring-blue-500" style="background-color: <%= color %>"></div>
    </label>
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

### 6.2 筛选器 UI

**标签筛选组件** (`app/views/transactions/searches/filters/_tag_filter.html.erb:1`)：
```erb
<div data-controller="list-filter">
  <input type="search" placeholder="Filter tags"
         data-list-filter-target="input"
         data-action="input->list-filter#filter">

  <div data-list-filter-target="list">
    <% Current.family.tags.alphabetically.each do |tag| %>
      <div class="filterable-item" data-filter-name="<%= tag.name %>">
        <%= form.check_box :tags, { multiple: true, checked: @q[:tags]&.include?(tag.name) }, tag.name, nil %>
        <%= render DS::FilledIcon.new(hex_color: tag.color, text: tag.name, size: "sm", rounded: true) %>
        <%= tag.name %>
      </div>
    <% end %>
  </div>
</div>
```

**筛选徽章展示**：选中的标签以徽章形式显示在搜索结果顶部：
```erb
# app/views/transactions/searches/_search.html.erb:19-21
<% Array(param_value).each do |value| %>
  <%= render partial: "transactions/searches/filters/badge", locals: { param_key: param_key, param_value: value } %>
<% end %>
```

---

## 七、Hotwire 与 Stimulus 控件消费方式

### 7.1 Hotwire Turbo 框架

**模态框编辑**：标签新建/编辑通过 Turbo Frame 模态框加载：
```erb
# app/views/tags/index.html.erb:15-21
<%= render DS::Link.new(
  text: t(".new"),
  variant: "primary",
  href: new_tag_path,
  icon: "plus",
  frame: :modal
) %>
```

**实时更新**：交易详情抽屉使用 Turbo 框架，标签修改后自动刷新：
```erb
# app/views/transactions/show.html.erb:1
<%= render DS::Dialog.new(variant: "drawer") do |dialog| %>
```

### 7.2 Stimulus 控制器

#### 7.2.1 列表筛选控制器 (`app/javascript/controllers/list_filter_controller.js`)

```javascript
export default class extends Controller {
  static targets = ["input", "list", "emptyMessage"];

  filter() {
    const filterValue = this.inputTarget.value.toLowerCase();
    const items = this.listTarget.querySelectorAll(".filterable-item");

    items.forEach((item) => {
      const text = item.getAttribute("data-filter-name").toLowerCase();
      const shouldDisplay = text.includes(filterValue);
      item.style.display = shouldDisplay ? "" : "none";
    });
  }
}
```

**消费方式**：
- 在标签筛选器中，通过 `data-controller="list-filter"` 绑定
- 输入框通过 `data-list-filter-target="input"` 注册
- 列表项通过 `data-list-filter-target="list"` 和 `data-filter-name` 注册
- 实时过滤无需服务器请求

#### 7.2.2 颜色头像控制器 (`app/javascript/controllers/color_avatar_controller.js`)

```javascript
export default class extends Controller {
  static targets = ["name", "avatar", "selection"];

  handleColorChange(e) {
    const color = e.currentTarget.value;
    this.avatarTarget.style.backgroundColor = `color-mix(in srgb, ${color} 10%, transparent)`;
    this.avatarTarget.style.borderColor = `color-mix(in srgb, ${color} 10%, transparent)`;
    this.avatarTarget.style.color = color;
  }
}
```

**消费方式**：
- 标签新建/编辑表单中，通过 `data-controller="color-avatar"` 绑定
- 颜色单选按钮通过 `data-action="change->color-avatar#handleColorChange"` 触发
- 实时预览头像颜色变化

#### 7.2.3 自动提交表单控制器 (`app/javascript/controllers/auto_submit_form_controller.js`)

在交易详情中修改标签后自动提交保存，无需手动点击保存按钮。

---

## 八、完整数据流

### 8.1 标签创建流程

```
用户点击"新建标签" → Turbo Frame 加载模态框 → 表单展示 (color-avatar 控制器)
    → 选择颜色 (Stimulus 实时预览) → 输入名称 → 提交表单
    → TagsController#create → 保存到数据库 → 重定向到标签列表
```

### 8.2 交易打标签流程

```
打开交易详情 → 标签多选器展示家庭所有标签 → 用户选择标签
    → auto-submit-form 控制器自动提交 → TransactionsController#update
    → 更新 tag_ids 关联 → 自动保存
```

### 8.3 标签筛选交易流程

```
打开交易搜索页面 → 展开标签筛选器 → list-filter 控制器实时过滤标签列表
    → 用户勾选标签 → 提交搜索表单 → Transaction::Search 应用标签筛选
    → joins(:tags).where(tags: { name: tags }) → 返回筛选结果
    → 顶部显示已选标签徽章
```

---

## 九、关键设计特点

| 设计点 | 说明 |
|--------|------|
| **家庭级归属** | 标签是家庭共享资源，而非用户私有 |
| **多态关联** | 通过 `taggings` 中间表支持未来扩展到其他模型 |
| **颜色内嵌** | 颜色直接存储在标签表，渲染时通过 CSS `color-mix()` 动态计算 |
| **实时交互** | Stimulus 控制器实现前端实时过滤和预览，无需服务器往返 |
| **自动保存** | 配合 `auto-submit-form` 实现无感保存体验 |
