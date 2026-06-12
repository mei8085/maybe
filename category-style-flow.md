# Category Icon & Color — 数据流代码理解记录

> 本文档追踪 `Category` 模型的 `color` 与 `lucide_icon` 两个字段，从编辑表单、持久化到列表展示的完整链路，并重点厘清：
> 1. 子分类对 color / lucide_icon 的处理边界（**只继承颜色不继承图标**）
> 2. 各场景下「复用通用分类徽章 _badge」与「独立渲染」的呈现分工
> 3. 虚拟分类的设计作用
> 4. **修訂**：子分类表单里仅颜色区被隐藏，图标区可见且可修改（v2 修正）
> 所有行号均基于源码当前版本。

---

## 1 数据模型层：字段定义与约束

### 1.1 数据库 Schema

`categories` 表在 [schema.rb](file:///d:/fz/0601-1/solo-dogfeeding/code/34-maybe/db/schema.rb#L162-L172) 中定义了两个关键字段：

| 字段 | 类型 | 默认值 | 约束 |
|---|---|---|---|
| `color` | `string` | `"#6172F3"` | `NOT NULL` |
| `lucide_icon` | `string` | `"shapes"` | `NOT NULL` |

### 1.2 Model 层

在 [category.rb](file:///d:/fz/0601-1/solo-dogfeeding/code/34-maybe/app/models/category.rb#L1-L129) 中：

- **验证**（第 11 行）：`validates :name, :color, :lucide_icon, :family, presence: true`
- **预设色板**（第 24 行）：`COLORS = %w[#e99537 #4da568 #6471eb #db5a54 #df4e92 #c44fe9 #eb5429 #61c9ea #805dee #6ad28a]`
- **预设图标**（第 49-51 行）：`icon_codes` 类方法返回约 45 个 Lucide 图标名称
- **颜色继承回调**（第 17 行 & 第 92-96 行）：`before_save :inherit_color_from_parent`
- **虚拟分类**（第 63-69 行）：`uncategorized` 类方法创建不持久化实例

---

## 2 编辑表单层：用户输入

### 2.1 入口视图

新建和编辑均通过 `DS::Dialog` 弹出模态框，两者共用同一个 `_form` partial：

- [new.html.erb](file:///d:/fz/0601-1/solo-dogfeeding/code/34-maybe/app/views/categories/new.html.erb#L1-L6)
- [edit.html.erb](file:///d:/fz/0601-1/solo-dogfeeding/code/34-maybe/app/views/categories/edit.html.erb#L1-L6)

### 2.2 表单结构

[_form.html.erb](file:///d:/fz/0601-1/solo-dogfeeding/code/34-maybe/app/views/categories/_form.html.erb#L1-L78) 是核心表单，挂载 Stimulus `category` controller：

```erb
<div data-controller="category" data-category-preset-colors-value="<%= Category::COLORS %>">
```

`Category::COLORS` 数组通过 `data-category-preset-colors-value` 传入 Stimulus controller，供客户端判断当前颜色是否为预设色。

**颜色选择区**（第 17-41 行）：

| 元素 | 作用 |
|---|---|
| 预设色圆点 | 遍历 `Category::COLORS`，渲染 `f.radio_button :color, color`，绑定 `change->category#handleColorChange` |
| 自定义色按钮 | `f.radio_button :color, "custom-color"`，点击后打开 Pickr 调色板 |
| `f.text_field :color` | 隐藏的真实表单字段，由 Stimulus controller 动态更新 value |
| `colorPreview` target | 颜色预览圆点，实时反映当前选色 |

**图标选择区**（第 43-55 行）：

遍历 `Category.icon_codes`，每个图标渲染为 `f.radio_button :lucide_icon, icon`，绑定 `change->category#handleIconChange change->category#handleIconColorChange` 双事件。

**头像实时预览**：

[_color_avatar.html.erb](file:///d:/fz/0601-1/solo-dogfeeding/code/34-maybe/app/views/categories/_color_avatar.html.erb#L1-L8) 在表单顶部展示实时预览：

```erb
<span style="background-color: color-mix(in oklab, <%= category.color %> 10%, transparent); color: <%= category.color %>">
  <%= icon(category.lucide_icon, size: "2xl", color: "current") %>
</span>
```

用 `color-mix(in oklab, ... 10%, transparent)` 生成 10% 透明度的背景色，前景色为 `category.color`。

### 2.3 Stimulus Controller 交互逻辑

[category_controller.js](file:///d:/fz/0601-1/solo-dogfeeding/code/34-maybe/app/javascript/controllers/category_controller.js#L1-L262) 管理所有客户端交互：

| 方法 | 触发时机 | 作用 |
|---|---|---|
| `handleColorChange`（第 126 行） | 预设色单选 | 更新 `colorInput` value、`colorPreview` 背景、头像颜色、选中图标颜色 |
| `initPicker`（第 52 行）+ Pickr `change` 回调（第 69 行） | 自定义调色板选色 | 提取 HEX → 更新 `colorInput` value → 计算对比度 → 对比度不足时显示警告 |
| `handleIconChange`（第 107 行） | 图标单选 | 克隆 SVG → 插入头像预览区 |
| `handleIconColorChange`（第 92 行） | 图标单选 | 记录 `selectedIcon` → 清除其他图标高亮 → 为选中图标着色 |
| `autoAdjust`（第 147 行） | 点击 auto-adjust | 循环减暗 RGB 值直到对比度 ≥ 4.5 |
| `handleParentChange`（第 153 行） | 父分类变更 | 子分类时隐藏颜色/图标选择区 |

**对比度校验**：自定义颜色时，控制器计算前景色与 10% 透明背景的 WCAG 对比度。若 < 4.5，通过 `setCustomValidity` 阻止表单提交，并显示 "Poor contrast" 提示。

---

## 3 持久化层：Controller → Model → Database

### 3.1 CategoriesController

[categories_controller.rb](file:///d:/fz/0601-1/solo-dogfeeding/code/34-maybe/app/controllers/categories_controller.rb#L1-L92)：

- **`new`**（第 12-15 行）：`Current.family.categories.new color: Category::COLORS.sample` —— 新建时随机选取一个预设色
- **`create`**（第 17-34 行）：`Current.family.categories.new(category_params)` → `@category.save`
- **`update`**（第 39-51 行）：`@category.update(category_params)`
- **`category_params`**（第 89-91 行）：`params.require(:category).permit(:name, :color, :parent_id, :classification, :lucide_icon)` —— `:color` 和 `:lucide_icon` 在白名单中

### 3.2 Model 回调干预

保存前，[category.rb 第 92-96 行](file:///d:/fz/0601-1/solo-dogfeeding/code/34-maybe/app/models/category.rb#L92-L96) 的 `before_save :inherit_color_from_parent` 回调会执行：

```ruby
def inherit_color_from_parent
  if subcategory?
    self.color = parent.color
  end
end
```

注意回调**只覆盖 `color`，不动 `lucide_icon`**。详见第 5 节。

---

## 4 列表展示层：复用通用徽章 vs 独立渲染

整个系统中，color + lucide_icon 的呈现分为两大路径：**复用 `_badge.html.erb`** 或 **各视图独立渲染**。

### 4.1 通用分类徽章 `_badge.html.erb`

[_badge.html.erb](file:///d:/fz/0601-1/solo-dogfeeding/code/34-maybe/app/views/categories/_badge.html.erb#L1-L15) 是**通用徽章组件**，提供一套固定的胶囊形视觉：

```erb
<span style="
  background-color: color-mix(in oklab, <%= category.color %> 10%, transparent);
  border-color: color-mix(in oklab, <%= category.color %> 10%, transparent);
  color: <%= category.color %>;">
  <% if category.lucide_icon.present? %>
    <%= icon category.lucide_icon, size: "sm", color: "current" %>
  <% end %>
  <%= category.name %>
</span>
```

渲染规则：
- **背景色**：`color-mix(in oklab, <color> 10%, transparent)` —— 10% 透明度的分类色
- **边框色**：同背景色
- **文字 & 图标色**：`<color>` —— 100% 分类色
- **图标**：若 `lucide_icon` 存在，渲染 Lucide SVG，颜色设为 `current`（继承父元素 `color`）

### 4.2 复用 `_badge` 的场景

以下场景**直接调用 `render "categories/badge"`**，所有视觉参数由 badge 内部统一决定：

| 场景 | 视图文件 | 行号 | 说明 |
|---|---|---|---|
| 分类设置列表行 | [_category.html.erb](file:///d:/fz/0601-1/solo-dogfeeding/code/34-maybe/app/views/categories/_category.html.erb#L11) | 11 | 列表每行的分类标识 |
| 交易分类菜单按钮 | [_menu.html.erb](file:///d:/fz/0601-1/solo-dogfeeding/code/34-maybe/app/views/categories/_menu.html.erb#L5) | 5 | 交易行上作为下拉菜单触发按钮 |
| 交易 Transfer/Payment 徽章 | [_transaction_category.html.erb](file:///d:/fz/0601-1/solo-dogfeeding/code/34-maybe/app/views/transactions/_transaction_category.html.erb#L7) | 7 | 不可分类的转账/还款用虚拟分类渲染 |
| 交易分类下拉行 | [_row.html.erb](file:///d:/fz/0601-1/solo-dogfeeding/code/34-maybe/app/views/category/dropdowns/_row.html.erb#L24) | 24 | 分类下拉选择器每行 |
| 交易搜索筛选器 | [_category_filter.html.erb](file:///d:/fz/0601-1/solo-dogfeeding/code/34-maybe/app/views/transactions/searches/filters/_category_filter.html.erb#L19) | 19 | 高级搜索中可勾选的分类标签 |
| 证券交易行 | [_trade.html.erb](file:///d:/fz/0601-1/solo-dogfeeding/code/34-maybe/app/views/trades/_trade.html.erb#L33) | 33 | 交易记录中的分类徽章 |

**复用 badge 的共同特征**：
- 需要同时展示 `color`（背景/边框/文字）和 `lucide_icon`（图标）以及分类名称
- 作为紧凑的标签/徽章出现，不需要额外的数据可视化元素
- 允许传入虚拟分类（Category 实例但不落库），同样能正确渲染

### 4.3 独立渲染的场景

以下场景**不调用 `_badge`**，而是在各自模板中直接读取 `category.color` / `category.lucide_icon`，配合各自的图表或布局需求做独立呈现。

#### 4.3.1 预算分类列表行：圆形图标 + 数据

[_budget_category.html.erb](file:///d:/fz/0601-1/solo-dogfeeding/code/34-maybe/app/views/budget_categories/_budget_category.html.erb#L1-L54)：

```erb
<div class="w-8 h-8 ... rounded-full flex justify-center items-center"
     style="color: <%= budget_category.category.color %>">
  <% if budget_category.category.lucide_icon %>
    <%= icon(budget_category.category.lucide_icon, color: "current") %>
  <% else %>
    <%= render DS::FilledIcon.new(variant: :text, hex_color: budget_category.category.color, ...) %>
  <% end %>
</div>
```

独立原因：需要将图标放在 32px 圆形背景内（而非胶囊形 badge），并在图标旁展示剩余预算金额、实际支出等预算专属数据。图标降级为 DS::FilledIcon（首字母填充图标）。

#### 4.3.2 预算分类甜甜圈中心图标

[_budget_category_donut.html.erb](file:///d:/fz/0601-1/solo-dogfeeding/code/34-maybe/app/views/budget_categories/_budget_category_donut.html.erb#L1-L24)：

```erb
<div style="background-color: <%= hex_with_alpha(budget_category.category.color, 0.05) %>">
  <% if budget_category.category.lucide_icon %>
    <span style="color: <%= budget_category.category.color %>">
      <%= icon(budget_category.category.lucide_icon, size: "sm", color: "current") %>
    </span>
  <% else %>
    <span class="text-sm uppercase" style="color: <%= budget_category.category.color %>">
      <%= budget_category.category.name.first.upcase %>
    </span>
  <% end %>
</div>
```

独立原因：图标需要嵌入甜甜圈图（donut-chart）的中心圆内，背景使用 `hex_with_alpha(..., 0.05)`（5% 透明）而非 badge 的 `color-mix 10%`，图标缺失时降级为大写首字母而非隐藏。

> `hex_with_alpha` 定义在 [application_helper.rb 第 31-34 行](file:///d:/fz/0601-1/solo-dogfeeding/code/34-maybe/app/helpers/application_helper.rb#L31-L34)，将 0-1 的 alpha 转为 8 位 HEX。

#### 4.3.3 预算分类表单：色条 + 名称

[_budget_category_form.html.erb](file:///d:/fz/0601-1/solo-dogfeeding/code/34-maybe/app/views/budget_categories/_budget_category_form.html.erb#L1-L31) 与 [_uncategorized_budget_category_form.html.erb](file:///d:/fz/0601-1/solo-dogfeeding/code/34-maybe/app/views/budget_categories/_uncategorized_budget_category_form.html.erb#L1-L21)：

```erb
<div class="w-1 h-3 rounded-xl mt-1" style="background-color: <%= budget_category.category.color %>"></div>
<p><%= budget_category.category.name %></p>
```

独立原因：这里只需要 1×3 px 的细竖色条作为颜色标识（类似书签条），图标完全省略，右侧展示"中位数月支出"和金额输入框，与 badge 的胶囊形标签语义完全不同。

#### 4.3.4 实际支出摘要：进度条 + 图例色点

[_actuals_summary.html.erb](file:///d:/fz/0601-1/solo-dogfeeding/code/34-maybe/app/views/budgets/_actuals_summary.html.erb#L1-L57)：

```erb
<!-- 分段进度条 -->
<div style="background-color: <%= category_total.category.color %>; width: <%= category_total.weight %>"></div>
<!-- 图例色点 -->
<div class="w-2.5 h-2.5 rounded-full shrink-0" style="background-color: <%= category_total.category.color %>"></div>
```

独立原因：`color` 用于绘制堆叠进度条（按 weight 分配宽度）和 10px 圆形图例色点。这里是数据可视化，完全不需要 `lucide_icon` 和分类名的胶囊呈现。

#### 4.3.5 预算总览甜甜圈：颜色映射

[_budget_donut.html.erb](file:///d:/fz/0601-1/solo-dogfeeding/code/34-maybe/app/views/budgets/_budget_donut.html.erb#L37-L59)：

```erb
<div class="w-1 h-3 rounded-xl" style="background-color: <%= bc.category.color %>"></div>
<p class="text-sm text-secondary"><%= bc.category.name %></p>
```

独立原因：隐藏的 DOM 节点（由 donut-chart Stimulus controller 在 hover 时切换显示），用 1×3 px 色条 + 文字展示当前 hover 的分类详情。

此外，[budget_category.rb 第 78-93 行](file:///d:/fz/0601-1/solo-dogfeeding/code/34-maybe/app/models/budget_category.rb#L78-L93) 的 `to_donut_segments_json` 方法直接把 `category.color` 序列化进 JSON，由 donut-chart JS controller 用 CSS 变量绘制 SVG 扇形。

#### 4.3.6 分类设置列表行的子分类缩进图标

[_category.html.erb 第 5-9 行](file:///d:/fz/0601-1/solo-dogfeeding/code/34-maybe/app/views/categories/_category.html.erb#L5-L9)：

```erb
<% if category.subcategory? %>
  <span style="color: <%= category.color %>">
    <%= icon "corner-down-right", size: "sm", color: "current" %>
  </span>
<% end %>
```

独立原因：这是一个布局辅助图标，视觉上用 `category.color` 给 `corner-down-right` 箭头着色，用于提示"此分类是子分类"。它**不使用 category 自身的 `lucide_icon`**，而是使用固定图标。

#### 4.3.7 API 输出

[_transaction.json.jbuilder](file:///d:/fz/0601-1/solo-dogfeeding/code/34-maybe/app/views/api/v1/transactions/_transaction.json.jbuilder#L19-L29)：

```ruby
json.category do
  json.color transaction.category.color
  json.icon transaction.category.lucide_icon
end
```

独立原因：这是数据序列化而非视图渲染，`lucide_icon` 被重命名为 `icon`，交由前端消费者自行决定如何呈现。

### 4.4 分工总结表

| 场景 | 是否复用 `_badge` | 读取 color | 读取 lucide_icon | 原因 |
|---|---|---|---|---|
| 分类设置列表 | ✅ 是 | badge 内部 | badge 内部 | 标准胶囊标签 |
| 交易菜单按钮 | ✅ 是 | badge 内部 | badge 内部 | 标准胶囊标签 |
| 交易下拉行 | ✅ 是 | badge 内部 | badge 内部 | 标准胶囊标签 |
| 交易搜索筛选器 | ✅ 是 | badge 内部 | badge 内部 | 标准胶囊标签 |
| Transfer/Payment 虚拟分类 | ✅ 是 | badge 内部 | badge 内部 | 共享徽章 + 虚拟 Category 实例 |
| 证券交易行 | ✅ 是 | badge 内部 | badge 内部 | 标准胶囊标签 |
| 预算分类行（_budget_category） | ❌ 否 | 直接 | 直接 | 需要圆形图标 + 金额数据，非胶囊 |
| 预算甜甜圈中心 | ❌ 否 | `hex_with_alpha(..., 0.05)` | 直接 | 嵌入 donut 中心，降级为大写首字母 |
| 预算分类表单 | ❌ 否 | 直接 | ❌ 不读 | 只需 1px 色条标识 |
| 实际支出进度条 | ❌ 否 | 直接 | ❌ 不读 | 数据可视化，只需要颜色 |
| 预算总览甜甜圈 | ❌ 否 | 直接 | ❌ 不读 | donut 扇形 + hover 色条 |
| 子分类缩进图标 | ❌ 否 | 直接 | ❌ 用固定 `corner-down-right` | 布局辅助，不需要 category 自身图标 |
| API JSON | ❌ 否 | 直接序列化 | 重命名为 `icon` 序列化 | 数据交付，不做渲染 |

---

## 5 子分类处理边界：颜色继承，图标独立

### 5.1 回调实现的精确边界

[category.rb 第 17 行 & 第 92-96 行](file:///d:/fz/0601-1/solo-dogfeeding/code/34-maybe/app/models/category.rb#L92-L96)：

```ruby
before_save :inherit_color_from_parent

def inherit_color_from_parent
  if subcategory?
    self.color = parent.color
    # 注意：这里没有 self.lucide_icon = parent.lucide_icon
  end
end
```

`subcategory?` 判定条件为 `parent.present?`（第 109-111 行）。

**关键事实**：回调只覆盖 `self.color = parent.color`，对 `lucide_icon` 完全不做处理。这意味着：
- 子分类数据库中的 `color` 永远等于父分类的 `color`
- 子分类数据库中的 `lucide_icon` 是子分类自己保存的值，与父分类无关

### 5.2 数据库 Schema 佐证

[schema.rb 第 162-172 行](file:///d:/fz/0601-1/solo-dogfeeding/code/34-maybe/db/schema.rb#L162-L172) 中 `color` 和 `lucide_icon` 都是独立列，有各自的默认值（`#6172F3` / `shapes`），没有任何外键或关联声明暗示图标需要从父分类取值。

### 5.3 表单层配合

[_form.html.erb 第 15 行](file:///d:/fz/0601-1/solo-dogfeeding/code/34-maybe/app/views/categories/_form.html.erb#L15)：

```erb
<div data-category-target="selection" style="<%= "display:none;" if @category.subcategory? %>">
```

该 `div` 包裹了**颜色选择区 + 图标选择区**两个区域。当编辑子分类时，颜色和图标选择区整体被 `display:none` 隐藏。

但从实际持久化行为看，这个"一起隐藏"是 UI 层面的简化：保存时颜色会被父分类覆盖（无论表单提交什么），而图标由于没有对应的继承回调，如果前端被绕过并直接提交了不同的 `lucide_icon`，数据库中会保留子分类自己的图标值。不过在正常 UI 路径下，子分类提交的 `lucide_icon` 就是它已有的值（因为图标选择区被隐藏，用户无法修改）。

`handleParentChange` 方法（[category_controller.js 第 153-158 行](file:///d:/fz/0601-1/solo-dogfeeding/code/34-maybe/app/javascript/controllers/category_controller.js#L153-L158)）在用户为分类选择父级时，动态隐藏颜色+图标选择区。

### 5.4 视图层的呈现表现

由于 `color` 继承 + `lucide_icon` 独立，各视图对父子分类的呈现效果为：

| 视图 | 父分类效果 | 子分类效果 |
|---|---|---|
| `_badge.html.erb` | 胶囊背景=父颜色，图标=父图标 | 胶囊背景=父颜色（继承），图标=子自己的图标（独立） |
| `_budget_category.html.erb` 圆形图标 | 颜色=父颜色，图标=父图标 | 颜色=父颜色（继承），图标=子自己的图标（独立） |
| `_budget_category_donut.html.erb` | 背景=父颜色，中心=父图标/首字母 | 背景=父颜色（继承），中心=子自己的图标/首字母（独立） |
| 预算表单色条 | 竖条=父颜色 | 竖条=父颜色（继承），不展示图标 |
| 进度条/甜甜圈扇形 | 色值=父颜色 | 色值=父颜色（继承），不展示图标 |
| `_category.html.erb` 子分类缩进 | —— | `corner-down-right` 箭头着色=父颜色（继承） |

### 5.5 设计作用

1. **颜色一致性（视觉分组）**：父子分类共享颜色，保证在徽章、预算色条、进度条等所有颜色驱动的视觉中，子分类与其父分类看起来属于同一组。
2. **图标独立性（语义区分）**：子分类可以拥有独立图标，让"Food & Drink"下的"Coffee"、"Groceries"、"Restaurant"虽然都共享橙色背景，但各自用不同的图标（如 `coffee`、`shopping-cart`、`utensils`）在视觉上被区分。
3. **渲染零条件分支**：由于颜色已在 Model 层被子分类继承覆盖，所有视图只需 `category.color`，不需要 `if subcategory? use parent.color else category.color end`。同理，图标直接读 `category.lucide_icon` 即可获取子分类独立值。
4. **API 层语义一致**：JSON 输出中 `color` 对于子分类也是父分类颜色，前端消费方无需处理父子颜色映射。

---

## 6 虚拟分类设计

### 6.1 定义位置

| 虚拟分类 | 定义位置 | 颜色 | 图标 | 用途 |
|---|---|---|---|---|
| `uncategorized` | [category.rb 第 63-69 行](file:///d:/fz/0601-1/solo-dogfeeding/code/34-maybe/app/models/category.rb#L63-L69) | `#737373` (`UNCATEGORIZED_COLOR`) | `circle-dashed` | 未分类交易的默认显示 |
| `transfer_category` | [categories_helper.rb 第 2-7 行](file:///d:/fz/0601-1/solo-dogfeeding/code/34-maybe/app/helpers/categories_helper.rb#L2-L7) | `#444CE7` (`TRANSFER_COLOR`) | `arrow-right-left` | 转账交易 |
| `payment_category` | [categories_helper.rb 第 9-14 行](file:///d:/fz/0601-1/solo-dogfeeding/code/34-maybe/app/helpers/categories_helper.rb#L9-L14) | `#db5a54` (`PAYMENT_COLOR`) | `arrow-right` | 还款交易 |
| `trade_category` | [categories_helper.rb 第 16-19 行](file:///d:/fz/0601-1/solo-dogfeeding/code/34-maybe/app/helpers/categories_helper.rb#L16-L19) | `#e99537` (`TRADE_COLOR`) | 无（未设置 `lucide_icon`） | 证券交易 |

### 6.2 创建方式

虚拟分类通过 `Category.new` 创建，**不调用 `.save`**，因此不落库。它们不拥有 `id`，也不关联 `family`。

### 6.3 BudgetCategory 中的虚拟分类桥接

[budget_category.rb 第 31-46 行](file:///d:/fz/0601-1/solo-dogfeeding/code/34-maybe/app/models/budget_category.rb#L31-L46) 中：

```ruby
class << self
  def uncategorized
    new(id: Digest::UUID.uuid_v5(...), category: nil)
  end
end

def category
  super || budget.family.categories.uncategorized
end
```

`BudgetCategory.uncategorized` 创建的是一个 BudgetCategory 实例（非 Category），其 `category` 关联为 `nil`，但通过方法覆写（`def category`）在访问时自动回退到 `Category.uncategorized` 虚拟分类。这样预算视图中 `budget_category.category.color` 和 `budget_category.category.lucide_icon` 可以统一取值，无需判断 nil。

### 6.4 使用场景

- **`uncategorized`**：在 [categories_helper.rb 第 23 行](file:///d:/fz/0601-1/solo-dogfeeding/code/34-maybe/app/helpers/categories_helper.rb#L23) 的 `family_categories` 方法中被插入到分类列表首位，也作为 [_badge.html.erb 第 2 行](file:///d:/fz/0601-1/solo-dogfeeding/code/34-maybe/app/views/categories/_badge.html.erb#L2) 的默认回退值（`category ||= Category.uncategorized`）
- **`transfer_category` / `payment_category`**：在 [_transaction_category.html.erb 第 7 行](file:///d:/fz/0601-1/solo-dogfeeding/code/34-maybe/app/views/transactions/_transaction_category.html.erb#L7) 中，当交易属于 Transfer 且不可分类时使用
- **`trade_category`**：在 [_trade.html.erb 第 33 行](file:///d:/fz/0601-1/solo-dogfeeding/code/34-maybe/app/views/trades/_trade.html.erb#L33) 中为证券交易行提供分类徽章（注意它未设置 `lucide_icon`，badge 中 `if category.lucide_icon.present?` 为 false，只显示名称）
- **BudgetCategory::uncategorized**（桥接）：在 [_uncategorized_budget_category_form.html.erb](file:///d:/fz/0601-1/solo-dogfeeding/code/34-maybe/app/views/budget_categories/_uncategorized_budget_category_form.html.erb#L1-L21) 和预算分配进度等场景中使用，统一走 `budget_category.category.color` 访问

### 6.5 设计作用

1. **复用渲染管道**：虚拟分类与真实分类共享 `_badge.html.erb` 模板，同一套 `color` + `lucide_icon` 渲染逻辑无需条件分支。
2. **避免空值处理**：未分类交易不需要在视图层写 `if category.nil?` 的分支逻辑，而是传入 `uncategorized` 虚拟实例，模板直接读取 `.color` 和 `.lucide_icon`。BudgetCategory 甚至通过覆写 `category` 方法让 nil 关联自动桥接到虚拟分类。
3. **语义明确**：转账、还款、证券交易在业务上不属于用户自建分类，用虚拟分类赋予它们独立的视觉标识，与用户分类区分开来。
4. **常量集中管理**：`UNCATEGORIZED_COLOR`、`TRANSFER_COLOR`、`PAYMENT_COLOR`、`TRADE_COLOR` 定义在 [category.rb 第 26-29 行](file:///d:/fz/0601-1/solo-dogfeeding/code/34-maybe/app/models/category.rb#L26-L29)，方便全局引用和修改。

---

## 7 完整数据流图

```
┌──────────────────────────────────────────────────────────────────────┐
│                     1. 编辑表单层 (Form Layer)                         │
│                                                                      │
│  new.html.erb / edit.html.erb                                        │
│       └→ _form.html.erb (挂载 Stimulus category controller)           │
│            ├→ _color_avatar.html.erb ← 实时预览 (color + icon)        │
│            ├→ 预设色: Category::COLORS → radio :color                 │
│            ├→ 自定义色: Pickr → colorInput.value                      │
│            ├→ 图标: Category.icon_codes → radio :lucide_icon          │
│            ├→ 对比度校验 (WCAG ≥ 4.5)                                 │
│            └→ 子分类: 颜色+图标选择区整体 display:none                 │
└────────────────────────────┬─────────────────────────────────────────┘
                             │ form submit (color + lucide_icon)
                             ▼
┌──────────────────────────────────────────────────────────────────────┐
│                   2. 控制器层 (Controller Layer)                       │
│                                                                      │
│  CategoriesController#create / #update                                │
│    category_params: permit(:name, :color, :lucide_icon, ...)          │
└────────────────────────────┬─────────────────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────────────────┐
│                    3. 模型层 (Model Layer)                              │
│                                                                      │
│  Category                                                             │
│    validates :color, :lucide_icon, presence: true                     │
│    before_save :inherit_color_from_parent                             │
│      └→ subcategory? → self.color = parent.color                      │
│         ⚠ lucide_icon 不被覆盖，保留子分类独立值                       │
└────────────────────────────┬─────────────────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────────────────┐
│                  4. 数据库层 (Database Layer)                          │
│                                                                      │
│  categories table                                                     │
│    color:       string, default: "#6172F3", null: false               │
│    lucide_icon: string, default: "shapes",   null: false               │
│    子分类: color=父分类color, lucide_icon=子分类独立值                 │
└────────────────────────────┬─────────────────────────────────────────┘
                             │ 读取
                             ▼
┌──────────────────────────────────────────────────────────────────────┐
│                  5. 列表展示层 (Presentation Layer)                     │
│                                                                      │
│  ┌─ 复用 _badge.html.erb 的场景 ──────────────────────────────────┐  │
│  │ • 分类设置列表 _category.html.erb                               │  │
│  │ • 交易菜单 _menu.html.erb                                       │  │
│  │ • 交易 Transfer/Payment 虚拟分类                                │  │
│  │ • 分类下拉 _row.html.erb                                        │  │
│  │ • 交易搜索过滤器 _category_filter.html.erb                      │  │
│  │ • 证券交易 _trade.html.erb                                      │  │
│  │  渲染规则: bg=color-mix 10%, border=同, fg=color, 图标=lucide_icon│  │
│  └─────────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌─ 独立渲染的场景 ────────────────────────────────────────────────┐  │
│  │ 预算分类行 _budget_category.html.erb                             │  │
│  │   → 32px 圆形图标 (color 着色, lucide_icon / DS::FilledIcon)    │  │
│  │ 预算甜甜圈中心 _budget_category_donut.html.erb                   │  │
│  │   → hex_with_alpha 5% 背景 + lucide_icon / 大写首字母            │  │
│  │ 预算分类表单 _budget_category_form.html.erb                      │  │
│  │   → 1×3px 色条 (只取 color, 不展示 icon)                         │  │
│  │ 实际支出进度条 _actuals_summary.html.erb                         │  │
│  │   → 堆叠进度条 + 圆形图例色点 (只取 color)                        │  │
│  │ 预算总览甜甜圈 _budget_donut.html.erb                            │  │
│  │   → hover 色条 + to_donut_segments_json (只取 color)             │  │
│  │ 子分类缩进 _category.html.erb                                    │  │
│  │   → corner-down-right 固定图标 (用 color 着色, 不用 lucide_icon)  │  │
│  │ API JSON _transaction.json.jbuilder                              │  │
│  │   → { color, icon } 序列化交付                                    │  │
│  └─────────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ─── 子分类读取规则 ───                                               │
│  color       → 永远等于父分类颜色 (保存时已被继承覆盖)                  │
│  lucide_icon → 子分类自身独立值                                       │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 8 关键设计要点总结

1. **颜色透明度公式统一**：徽章和表单头像使用 `color-mix(in oklab, <color> 10%, transparent)` 生成浅色背景；预算甜甜圈中心使用 `hex_with_alpha(<color>, 0.05)`（5% 透明）。二者透明度不同，对应胶囊标签 vs 甜甜圈中心圆的视觉密度需求。公式分别实现在 [_badge.html.erb](file:///d:/fz/0601-1/solo-dogfeeding/code/34-maybe/app/views/categories/_badge.html.erb#L7-L9)、[_color_avatar.html.erb](file:///d:/fz/0601-1/solo-dogfeeding/code/34-maybe/app/views/categories/_color_avatar.html.erb#L6) 和 [application_helper.rb 第 31-34 行](file:///d:/fz/0601-1/solo-dogfeeding/code/34-maybe/app/helpers/application_helper.rb#L31-L34)。

2. **子分类边界（颜色继承 / 图标独立）**：`before_save :inherit_color_from_parent` 只覆盖 `self.color = parent.color`，对 `lucide_icon` 不做处理。表单层将颜色和图标选择区一起隐藏是 UI 简化，但真正的语义边界在 Model 层。这使得父子分类在颜色上形成视觉分组，同时子分类可通过独立图标进行语义区分。

3. **_badge.html.erb 复用 vs 独立渲染的分工**：当场景需要"胶囊形标签 + 同时展示图标和名称"时走 badge；当场景涉及数据可视化（进度条、甜甜圈、色条）、特殊布局（圆形图标嵌入 donut）、或只需要 color 不需要 icon 时走独立渲染。预算模块是独立渲染的主要聚集区，预算视图有 6 处独立渲染，全部不走 badge。

4. **虚拟分类 + BudgetCategory 桥接**：`Category.uncategorized` 等虚拟实例让 badge 模板无需 nil 判断；`BudgetCategory` 通过覆写 `def category` 自动将 nil 关联桥接到虚拟分类，实现"预算视图统一通过 `budget_category.category.color` 取值，无需处理未分配预算"。

5. **对比度守卫**：Stimulus controller 计算自定义颜色与 10% 透明背景的 WCAG 对比度，不足 4.5 时阻止提交；badge 使用 10% 透明背景从另一层保障文字可读性。两者形成表单层与渲染层的配合。

6. **BudgetCategory::Group 委托**：[budget_category.rb 第 14-15 行](file:///d:/fz/0601-1/solo-dogfeeding/code/34-maybe/app/models/budget_category.rb#L14-L15) 使用 `delegate :name, :color, to: :category`，使预算分组可以直接 `group.color` 拿到分类颜色，而 `lucide_icon` 没有被 delegate，预算视图统一通过 `budget_category.category.lucide_icon` 访问。
