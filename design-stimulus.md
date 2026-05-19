# 设计系统组件与 Stimulus 控制器协作机制

## 1. 概述

本项目采用 **ViewComponent + Stimulus** 的架构模式，将设计系统组件的视图渲染与交互逻辑分离但紧密协作。Ruby 组件负责生成语义化 HTML 标记并通过 `data-*` 属性声明行为契约，Stimulus 控制器则负责在浏览器端接管交互状态管理。

## 2. 组件发现机制：标记驱动的自动连接

### 2.1 命名约定与文件结构

设计系统组件位于 `app/components/DS/`，每个交互组件通常包含三个文件：

```
app/components/DS/
├── dialog.rb              # Ruby 组件类
├── dialog.html.erb        # 模板（生成 HTML 标记）
└── dialog_controller.js   # Stimulus 控制器（可选，仅交互组件需要）
```

### 2.2 Importmap 配置实现自动发现

`config/importmap.rb` 第 8 行是关键配置：

```ruby
pin_all_from "app/components", under: "controllers", to: ""
```

这行配置将 `app/components` 目录下所有 `*_controller.js` 文件映射到 Stimulus 的控制器命名空间。配合 `app/javascript/controllers/index.js` 中的懒加载/预加载机制：

```javascript
import { eagerLoadControllersFrom } from "@hotwired/stimulus-loading";
eagerLoadControllersFrom("controllers", application);
```

Stimulus 会自动扫描 DOM，当发现带有 `data-controller="DS--dialog"` 的元素时，自动实例化对应的控制器。

### 2.3 控制器命名约定

| Ruby 组件路径 | Stimulus 标识符 | HTML data-controller 值 |
|--------------|----------------|------------------------|
| `app/components/DS/dialog_controller.js` | `DS--dialog` | `data-controller="DS--dialog"` |
| `app/components/DS/tabs_controller.js` | `DS--tabs` | `data-controller="DS--tabs"` |
| `app/components/DS/menu_controller.js` | `DS--menu` | `data-controller="DS--menu"` |

### 2.4 完整链路：从 Ruby 符号到 HTML 属性

#### DS__ 命名转换机制

Rails 的 `tag` 辅助方法和 `data:` 选项会自动处理命名转换。核心转换规则：

1. **双下划线 `__` → 双连字符 `--`**：用于标识命名空间
2. **单下划线 `_` → 单连字符 `-`**：用于分隔单词
3. **自动添加 `data-` 前缀**：所有 `data:` 哈希中的键都会加上 `data-` 前缀

**转换示例：**

| Ruby 符号 | 生成的 HTML 属性 |
|----------|-----------------|
| `controller: "DS--tabs"` | `data-controller="DS--tabs"` |
| `DS__tabs_target: "navBtn"` | `data-DS--tabs-target="navBtn"` |
| `DS__tabs_session_key_value: "accounts_tab"` | `data-DS--tabs-session-key-value="accounts_tab"` |
| `DS__tabs_nav_btn_active_class: "bg-white..."` | `data-DS--tabs-nav-btn-active-class="bg-white..."` |

**代码示例（Tabs 组件）：**

```ruby
# app/components/DS/tabs.html.erb:1-8
<%= tag.div data: {
  controller: "DS--tabs",
  DS__tabs_session_key_value: session_key,
  DS__tabs_url_param_key_value: url_param_key,
  DS__tabs_nav_btn_active_class: active_btn_classes,
  DS__tabs_nav_btn_inactive_class: inactive_btn_classes
} do %>
```

生成的 HTML：
```html
<div data-controller="DS--tabs"
     data-DS--tabs-session-key-value="accounts_tab"
     data-DS--tabs-url-param-key-value="tab"
     data-DS--tabs-nav-btn-active-class="bg-white text-primary shadow-sm"
     data-DS--tabs-nav-btn-inactive-class="text-secondary hover:bg-surface-inset-hover">
  ...
</div>
```

#### 三层注入机制详解

```
┌─────────────────────────────────────────────────────────┐
│  第一层：data-controller - 控制器识别                     │
├─────────────────────────────────────────────────────────┤
│  Ruby:  data: { controller: "DS--tabs" }                │
│  HTML:  data-controller="DS--tabs"                       │
│  作用：  Stimulus 扫描到此属性时实例化 TabsController     │
└─────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────┐
│  第二层：data-target - 元素标记                           │
├─────────────────────────────────────────────────────────┤
│  Ruby:  data: { DS__tabs_target: "navBtn" }             │
│  HTML:  data-DS--tabs-target="navBtn"                    │
│  作用：  控制器通过 this.navBtnTargets 访问这些元素       │
└─────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────┐
│  第三层：data-value / data-class - 配置传递               │
├─────────────────────────────────────────────────────────┤
│  Ruby:  data: { DS__tabs_auto_open_value: true }        │
│  HTML:  data-DS--tabs-auto-open-value="true"             │
│  作用：  控制器通过 this.autoOpenValue 读取配置值         │
└─────────────────────────────────────────────────────────┘
```

#### Stimulus 侧的属性读取

控制器通过静态声明告诉 Stimulus 要读取哪些属性：

```javascript
// app/components/DS/tabs_controller.js:5-7
static classes = ["navBtnActive", "navBtnInactive"];  // 读取 data-DS--tabs-nav-btn-active-class
static targets = ["panel", "navBtn"];                   // 读取 data-DS--tabs-target="panel"
static values = { sessionKey: String, urlParamKey: String };  // 读取 data-DS--tabs-session-key-value
```

Stimulus 自动完成：
- `navBtnActive` → 查找 `data-DS--tabs-nav-btn-active-class`
- `sessionKeyValue` → 查找 `data-DS--tabs-session-key-value`
- `navBtnTargets` → 收集所有 `data-DS--tabs-target="navBtn"` 的元素

## 3. 控制器接管局部交互状态

### 3.1 Stimulus 核心概念映射

Stimulus 通过四个核心概念实现状态管理：

| 概念 | 声明方式 | 用途 | 示例 |
|-----|---------|------|------|
| **Targets** | `static targets = ["content"]` | 标记组件内关键 DOM 元素 | `data-DS--dialog-target="content"` |
| **Values** | `static values = { autoOpen: Boolean }` | 传递配置参数 | `data-DS--dialog-auto-open-value="true"` |
| **Classes** | `static classes = ["navBtnActive"]` | 传递视觉类名 | `data-DS--tabs-nav-btn-active-class="..."` |
| **Actions** | `data: { action: "DS--dialog#close" }` | 绑定事件处理 | `data-action="click->DS--tabs#show"` |

### 3.2 生命周期与状态初始化

以 `DS::Dialog` 为例，控制器的 `connect()` 方法在 DOM 插入时自动调用：

```javascript
// app/components/DS/dialog_controller.js:12-17
connect() {
  if (this.element.open) return;
  if (this.autoOpenValue) {
    this.element.showModal();  // 从服务端传递的 auto_open 参数决定初始状态
  }
}
```

状态完全由服务端渲染的 HTML 标记决定，控制器仅在浏览器端"激活"这些状态。

### 3.3 深入案例：Dialog ESC 关闭的完整链路

DS::Dialog 的 ESC 键关闭功能展示了**多控制器协作**模式，涉及三个层级的代码配合：

**第一步：Ruby 组件注入双控制器和配置** (`app/components/DS/dialog.rb:98-110`)
```ruby
def merged_opts
  data[:controller] = [ "DS--dialog", "hotkey", data[:controller] ].compact.join(" ")
  data[:DS__dialog_auto_open_value] = auto_open
  data[:DS__dialog_reload_on_close_value] = reload_on_close
  data[:action] = [ "mousedown->DS--dialog#clickOutside", data[:action] ].compact.join(" ")
  data[:hotkey] = "esc:DS--dialog#close"  # 声明快捷键映射
end
```

生成的 HTML 标记：
```html
<dialog data-controller="DS--dialog hotkey"
        data-hotkey="esc:DS--dialog#close"
        data-action="mousedown->DS--dialog#clickOutside">
  ...
</dialog>
```

**第二步：Hotkey 控制器安装监听** (`app/javascript/controllers/hotkey_controller.js:1-12`)
```javascript
import { install, uninstall } from "@github/hotkey";

export default class extends Controller {
  connect() {
    install(this.element);  // 解析 data-hotkey 属性，安装键盘事件监听
  }

  disconnect() {
    uninstall(this.element);
  }
}
```

`@github/hotkey` 库会：
1. 读取 `data-hotkey="esc:DS--dialog#close"` 属性
2. 解析出快捷键 `esc` 和目标动作 `DS--dialog#close`
3. 在 `document` 上安装 `keydown` 事件监听器
4. 当用户按下 ESC 键时，触发 Stimulus action 调用

**第三步：触发 Dialog 控制器的 close 方法**
```javascript
// app/components/DS/dialog_controller.js:26-32
close() {
  this.element.close();  // 调用原生 <dialog> API 关闭
  if (this.reloadOnCloseValue) {
    Turbo.visit(window.location.href);
  }
}
```

**完整链路图：**
```
用户按下 ESC 键
      ↓
@github/hotkey 库捕获 keydown 事件
      ↓
匹配 data-hotkey="esc:DS--dialog#close"
      ↓
触发 Stimulus action: DS--dialog#close
      ↓
DialogController.close() 执行
      ↓
this.element.close() 关闭对话框
```

### 3.4 状态变更示例：Tabs 切换

`DS::Tabs` 展示了完整的状态流转：

**Ruby 侧生成标记** (`app/components/DS/tabs.html.erb:1-8`):
```erb
<%= tag.div data: {
  controller: "DS--tabs",
  DS__tabs_session_key_value: session_key,
  DS__tabs_url_param_key_value: url_param_key,
  DS__tabs_nav_btn_active_class: active_btn_classes,
  DS__tabs_nav_btn_inactive_class: inactive_btn_classes
} do %>
```

**按钮绑定 Action** (`app/components/DS/tabs/nav.rb:14-17`):
```ruby
data: { id: id, action: "DS--tabs#show", DS__tabs_target: "navBtn" }
```

**控制器处理状态切换** (`app/components/DS/tabs_controller.js:9-41`):
```javascript
show(e) {
  const btn = e.target.closest("button");
  const selectedTabId = btn.dataset.id;
  
  // 更新按钮视觉状态
  this.navBtnTargets.forEach((navBtn) => {
    if (navBtn.dataset.id === selectedTabId) {
      navBtn.classList.add(...this.navBtnActiveClasses);
      navBtn.classList.remove(...this.navBtnInactiveClasses);
    } else {
      navBtn.classList.add(...this.navBtnInactiveClasses);
      navBtn.classList.remove(...this.navBtnActiveClasses);
    }
  });
  
  // 更新面板显示状态
  this.panelTargets.forEach((panel) => {
    panel.classList.toggle("hidden", panel.dataset.id !== selectedTabId);
  });
  
  // 可选：同步状态到 URL 或 Session
  if (this.urlParamKeyValue) { /* 更新 URL */ }
  if (this.sessionKeyValue) { /* 发送到服务端 */ }
}
```

## 4. 无 JavaScript 时的组件行为核对

### 4.1 渐进增强策略验证

| 组件 | 无 JS 时的表现 | 依赖 JS 的功能 | 结论 |
|-----|---------------|---------------|------|
| **DS::Disclosure** | 使用原生 `<details>` 元素，`open` 属性控制初始状态。点击 summary 可展开/折叠，完全正常工作。 | 无 | ✅ 完全可用 |
| **DS::Toggle** | 使用原生 `<input type="checkbox">` + CSS `peer-checked` 选择器实现开关效果。点击 label 可切换状态，表单提交时值正确。 | 无 | ✅ 完全可用 |
| **DS::Alert** | 纯展示组件，消息和图标正常渲染。 | 无 | ✅ 完全可用 |
| **DS::Button** | 原生 `<button>` 或 `<a>` 元素，点击可提交表单或跳转。 | `confirm` 选项依赖 Turbo JS 实现确认对话框。 | ⚠️ 基础功能可用，增强功能失效 |
| **DS::Link** | 原生 `<a>` 元素，点击可跳转。 | `turbo_frame` 等增强功能失效。 | ⚠️ 基础功能可用，增强功能失效 |
| **DS::Dialog** | `<dialog>` 元素。如果渲染时带有 `open` 属性，则显示；否则隐藏。**无法打开**（需要 `showModal()`）。关闭按钮的 `data-action` 失效。 | 打开/关闭动画、点击外部关闭、ESC 键关闭。 | ❌ 不可用（除非初始打开） |
| **DS::Menu** | 按钮可见，但内容区域有 `hidden` 类，**无法打开**。 | 打开/关闭、浮动定位、点击外部关闭、ESC 键关闭。 | ❌ 不可用 |
| **DS::Tabs** | 初始激活的 tab 内容可见，其他 tab 面板有 `hidden` 类。按钮可见但点击无反应，**无法切换**。 | Tab 切换、URL 参数同步、Session 同步。 | ❌ 不可用（只能看初始 tab） |
| **DS::Tooltip** | 图标可见，但提示内容有 `hidden` 类，**悬停不显示**。 | 悬停显示、浮动定位。 | ❌ 不可用 |

### 4.2 修正后的渐进增强结论

> **原结论修正**：并非所有组件在无 JavaScript 时都能正常工作。
>
> 实际情况分为三类：
> 1. **完全可用**：Disclosure、Toggle、Alert - 依赖原生 HTML/CSS
> 2. **基础可用**：Button、Link - 原生功能可用，JS 增强功能失效
> 3. **不可用**：Dialog、Menu、Tabs、Tooltip - 核心交互依赖 JS

这种设计是合理的权衡：
- 简单组件优先使用原生 HTML 语义
- 复杂交互组件接受 JS 依赖，因为它们的交互模式（如浮动定位、模态框）无法用纯 HTML/CSS 优雅实现

## 5. 视觉变体到行为的映射

### 5.1 变体定义模式

设计系统组件通过 `VARIANTS` 常量定义视觉变体，这是一个标准模式。以 `DS::Buttonish` 为例 (`app/components/DS/buttonish.rb:2-35`):

```ruby
VARIANTS = {
  primary: {
    container_classes: "text-inverse bg-inverse hover:bg-inverse-hover disabled:bg-gray-500 theme-dark:disabled:bg-gray-400",
    icon_classes: "fg-inverse"
  },
  destructive: {
    container_classes: "text-inverse bg-red-500 theme-dark:bg-red-400 hover:bg-red-600 theme-dark:hover:bg-red-500 disabled:bg-red-200 theme-dark:disabled:bg-red-600",
    icon_classes: "fg-white"
  },
  ghost: {
    container_classes: "text-primary bg-transparent hover:bg-gray-100 theme-dark:hover:bg-gray-700",
    icon_classes: "fg-gray"
  },
  # ... 更多变体
}.freeze
```

### 5.2 变体参数传递链

```
调用方传入 variant 参数
        ↓
Ruby 组件 initialize 接收并存储
        ↓
模板渲染时通过 VARIANTS 查找对应的 CSS 类
        ↓
类名通过 data-* 属性传递给 Stimulus 控制器（如需要）
        ↓
控制器使用这些类名进行状态切换
```

### 5.3 变体影响行为的三种方式

#### 方式一：纯视觉变体（CSS 驱动）

大多数变体仅影响视觉呈现，通过 CSS 类名实现，不涉及 JavaScript 逻辑变化。

`DS::Button` 的 `primary` vs `ghost` 变体仅在样式上不同，行为完全一致。

#### 方式二：变体改变 DOM 结构（间接影响行为）

有些变体会改变生成的 HTML 结构，从而间接改变交互行为。以 `DS::Menu` 为例 (`app/components/DS/menu.rb:26-37`):

```ruby
VARIANTS = %i[icon button avatar].freeze

def initialize(variant: "icon", ...)
  @variant = variant.to_sym
  raise ArgumentError unless VARIANTS.include?(@variant)
end
```

模板根据变体渲染不同的触发按钮 (`app/components/DS/menu.html.erb:2-12`):

```erb
<% if variant == :icon %>
  <%= render DS::Button.new(variant: "icon", icon: "more-horizontal", data: { DS__menu_target: "button" }) %>
<% elsif variant == :button %>
  <%= button %>
<% elsif variant == :avatar %>
  <button data-DS--menu-target="button">
    <!-- avatar 内容 -->
  </button>
<% end %>
```

虽然变体改变了触发按钮的外观，但 Stimulus 控制器 (`menu_controller.js`) 的逻辑保持不变，因为它只依赖 `button` target 的存在，不关心其内部结构。

#### 方式三：变体传递行为参数

`DS::Dialog` 的 `modal` vs `drawer` 变体通过改变 CSS 类影响布局行为 (`app/components/DS/dialog.rb:69-96`):

```ruby
def dialog_outer_classes
  variant_classes = if drawer?
    "items-end justify-end"  # drawer 右下角对齐
  else
    "items-center justify-center"  # modal 居中显示
  end
end
```

### 5.4 类名传递给控制器：Tabs 案例

当控制器需要根据状态切换视觉效果时，Ruby 组件将变体对应的类名通过 `data-*` 属性传递：

```ruby
# app/components/DS/tabs.rb:42-48
def active_btn_classes
  VARIANTS.dig(variant, :active_btn_classes)
end
```

```erb
<!-- app/components/DS/tabs.html.erb:6-7 -->
DS__tabs_nav_btn_active_class: active_btn_classes,
DS__tabs_nav_btn_inactive_class: inactive_btn_classes
```

```javascript
// app/components/DS/tabs_controller.js:5
static classes = ["navBtnActive", "navBtnInactive"];

// 使用时
navBtn.classList.add(...this.navBtnActiveClasses);
```

这种设计使控制器完全不依赖具体的类名字符串，所有视觉决策都在 Ruby 组件层完成。

## 6. 端到端变体映射案例

### 6.1 案例一：DS::Menu 变体到行为的完整链路

**调用方代码：**
```erb
<%# 变体 1: 图标菜单 %>
<%= render DS::Menu.new(variant: "icon", placement: "bottom-end") do |menu| %>
  <% menu.with_item(text: "Edit", href: edit_path) %>
<% end %>

<%# 变体 2: 头像菜单 %>
<%= render DS::Menu.new(variant: "avatar", avatar_url: user.avatar_url, placement: "right-start") do |menu| %>
  <% menu.with_item(text: "Settings", href: settings_path) %>
<% end %>
```

**Ruby 侧处理：**
```ruby
# app/components/DS/menu.rb:26-37
def initialize(variant: "icon", placement: "bottom-end", offset: 12, ...)
  @variant = variant.to_sym
  @placement = placement
  @offset = offset
end
```

**模板渲染（变体差异）：**
```erb
<!-- app/components/DS/menu.html.erb:1-12 -->
<%= tag.div data: { 
  controller: "DS--menu", 
  DS__menu_placement_value: placement, 
  DS__menu_offset_value: offset 
} do %>
  <% if variant == :icon %>
    <%= render DS::Button.new(variant: "icon", icon: "more-horizontal", 
          data: { DS__menu_target: "button" }) %>
  <% elsif variant == :avatar %>
    <button data-DS--menu-target="button">
      <img src="<%= avatar_url %>" class="w-9 h-9 rounded-full">
    </button>
  <% end %>
  ...
<% end %>
```

**生成的 HTML（变体 1: icon）：**
```html
<div data-controller="DS--menu" 
     data-DS--menu-placement-value="bottom-end"
     data-DS--menu-offset-value="12">
  <button data-DS--menu-target="button" class="hover:bg-gray-100 ...">
    <svg data-icon="more-horizontal">...</svg>
  </button>
  <div data-DS--menu-target="content" class="hidden">...</div>
</div>
```

**生成的 HTML（变体 2: avatar）：**
```html
<div data-controller="DS--menu" 
     data-DS--menu-placement-value="right-start"
     data-DS--menu-offset-value="12">
  <button data-DS--menu-target="button">
    <img src="/avatars/123.jpg" class="w-9 h-9 rounded-full">
  </button>
  <div data-DS--menu-target="content" class="hidden">...</div>
</div>
```

**控制器侧行为（统一逻辑）：**
```javascript
// app/components/DS/menu_controller.js:14-20
static targets = ["button", "content"];
static values = {
  show: Boolean,
  placement: { type: String, default: "bottom-end" },
  offset: { type: Number, default: 6 },
};

// 打开菜单时使用变体参数
update() {
  computePosition(this.buttonTarget, this.contentTarget, {
    placement: this.placementValue,  // "bottom-end" 或 "right-start"
    middleware: [offset(this.offsetValue), flip(), shift({ padding: 5 })],
  }).then(({ x, y }) => {
    Object.assign(this.contentTarget.style, {
      position: "fixed",
      left: `${x}px`,
      top: `${y}px`,
    });
  });
}
```

**映射总结：**

| 变体 | 触发按钮外观 | placement 值 | 菜单弹出位置 | 控制器逻辑 |
|-----|-------------|-------------|-------------|-----------|
| `icon` | ⋯ 图标按钮 | `bottom-end` | 按钮下方右对齐 | 完全相同 |
| `avatar` | 用户头像 | `right-start` | 头像右侧上对齐 | 完全相同 |
| `button` | 自定义按钮 | 可配置 | 可配置 | 完全相同 |

### 6.2 案例二：DS::Dialog 变体到行为的完整链路

**调用方代码：**
```erb
<%# 变体 1: Modal 模态框 %>
<%= render DS::Dialog.new(variant: "modal", width: "md") do |dialog| %>
  <% dialog.with_header(title: "确认删除") %>
  <% dialog.with_body do %>确定要删除这条记录吗？<% end %>
<% end %>

<%# 变体 2: Drawer 抽屉 %>
<%= render DS::Dialog.new(variant: "drawer", reload_on_close: true) do |dialog| %>
  <% dialog.with_header(title: "编辑设置") %>
  <% dialog.with_body do %><%= render "form" %><% end %>
<% end %>
```

**Ruby 侧处理：**
```ruby
# app/components/DS/dialog.rb:38-54
VARIANTS = %w[modal drawer].freeze
WIDTHS = { sm: "lg:max-w-[300px]", md: "lg:max-w-[550px]", ... }.freeze

def initialize(variant: "modal", auto_open: true, reload_on_close: false, width: "md", ...)
  @variant = variant.to_sym
  @auto_open = auto_open
  @reload_on_close = reload_on_close
  @width = width.to_sym
end

# 变体决定布局类
def dialog_outer_classes
  variant_classes = if drawer?
    "items-end justify-end"  # 底部对齐
  else
    "items-center justify-center"  # 居中
  end
end

# 变体决定尺寸类
def dialog_inner_classes
  variant_classes = if drawer?
    "lg:w-[550px] h-full"  # 全高，固定宽度
  else
    class_names("max-h-full", WIDTHS[width])  # 最大高度，可变宽度
  end
end

# 所有变体共享的 data 属性注入
def merged_opts
  data[:controller] = "DS--dialog"
  data[:DS__dialog_auto_open_value] = auto_open
  data[:DS__dialog_reload_on_close_value] = reload_on_close
end
```

**生成的 HTML（变体 1: modal）：**
```html
<dialog data-controller="DS--dialog"
        data-DS--dialog-auto-open-value="true"
        data-DS--dialog-reload-on-close-value="false"
        class="w-full h-full bg-transparent ...">
  <div class="flex h-full w-full items-center justify-center">
    <div class="flex flex-col bg-container rounded-xl max-h-full lg:max-w-[550px] ..."
         data-DS--dialog-target="content">
      ...
    </div>
  </div>
</dialog>
```

**生成的 HTML（变体 2: drawer）：**
```html
<dialog data-controller="DS--dialog"
        data-DS--dialog-auto-open-value="true"
        data-DS--dialog-reload-on-close-value="true"
        class="w-full h-full bg-transparent ...">
  <div class="flex h-full w-full items-end justify-end">
    <div class="flex flex-col bg-container rounded-xl lg:w-[550px] h-full ..."
         data-DS--dialog-target="content">
      ...
    </div>
  </div>
</dialog>
```

**控制器侧行为：**
```javascript
// app/components/DS/dialog_controller.js:5-10
static targets = ["content"]
static values = {
  autoOpen: { type: Boolean, default: false },
  reloadOnClose: { type: Boolean, default: false },
};

// 关闭时根据变体参数决定是否刷新
close() {
  this.element.close();
  if (this.reloadOnCloseValue) {  // modal 为 false，drawer 为 true
    Turbo.visit(window.location.href);
  }
}
```

**映射总结：**

| 变体 | 布局类 | 尺寸类 | 视觉效果 | reload_on_close | 关闭后行为 |
|-----|--------|--------|---------|----------------|-----------|
| `modal` | `items-center justify-center` | `max-h-full lg:max-w-[550px]` | 屏幕中央的模态框 | `false` | 直接关闭 |
| `drawer` | `items-end justify-end` | `lg:w-[550px] h-full` | 右下角对齐的全高面板 | `true` | 关闭后刷新页面 |

**Drawer 行为边界说明：**

源码中 `drawer` 变体**不包含任何过渡动画或滑入效果**。其行为边界：
- **仅通过 CSS 类控制静态布局**：`items-end justify-end` 使内容靠右下角对齐，`h-full` 使其占满整个视口高度
- **无 transition/transform 动画类**：查看 dialog 相关的所有 CSS 类，未发现 `transition`、`translate`、`duration` 等动画属性
- **打开/关闭行为与 Modal 完全相同**：都通过原生 `<dialog>` 的 `showModal()` / `close()` API 实现，无额外的 JS 动画逻辑
- **实际视觉效果**：大屏下是一个右侧固定宽度（550px）的全高面板，小屏下是全屏对话框，打开时直接显示，无滑入动画

## 7. 组件协作流程图

```
┌─────────────────────────────────────────────────────────────┐
│                    服务端 (Ruby/Rails)                       │
├─────────────────────────────────────────────────────────────┤
│  DS::Tabs.new(variant: :default, active_tab: "overview")    │
│         │                                                    │
│         ▼                                                    │
│  VARIANTS[:default] 查找类名配置                              │
│         │                                                    │
│         ▼                                                    │
│  生成 HTML 标记:                                             │
│  <div data-controller="DS--tabs"                              │
│       data-DS--tabs-nav-btn-active-class="bg-white ...">     │
│    <button data-action="DS--tabs#show"                       │
│            data-DS--tabs-target="navBtn">Overview</button>   │
│    <div data-DS--tabs-target="panel">...</div>               │
│  </div>                                                      │
└─────────────────────────────┬───────────────────────────────┘
                              │
                              ▼  HTTP 响应
┌─────────────────────────────┴───────────────────────────────┐
│                    浏览器端 (JavaScript)                     │
├─────────────────────────────────────────────────────────────┤
│  Stimulus 观察 DOM，发现 data-controller="DS--tabs"          │
│         │                                                    │
│         ▼                                                    │
│  实例化 TabsController，读取 targets/values/classes          │
│         │                                                    │
│         ▼                                                    │
│  connect() 钩子执行，初始化状态                               │
│         │                                                    │
│         ▼                                                    │
│  用户点击按钮 → 触发 action → show() 方法执行                 │
│         │                                                    │
│         ▼                                                    │
│  使用 this.navBtnActiveClasses 切换类名                      │
│  使用 this.panelTargets 切换显示/隐藏                        │
└─────────────────────────────────────────────────────────────┘
```

## 8. 关键设计原则

### 8.1 渐进增强（Progressive Enhancement）

组件按优先级采用不同策略：
- **优先原生**：Disclosure、Toggle 使用原生 HTML 元素
- **基础可用**：Button、Link 保证核心功能，JS 仅提供增强
- **接受依赖**：Dialog、Menu、Tabs、Tooltip 等复杂组件依赖 JS

### 8.2 标记即契约（Markup as Contract）

Ruby 组件与 Stimulus 控制器之间通过 `data-*` 属性建立明确的契约，而非通过 JavaScript 配置对象。这种方式使：
- 服务端可以完全控制组件的初始状态
- 模板变更不会意外破坏 JavaScript 逻辑
- 控制器可独立测试（只需提供符合契约的 HTML 片段）

### 8.3 关注点分离

| 层级 | 职责 |
|-----|------|
| Ruby 组件类 | 变体解析、参数验证、结构组织 |
| ERB 模板 | 生成语义化 HTML、声明 data-* 契约 |
| Stimulus 控制器 | 事件绑定、状态流转、DOM 操作 |

### 8.4 可组合性

组件可以嵌套使用，例如 `DS::Dialog` 内部使用 `DS::Button` 和 `DS::Disclosure`，每个子组件独立管理自己的 Stimulus 控制器。

### 8.5 视觉决策在上层

所有 CSS 类名决策都在 Ruby 组件层完成，控制器仅接收类名并在适当的时候应用。这意味着：
- 设计师可以修改 Ruby 组件中的 `VARIANTS` 配置来调整样式
- 无需修改 JavaScript 代码
- 保持了单源真值（Single Source of Truth）

## 9. 典型组件深度解析

### 9.1 DS::Menu - 浮动定位组件

**协作要点：**
- Ruby 侧传递 `placement` 和 `offset` 参数
- Stimulus 控制器使用 `@floating-ui/dom` 进行动态定位
- 状态（打开/关闭）完全由控制器管理，通过 `hidden` 类实现视觉切换

**数据流：**
```
Ruby: placement: "bottom-end", offset: 12
  ↓ (data 属性)
JS: this.placementValue, this.offsetValue
  ↓
Floating UI: computePosition(button, content, { placement, offset })
  ↓
DOM: style.left = `${x}px`, style.top = `${y}px`
```

### 9.2 DS::Tooltip - 悬停提示

**协作要点：**
- 采用与 Menu 相同的 Floating UI 定位方案
- 事件绑定在 `connect()` 中建立，`disconnect()` 中清理
- 使用 `mouseenter`/`mouseleave` 触发显示/隐藏

### 9.3 DS::Button - 纯视觉组件

**协作要点：**
- 无对应的 Stimulus 控制器
- 变体系统仅影响 CSS 类
- 可作为子组件被其他组件（如 Dialog、Menu）使用
- 支持 `confirm` 选项，通过 `turbo-confirm` 与 Turbo 表单确认机制集成

**turbo-confirm 完整证据链：**

1. **Ruby 侧注入 data 属性** (`app/components/DS/button.rb:28-30`)：
   ```ruby
   if confirm.present?
     data = data.merge(turbo_confirm: confirm.to_data_attribute)
   end
   ```
   生成的 HTML：`<button data-turbo-confirm="Are you sure?">...</button>`

2. **Turbo 框架消费**：Turbo 内置的表单处理逻辑会检测 `data-turbo-confirm` 属性，拦截表单提交或链接点击。

3. **自定义确认对话框** (`app/javascript/controllers/application.js:9-17`)：
   ```javascript
   Turbo.config.forms.confirm = (data) => {
     const confirmDialogController = application.getControllerForElementAndIdentifier(
       document.getElementById("confirm-dialog"),
       "confirm-dialog",
     );
     return confirmDialogController.handleConfirm(data);
   };
   ```
   项目将 Turbo 默认的浏览器 `confirm()` 替换为自定义的 `confirm-dialog` 控制器实现。

4. **对话框控制器处理** (`confirm_dialog_controller.js:8-25`)：`handleConfirm()` 方法返回 Promise，用户确认后 resolve 为 `true`，取消则为 `false`，Turbo 根据结果决定是否继续提交。

### 9.4 DS::Disclosure - 原生 HTML 优先

**协作要点：**
- 无控制器，完全使用原生 `<details>` 元素
- 使用 CSS `group-open:` 变体实现箭头旋转动画
- `open` 参数控制初始状态

## 10. 开发指南

### 10.1 新增交互组件的步骤

1. **创建 Ruby 组件类** (`app/components/DS/widget.rb`)
   - 继承 `DesignSystemComponent`
   - 定义 `VARIANTS` 常量（如需要）
   - 使用 `renders_one` / `renders_many` 声明插槽

2. **创建模板** (`app/components/DS/widget.html.erb`)
   - 根元素添加 `data-controller="DS--widget"`
   - 使用 `data-DS__widget_target: "name"` 标记关键元素（注意双下划线）
   - 使用 `data-DS__widget_*_value: value` 传递配置
   - 使用 `data: { action: "DS--widget#method" }` 绑定事件

3. **创建 Stimulus 控制器** (`app/components/DS/widget_controller.js`)
   - 声明 `static targets`、`static values`、`static classes`
   - 实现 `connect()` / `disconnect()` 生命周期钩子
   - 实现事件处理方法

4. **无需额外配置** - Importmap 和 Stimulus 自动加载会处理剩下的事情

### 10.2 新增视觉变体的步骤

1. 在组件的 `VARIANTS` 哈希中添加新条目
2. 定义对应的 CSS 类名
3. 在 `initialize` 中验证参数
4. 模板自动使用新变体，控制器逻辑无需修改

### 10.3 data 属性命名速查表

| 用途 | Ruby 写法 | HTML 结果 | JS 访问方式 |
|-----|----------|----------|------------|
| 控制器 | `controller: "DS--widget"` | `data-controller="DS--widget"` | 自动实例化 |
| Target | `DS__widget_target: "name"` | `data-DS--widget-target="name"` | `this.nameTarget` / `this.nameTargets` |
| Value | `DS__widget_count_value: 5` | `data-DS--widget-count-value="5"` | `this.countValue` |
| Class | `DS__widget_active_class: "bg-red"` | `data-DS--widget-active-class="bg-red"` | `this.activeClass` / `this.activeClasses` |
| Action | `action: "click->DS--widget#toggle"` | `data-action="click->DS--widget#toggle"` | 方法 `toggle()` 被调用 |

## 11. 代码索引

| 组件 | Ruby 类 | 模板 | 控制器 | 无 JS 可用性 |
|-----|---------|------|--------|-------------|
| Tabs | `app/components/DS/tabs.rb` | `tabs.html.erb` | `tabs_controller.js` | ❌ |
| Dialog | `app/components/DS/dialog.rb` | `dialog.html.erb` | `dialog_controller.js` | ❌ |
| Menu | `app/components/DS/menu.rb` | `menu.html.erb` | `menu_controller.js` | ❌ |
| Tooltip | `app/components/DS/tooltip.rb` | `tooltip.html.erb` | `tooltip_controller.js` | ❌ |
| Button | `app/components/DS/button.rb` | `button.html.erb` | 无 | ⚠️ |
| Link | `app/components/DS/link.rb` | `link.html.erb` | 无 | ⚠️ |
| Disclosure | `app/components/DS/disclosure.rb` | `disclosure.html.erb` | 无 | ✅ |
| Toggle | `app/components/DS/toggle.rb` | `toggle.html.erb` | 无 | ✅ |
| Alert | `app/components/DS/alert.rb` | `alert.html.erb` | 无 | ✅ |

**核心配置：**
- Importmap: `config/importmap.rb:8`
- Stimulus 入口: `app/javascript/controllers/index.js`
- Stimulus 应用: `app/javascript/controllers/application.js`
- 基类: `app/components/design_system_component.rb`
