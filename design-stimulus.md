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

## 3. 控制器接管局部交互状态

### 3.1 Stimulus 核心概念映射

Stimulus 通过三个核心概念实现状态管理：

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

### 3.3 状态变更示例：Tabs 切换

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

## 4. 视觉变体到行为的映射

### 4.1 变体定义模式

设计系统组件通过 `VARIANTS` 常量定义视觉变体，这是一个标准模式。以 `DS::Buttonish` 为例 (`app/components/DS/buttonish.rb:2-35`):

```ruby
VARIANTS = {
  primary: {
    container_classes: "text-inverse bg-inverse hover:bg-inverse-hover ...",
    icon_classes: "fg-inverse"
  },
  destructive: {
    container_classes: "text-inverse bg-red-500 hover:bg-red-600 ...",
    icon_classes: "fg-white"
  },
  ghost: {
    container_classes: "text-primary bg-transparent hover:bg-gray-100 ...",
    icon_classes: "fg-gray"
  },
  # ... 更多变体
}.freeze
```

### 4.2 变体参数传递链

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

### 4.3 变体影响行为的两种方式

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
    "items-end justify-end"  # drawer 从底部滑入
  else
    "items-center justify-center"  # modal 居中显示
  end
end
```

### 4.4 类名传递给控制器：Tabs 案例

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

## 5. 组件协作流程图

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

## 6. 关键设计原则

### 6.1 渐进增强（Progressive Enhancement）

所有组件在无 JavaScript 的情况下仍能正常工作（基础内容可见），Stimulus 仅增强交互体验。例如 `DS::Disclosure` 使用原生 `<details>` 元素，无需 JavaScript 即可展开/折叠。

### 6.2 标记即契约（Markup as Contract）

Ruby 组件与 Stimulus 控制器之间通过 `data-*` 属性建立明确的契约，而非通过 JavaScript 配置对象。这种方式使：
- 服务端可以完全控制组件的初始状态
- 模板变更不会意外破坏 JavaScript 逻辑
- 控制器可独立测试（只需提供符合契约的 HTML 片段）

### 6.3 关注点分离

| 层级 | 职责 |
|-----|------|
| Ruby 组件类 | 变体解析、参数验证、结构组织 |
| ERB 模板 | 生成语义化 HTML、声明 data-* 契约 |
| Stimulus 控制器 | 事件绑定、状态流转、DOM 操作 |

### 6.4 可组合性

组件可以嵌套使用，例如 `DS::Dialog` 内部使用 `DS::Button` 和 `DS::Disclosure`，每个子组件独立管理自己的 Stimulus 控制器。

## 7. 典型组件深度解析

### 7.1 DS::Menu - 浮动定位组件

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

### 7.2 DS::Tooltip - 悬停提示

**协作要点：**
- 采用与 Menu 相同的 Floating UI 定位方案
- 事件绑定在 `connect()` 中建立，`disconnect()` 中清理
- 使用 `mouseenter`/`mouseleave` 触发显示/隐藏

### 7.3 DS::Button - 纯视觉组件

**协作要点：**
- 无对应的 Stimulus 控制器
- 变体系统仅影响 CSS 类
- 可作为子组件被其他组件（如 Dialog、Menu）使用
- 支持 `confirm` 选项，通过 `turbo-confirm` 与 Rails UJS 集成

## 8. 开发指南

### 8.1 新增交互组件的步骤

1. **创建 Ruby 组件类** (`app/components/DS/widget.rb`)
   - 继承 `DesignSystemComponent`
   - 定义 `VARIANTS` 常量（如需要）
   - 使用 `renders_one` / `renders_many` 声明插槽

2. **创建模板** (`app/components/DS/widget.html.erb`)
   - 根元素添加 `data-controller="DS--widget"`
   - 使用 `data-DS--widget-target="name"` 标记关键元素
   - 使用 `data-DS--widget-*-value` 传递配置
   - 使用 `data-action="DS--widget#method"` 绑定事件

3. **创建 Stimulus 控制器** (`app/components/DS/widget_controller.js`)
   - 声明 `static targets`、`static values`、`static classes`
   - 实现 `connect()` / `disconnect()` 生命周期钩子
   - 实现事件处理方法

4. **无需额外配置** - Importmap 和 Stimulus 自动加载会处理剩下的事情

### 8.2 新增视觉变体的步骤

1. 在组件的 `VARIANTS` 哈希中添加新条目
2. 定义对应的 CSS 类名
3. 在 `initialize` 中验证参数
4. 模板自动使用新变体，控制器逻辑无需修改

## 9. 代码索引

| 组件 | Ruby 类 | 模板 | 控制器 |
|-----|---------|------|--------|
| Tabs | `app/components/DS/tabs.rb` | `tabs.html.erb` | `tabs_controller.js` |
| Dialog | `app/components/DS/dialog.rb` | `dialog.html.erb` | `dialog_controller.js` |
| Menu | `app/components/DS/menu.rb` | `menu.html.erb` | `menu_controller.js` |
| Tooltip | `app/components/DS/tooltip.rb` | `tooltip.html.erb` | `tooltip_controller.js` |
| Button | `app/components/DS/button.rb` | `button.html.erb` | 无 |
| Disclosure | `app/components/DS/disclosure.rb` | `disclosure.html.erb` | 无 |

**核心配置：**
- Importmap: `config/importmap.rb:8`
- Stimulus 入口: `app/javascript/controllers/index.js`
- 基类: `app/components/design_system_component.rb`
