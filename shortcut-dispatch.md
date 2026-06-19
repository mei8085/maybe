# 快捷键触发命令与页面上下文分发机制分析

## 一、整体架构概览

本项目快捷键系统采用 **三层协同架构**：全局监听层 → 焦点/作用域过滤层 → 命令分发层。核心技术栈为 `@github/hotkey` v3.1.1 + Stimulus 控制器 + HTML5 Dialog。

```
用户按键
  ↓
[全局监听层] @github/hotkey (document.addEventListener("keydown"))
  ↓ RadixTrie 匹配快捷键序列
[焦点过滤层] isFormField() / data-hotkey-scope / <dialog> showModal()
  ↓ 命中有效目标
[命令分发层] fireDeterminedAction() → click()/focus() → Stimulus data-action
  ↓
业务控制器方法执行
```

---

## 二、全局监听层：@github/hotkey + Stimulus hotkey_controller

### 2.1 核心库：@github/hotkey

源码位置：[vendor/javascript/@github--hotkey.js](file:///d:/fz/0601-2/solo-dogfeeding/code/41-maybe/vendor/javascript/@github--hotkey.js)（v3.1.1，见 [importmap.rb:10](file:///d:/fz/0601-2/solo-dogfeeding/code/41-maybe/config/importmap.rb#L10)）

**数据结构 — RadixTrie（基数树）：**
- 使用前缀树存储所有已注册的快捷键组合，高效支持序列快捷键（如 `t t /`）
- `Leaf` 节点挂载一个或多个绑定了该快捷键的 DOM 元素
- `SequenceTracker` 维护按键序列路径，超时 `CHORD_TIMEOUT = 1500ms` 自动重置

**全局监听器注册：**
```javascript
// install() 中：首次注册快捷键时绑定全局 keydown
Object.keys(o.children).length === 0 && document.addEventListener("keydown", keyDownHandler);

// uninstall() 中：最后一个快捷键被移除时解绑
Object.keys(o.children).length === 0 && document.removeEventListener("keydown", keyDownHandler);
```

**keyDownHandler 核心流程：**
1. `event.defaultPrevented` → 已被阻止的事件直接跳过
2. `event.target instanceof Node` → 合法性校验
3. **焦点过滤**（见第三章）
4. `eventToHotkeyString(e)` → 将键盘事件归一化为标准字符串（如 `Control+k`、`Escape`）
5. 在 RadixTrie 中逐级匹配，命中 `Leaf` 节点后：
   - 反向遍历绑定元素，配合 `data-hotkey-scope` 选择合适目标
   - 调用 `fireDeterminedAction(r, a.path)` 触发动作
   - `e.preventDefault()` 阻止浏览器默认行为

### 2.2 Stimulus 包装层：hotkey_controller

源码位置：[hotkey_controller.js](file:///d:/fz/0601-2/solo-dogfeeding/code/41-maybe/app/javascript/controllers/hotkey_controller.js)

```javascript
export default class extends Controller {
  connect()    { install(this.element); }
  disconnect() { uninstall(this.element); }
  navigateBack(event) { window.history.back(); }
}
```

**设计意图：** 将 `@github/hotkey` 的生命周期与 Stimulus 控制器的 DOM 绑定周期对齐——元素挂载时注册快捷键，卸载时自动清理，防止内存泄漏和幽灵快捷键。

### 2.3 data-hotkey 属性的解析逻辑 — expandHotkeyToEdges

`install(element)` 被调用时，读取 `element.getAttribute("data-hotkey")`，交给 `expandHotkeyToEdges` 解析。

**解析规则（逐字符状态机）：**

| 字符 | 作用 | 含义 |
|---|---|---|
| 空格 ` ` | 序列步分隔符 | `t t /` → 三步序列 |
| 逗号 `,` | 替代键分隔符 | `k,K` → k 或 K |
| 加号 `+` | 组合键修饰符连接 | `Control+k` |
| 其他字符 | 键名的一部分 | 原样拼接到当前 token |

**关键点：冒号 `:` 不是分隔符，它只是键名字符串的一部分。** `data-hotkey` 属性本身**不包含动作声明**——动作由同元素上的 `data-action`（Stimulus 约定）决定，冒号后的内容不会被 @github/hotkey 解析为任何有意义的字段。

解析后每个 token 再经过 `normalizeHotkey()`，该函数只做两件事：
1. `localizeMod`：将 `Mod` 替换为 `Control`（Windows/Linux）或 `Meta`（macOS）
2. `sortModifiers`：按 `Control → Alt → Meta → Shift → 键名` 排序

**实际解析结果对照：**

| data-hotkey 值 | expandHotkeyToEdges 输出 | 说明 |
|---|---|---|
| `"Escape"` | `[["Escape"]]` | 单步序列，单键 |
| `"t t /"` | `[["t"], ["t"], ["/"]]` | 三步序列 |
| `"k,K,ArrowUp,ArrowLeft"` | `[["k", "K", "ArrowUp", "ArrowLeft"]]` | 四个替代键 |
| **`"esc:DS--dialog#close"`** | **`[["esc:DS--dialog#close"]]`** | **整个字符串被当作一个键名，冒号和 # 均无特殊含义** |

**`eventToHotkeyString` 对 Escape 键的输出：**
- 读取 `event.key`，值为 `"Escape"`
- 经过修饰符映射（无修饰符按下）和别名映射 `n = {" ":"Space","+":"Plus"}`（无 Escape 映射）
- 最终输出：**`"Escape"`**

**匹配结论：** RadixTrie 中注册的是 `"esc:DS--dialog#close"`，但键盘事件归一化为 `"Escape"`——二者永远不匹配。`data-hotkey="esc:DS--dialog#close"` 在 @github/hotkey 的路由中**是无效绑定**。冒号后的 `DS--dialog#close` 对 hotkey 库无意义，看起来是写代码时混淆了 `data-action` 语法（`controller#method`）和 `data-hotkey` 语法。

### 2.4 序列按键状态的重置机制 — 逐行代码追踪

源码位置：[@github--hotkey.js](file:///d:/fz/0601-2/solo-dogfeeding/code/41-maybe/vendor/javascript/@github--hotkey.js)

#### 2.4.1 三个核心变量

```javascript
const o = new RadixTrie;        // 根节点（固定不变）
let c = o;                      // 当前位置指针（每次按键后可能移动）
const a = new SequenceTracker({ // 序列追踪器
  onReset() { c = o }           // reset() 的回调：把 c 指回根节点
});
```

- `c` 是序列进度的"书签"——指向 RadixTrie 中当前匹配到的层级
- `a.reset()` 做两件事：清空 `_path` 数组 + 把 `c` 指回根节点 `o`

#### 2.4.2 keyDownHandler 完整逻辑展开

将 minified 代码展开为可读伪代码，**每一步标注 `c` 和 `a` 的变化**：

```javascript
function keyDownHandler(e) {
  // ── 前置过滤 ──
  if (e.defaultPrevented) return;                    // ① 已被阻止 → 退出
  if (!(e.target instanceof Node)) return;           // ② 非法 target → 退出
  if (isFormField(e.target)) {                       // ③ 表单字段过滤
    const t = e.target;
    if (!t.id) return;                               //    无 id → 退出
    if (!t.ownerDocument.querySelector(              //    无 scope 绑定 → 退出
      `[data-hotkey-scope="${t.id}"]`
    )) return;
  }
  // ⚠ 注意：③ 中的三个 return 都不调用 a.reset()
  //    序列状态保留，计时器继续跑

  // ── 核心匹配 ──
  const t = c.get(eventToHotkeyString(e));
  //        ↑ 从当前位置 c 查找按键对应的子节点

  if (t) {                                           // ④ 找到了子节点
    a.registerKeypress(e);                           //    _path 推入本次按键 + 重启 1500ms 计时
    c = t;                                           //    移动书签到子节点

    if (t instanceof Leaf) {                         // ⑤ 子节点是 Leaf（序列完成）
      const n = e.target;
      let i = false;
      let r;
      const s = isFormField(n);
      for (let e = t.children.length - 1; e >= 0; e -= 1) {
        r = t.children[e];
        const o = r.getAttribute("data-hotkey-scope");
        if (!s && !o || s && n.id === o) {           //    scope 匹配
          i = true;
          break;
        }
      }
      if (r && i) {
        fireDeterminedAction(r, a.path);             //    触发动作
        e.preventDefault();                          //    阻止默认行为
      }
      a.reset();                                     //    ← 无论是否触发动作，都重置！
                                                       //      a._path = [], c = o

    } else {                                         // ⑥ 子节点是 RadixTrie（中间节点）
      // 什么都不做！
      // c 已经指向中间节点，a._path 已记录本次按键
      // 等待下一次 keydown → 从 c（中间节点）继续查找
    }

  } else {                                           // ⑦ 没找到子节点（完全不匹配）
    a.reset();                                       //    ← 立即重置！
                                                       //      a._path = [], c = o
  }
}
```

#### 2.4.3 五种场景的状态变化对照

以项目中的序列快捷键 `t t /`（主题切换）为例，RadixTrie 结构：

```
根节点 o
  └── "t" → RadixTrie（中间节点）
        └── "t" → RadixTrie（中间节点）
              └── "/" → Leaf（挂载 theme#toggle 按钮）
```

| # | 用户操作 | `c.get(按键)` 结果 | 走哪条分支 | `c` 变化 | `a._path` 变化 | 是否重置 |
|---|---|---|---|---|---|---|
| 1 | 按 `t`（第 1 次） | 找到中间节点 `"t"` | ④→⑥ | `o → 中间节点1` | `[] → ["t"]` | ❌ 不重置，等待下一步 |
| 2 | 按 `t`（第 2 次） | 找到中间节点 `"t"` | ④→⑥ | `中间节点1 → 中间节点2` | `["t"] → ["t","t"]` | ❌ 不重置，等待下一步 |
| 3 | 按 `/` | 找到 Leaf | ④→⑤ | `中间节点2 → Leaf` | `["t","t","/"]` | ✅ `a.reset()` → `c=o, _path=[]` |
| 4 | 按 `t` 后按 `k` | `c="中间节点1"`, `.get("k")` 为 `undefined` | ⑦ | — | — | ✅ `a.reset()` → `c=o, _path=[]` |
| 5 | 按 `t` 后等 1500ms | 超时 | — | — | — | ✅ `a.reset()` → `c=o, _path=[]` |
| 6 | 按 `t` 后切到输入框按键 | `isFormField` return | ③ | 不变 | 不变 | ❌ 不重置，计时继续 |

**核心区分：分支 ④→⑥ vs 分支 ⑦**

- **④→⑥ 找到了子节点且是中间节点**：这是"序列进行中"——`c` 移动到子节点，`a._path` 记录按键，计时器重启，等待下一步
- **⑦ 完全没找到子节点**：这是"序列中断"——`a.reset()` 立即执行，`c` 回到根节点，`_path` 清空

二者的区别完全由 `c.get(eventToHotkeyString(e))` 的返回值决定：
- 返回 `RadixTrie 实例` → 进入 ④→⑥ 分支，序列继续
- 返回 `undefined` → 进入 ⑦ 分支，序列立即重置
- 返回 `Leaf 实例` → 进入 ④→⑤ 分支，动作触发后重置

#### 2.4.4 isFormField 过滤 return 不重置的隐含影响

代码中 `isFormField` 的三个 `return` 路径都不调用 `a.reset()`：

```javascript
if (isFormField(e.target)) {
  const t = e.target;
  if (!t.id) return;                               // ← 不重置
  if (!t.ownerDocument.querySelector(...)) return;  // ← 不重置
}
```

**实际后果：** 如果用户按了序列首键 `"t"`（`c` 指向中间节点），然后切到输入框打字：
1. 输入框内每次 keydown 都经过 ③ 的 `return` → `c` 仍停留在中间节点
2. `a._path` 仍为 `["t"]`，计时器在跑
3. 用户如果 1500ms 内切回非表单元素并按 `"t"` → 序列继续（从中间节点再推进一步）
4. 如果超时 → `a.reset()` 自动清理

这个行为是**有意设计**还是**遗漏**？从代码结构看，`isFormField` 过滤位于 RadixTrie 匹配之前，它只决定"是否进入匹配逻辑"，但不干预已有的序列状态。设计意图可能是：输入框内的按键不应打断用户之前在非输入区域启动的序列操作。

---

## 三、焦点限制与作用域策略

### 3.1 表单字段过滤（Form Field Guard）

`@github/hotkey` 内置 `isFormField(e)` 函数，判断当前焦点是否在可编辑区域：

```javascript
function isFormField(e) {
  if (!(e instanceof HTMLElement)) return false;
  const t = e.nodeName.toLowerCase();
  const n = (e.getAttribute("type") || "").toLowerCase();
  return t === "select" || t === "textarea" ||
    (t === "input" && n !== "submit" && n !== "reset" && n !== "checkbox"
      && n !== "radio" && n !== "file") ||
    e.isContentEditable;
}
```

**在 keyDownHandler 中的应用——全量短路逻辑：**
```javascript
if (isFormField(e.target)) {
  const t = e.target;
  if (!t.id) return;  // ① 表单字段无 id → 所有快捷键禁用，直接退出
  // ② 只有存在 data-hotkey-scope="该字段id" 的绑定元素时才继续
  if (!t.ownerDocument.querySelector(`[data-hotkey-scope="${t.id}"]`)) return;
}
```

**效果：** 用户在输入框输入时，全局导航类快捷键（如 `j/k`、`Escape` 关闭页面级菜单）不会误触发，避免打断输入流。

**边界条件：**
- `isFormField` 对 `contentEditable` 元素也返回 true，因此富文本编辑区同样会屏蔽全局快捷键
- 判断粒度是"全有或全无"——无法区分"应屏蔽的字母键"和"仍需响应的修饰键组合"（如 `Cmd+Enter`）

### 3.2 data-hotkey-scope 作用域限定

在 `Leaf` 节点的反向选择中（同一快捷键可能绑定了多个元素）：
```javascript
const s = isFormField(n);  // n = e.target（当前焦点元素）
for (let e = t.children.length - 1; e >= 0; e -= 1) {
  r = t.children[e];
  const o = r.getAttribute("data-hotkey-scope");
  if (!s && !o || s && n.id === o) { i = true; break; }
}
```

规则：
- **非表单焦点 (`s=false`)**：只匹配没有 `data-hotkey-scope` 属性的绑定元素（全局快捷键）
- **表单焦点 (`s=true`)**：只匹配 `data-hotkey-scope` 等于当前焦点元素 `id` 的绑定元素

**边界条件：** 如果同一快捷键同时有带 scope 和不带 scope 的绑定元素，表单焦点下只会选带 scope 的那个；非表单焦点下只会选不带 scope 的那个——两者互斥，不会同时触发。

### 3.3 HTML5 Dialog 的 Escape 关闭——两条并行路径

Dialog 组件使用原生 `<dialog>` 元素。

源码位置：
- [dialog.rb:98-110](file:///d:/fz/0601-2/solo-dogfeeding/code/41-maybe/app/components/DS/dialog.rb#L98-L110)（merged_opts 注入）
- [dialog_controller.js](file:///d:/fz/0601-2/solo-dogfeeding/code/41-maybe/app/components/DS/dialog_controller.js)（Stimulus 控制器）
- [dialog.html.erb](file:///d:/fz/0601-2/solo-dogfeeding/code/41-maybe/app/components/DS/dialog.html.erb)（模板）

在 [dialog.rb:102-106](file:///d:/fz/0601-2/solo-dogfeeding/code/41-maybe/app/components/DS/dialog.rb#L102-L106) 的 `merged_opts` 中自动注入：
```ruby
data[:controller] = [ "DS--dialog", "hotkey", data[:controller] ].compact.join(" ")
data[:action] = [ "mousedown->DS--dialog#clickOutside", data[:action] ].compact.join(" ")
data[:hotkey] = "esc:DS--dialog#close"
```

**渲染后的 `<dialog>` 元素上会同时出现：**
```html
<dialog data-controller="DS--dialog hotkey ..."
        data-action="mousedown->DS--dialog#clickOutside"
        data-hotkey="esc:DS--dialog#close"
        data-ds--dialog-auto-open-value="true"
        data-ds--dialog-reload-on-close-value="false">
```

**路径 A — 原生 `<dialog>` 行为（实际生效的路径）：**

`<dialog>.showModal()` 激活后，浏览器 UA 层面自动处理 Escape 键：
1. 用户按 Escape → 浏览器在 UA 层触发 `cancel` 事件
2. 若 `cancel` 未被 `preventDefault()` → 浏览器调用 `dialog.close()`
3. `close` 事件在 dialog 上触发
4. dialog 的 `open` 属性变为 `false`，页面其余部分解除 `inert`

**路径 B — @github/hotkey 路由（失效的路径）：**

1. 用户按 Escape → `keydown` 事件冒泡到 `document`
2. `keyDownHandler` 调用 `eventToHotkeyString(e)` → 产生 `"Escape"`
3. 在 RadixTrie 中查找 `"Escape"` → 找不到 `"esc:DS--dialog#close"`（见 2.3 节分析）→ 无匹配
4. 此路径**永远不会触发 `DS--dialog#close`**

**两条路径的关键差异：**

| | 路径 A（原生） | 路径 B（hotkey，失效） |
|---|---|---|
| 触发条件 | UA 层自动处理 | RadixTrie 匹配（实际不匹配） |
| 调用方法 | `dialog.close()`（浏览器内置） | `DS--dialog#close()`（Stimulus） |
| `reloadOnClose` | **不执行** | 会执行 `Turbo.visit()` |
| `e.preventDefault()` | 不涉及 | 会阻止原生 Escape 行为 |

**后果：** [dialog_controller.js:26-32](file:///d:/fz/0601-2/solo-dogfeeding/code/41-maybe/app/components/DS/dialog_controller.js#L26-L32) 中的 `reloadOnClose` 逻辑在用户按 Escape 时不会被执行：

```javascript
close() {
  this.element.close();
  if (this.reloadOnCloseValue) {  // ← 原生 Escape 关闭时这段被跳过
    Turbo.visit(window.location.href);
  }
}
```

### 3.4 Dialog 打开时外部 Escape 绑定的冲突

当 `<dialog showModal()>` 打开时，页面其余部分被标记 `inert`。但 RadixTrie 中仍保留着外部元素的快捷键注册。

源码位置：[_settings_nav.html.erb:43](file:///d:/fz/0601-2/solo-dogfeeding/code/41-maybe/app/views/settings/_settings_nav.html.erb#L43)

```erb
<%= link_to previous_path, ..., data: { controller: "hotkey", hotkey: "Escape" } do %>
  <kbd>esc</kbd>
<% end %>
```

**在 Dialog 打开时按 Escape 的实际执行流：**

1. `keyDownHandler` 在 document 上触发
2. `eventToHotkeyString(e)` → `"Escape"`
3. RadixTrie 查找 `"Escape"` → **找到设置页链接**（其 `data-hotkey="Escape"` 正确注册）
4. 反向遍历 Leaf 的 children，该链接无 `data-hotkey-scope`，焦点不在表单字段 → `i = true`
5. `fireDeterminedAction(settingsNavLink, path)` → `settingsNavLink.click()`
6. `e.preventDefault()` → **阻止了浏览器原生的 Dialog 关闭行为！**

**这意味着：在设置页面上，如果 Dialog 是打开的，按 Escape 不会关闭 Dialog，而是触发页面返回导航。** 外部 `inert` 元素上的 `click()` 在程序层面仍能被调用（`inert` 只阻止用户交互，不阻止程序触发的 `click()`），而 `e.preventDefault()` 则阻止了原生的 Dialog Escape 关闭。

**不过**，该链接有 `pointer-events-none` CSS 类，且通常 Turbo 对 `inert` 元素内的导航链接会有拦截。实际表现取决于浏览器对 `inert` + 程序化 `click()` 的处理细节——不同浏览器可能有差异。

### 3.5 Menu 组件的局部监听 — 不阻止冒泡，全局快捷键仍会触发

源码位置：[menu_controller.js:37-62](file:///d:/fz/0601-2/solo-dogfeeding/code/41-maybe/app/components/DS/menu_controller.js#L37-L62)

Menu 不使用全局 `@github/hotkey`，而是在组件元素上直接监听：
```javascript
this.element.addEventListener("keydown", this.handleKeydown);
// ...
handleKeydown = (event) => {
  if (event.key === "Escape") {
    this.close();
    this.buttonTarget.focus();
  }
};
```

**特点：**
- 作用域严格限定在菜单 DOM 子树内
- `Escape` 同时负责关闭菜单 + 归还焦点到触发按钮
- 不经过 RadixTrie，不与其他 Escape 绑定冲突

---

## 四、命令分发流程：从按键到 Stimulus 动作

### 4.1 fireDeterminedAction — @github/hotkey 的触发机制

```javascript
function fireDeterminedAction(e, t) {
  const n = new CustomEvent("hotkey-fire", { cancelable: true, detail: { path: t } });
  const i = !e.dispatchEvent(n);  // 允许外部通过 hotkey-fire 事件阻止触发
  i || (isFormField(e) ? e.focus() : e.click());
}
```

**分发策略二选一：**
| 目标元素类型 | 触发动作 | 典型场景 |
|---|---|---|
| 表单字段 (`isFormField=true`) | `e.focus()` | 将焦点移到某个输入框 |
| 非表单元素 | `e.click()` | 触发按钮/链接的点击事件 |

**边界条件：**
- `hotkey-fire` 事件是 `cancelable` 的——外部代码可以 `addEventListener("hotkey-fire", e => e.preventDefault())` 来阻止默认的 click/focus 行为
- `e.focus()` 对 `hidden` 元素无效——但 `hidden` 按钮上的 `isFormField` 返回 false，所以实际走 `click()` 路径

### 4.2 Stimulus data-action 桥接

`click()` 事件触发后，由 Stimulus 的 Action 系统接管。关键：`click()` 是在**绑定 hotkey 的元素自身**上触发的，该元素同时有 `data-action` 声明。

**声明式绑定示例：**

1. **开发环境主题切换** — [_htmldoc.html.erb:18](file:///d:/fz/0601-2/solo-dogfeeding/code/41-maybe/app/views/layouts/shared/_htmldoc.html.erb#L18)
   ```erb
   <button hidden data-controller="hotkey" data-hotkey="t t /" data-action="theme#toggle"></button>
   ```
   流程：`t`→`t`→`/` 序列 → `fireDeterminedAction` → `button.click()` → Stimulus 捕获 click → `theme_controller.toggle()`

2. **Dialog ESC 关闭** — [dialog.rb:106](file:///d:/fz/0601-2/solo-dogfeeding/code/41-maybe/app/components/DS/dialog.rb#L106)
   ```ruby
   data[:hotkey] = "esc:DS--dialog#close"
   ```
   **此绑定无效**——`"esc:DS--dialog#close"` 整体被当作键名，无法匹配 `eventToHotkeyString` 输出的 `"Escape"`。实际的 Dialog Escape 关闭走的是原生 `<dialog>` 路径（见 3.3 节）。
   
   如果要让此绑定生效，`data-hotkey` 应改为 `"Escape"`，并添加 `data-action="keydown->DS--dialog#close"` 或通过 Stimulus 的 click → action 路由。但需注意与原生行为的 `preventDefault` 协调。

3. **列表键盘导航** — [_container.html.erb:23-24](file:///d:/fz/0601-2/solo-dogfeeding/code/41-maybe/app/views/accounts/new/_container.html.erb#L23-L24)
   ```erb
   <button hidden data-controller="hotkey"
     data-hotkey="k,K,ArrowUp,ArrowLeft"
     data-action="list-keyboard-navigation#focusPrevious">Previous</button>
   ```
   流程：`k`/`↑`/`←` → `button.click()` → Stimulus 路由到 [list_keyboard_navigation_controller.js](file:///d:/fz/0601-2/solo-dogfeeding/code/41-maybe/app/javascript/controllers/list_keyboard_navigation_controller.js) 的 `focusPrevious()` → 计算索引 → `element.focus()`

4. **设置页 Escape 返回** — [_settings_nav.html.erb:43](file:///d:/fz/0601-2/solo-dogfeeding/code/41-maybe/app/views/settings/_settings_nav.html.erb#L43)
   ```erb
   <%= link_to previous_path, ..., data: { controller: "hotkey", hotkey: "Escape" } do %>
     <kbd>esc</kbd>
   <% end %>
   ```
   流程：`Escape` → `link.click()` → 浏览器导航到 `previous_path`
   
   **注意**：这是代码库中唯一一个**正确使用** `data-hotkey="Escape"` 的绑定（与 `eventToHotkeyString` 的输出一致）。

### 4.3 Chat 输入的特殊处理

源码位置：[chat_controller.js:36-41](file:///d:/fz/0601-2/solo-dogfeeding/code/41-maybe/app/javascript/controllers/chat_controller.js#L36-L41)

```javascript
handleInputKeyDown(e) {
  if (e.key === "Enter" && !e.shiftKey) {
    e.preventDefault();
    this.formTarget.requestSubmit();
  }
}
```

**为什么不走 @github/hotkey？**
- Chat 输入框是表单字段，`isFormField` 返回 true
- 若未配置 `data-hotkey-scope`，@github/hotkey 会直接忽略该按键
- 业务需求是 Enter 发送（而非换行），Shift+Enter 换行——这是输入框内部语义，与导航快捷键体系不同
- 因此直接在输入元素上绑定 `keydown` 监听

**边界条件：** Chat 区域的 `<div>` 声明了 `data-controller="chat hotkey"`（见 [application.html.erb:140](file:///d:/fz/0601-2/solo-dogfeeding/code/41-maybe/app/views/layouts/application.html.erb#L140)），但并未设置 `data-hotkey` 属性——所以 `hotkey` 控制器虽然挂载，`install(this.element)` 时读到空字符串，不会注册任何快捷键。这个 `hotkey` 控制器的挂载可能是预留扩展位或历史遗留。

### 4.4 Turbo Confirm Dialog 覆盖

源码位置：
- [application.js:9-17](file:///d:/fz/0601-2/solo-dogfeeding/code/41-maybe/app/javascript/controllers/application.js#L9-L17)
- [confirm_dialog_controller.js](file:///d:/fz/0601-2/solo-dogfeeding/code/41-maybe/app/javascript/controllers/confirm_dialog_controller.js)
- [custom_confirm.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/41-maybe/app/helpers/custom_confirm.rb)

```javascript
Turbo.config.forms.confirm = (data) => {
  const confirmDialogController = application.getControllerForElementAndIdentifier(
    document.getElementById("confirm-dialog"), "confirm-dialog"
  );
  return confirmDialogController.handleConfirm(data);
};
```

**流程：**
1. 带 `data-turbo-confirm` 的表单提交 → Turbo 调用自定义 confirm
2. 获取全局单例 `#confirm-dialog` 的 Stimulus 控制器
3. `handleConfirm()` 准备数据、调用 `showModal()`、返回 Promise
4. 用户在 Modal 中按 Enter（确认按钮 `autofocus`）或点按钮 → `dialog.returnValue = "confirm"` → Promise resolve(true)
5. 用户按 Escape → **原生 `<dialog>` 行为关闭** → `dialog.returnValue = ""` → `"" !== "confirm"` → Promise resolve(false) → 取消提交

**Confirm Dialog 的 Escape 处理恰好正确**——虽然 `data-hotkey="esc:DS--dialog#close"` 无效，但原生关闭将 `returnValue` 设为空字符串，等效于"取消"。

---

## 五、全局监听、输入焦点限制与命令触发的边界条件总结

### 5.1 keyDownHandler 完整判定链（与 2.4.2 节代码编号一一对应）

```
keydown 事件到达 document → keyDownHandler(e)
  │
  ├─ ① e.defaultPrevented? ──→ 是 → return（不重置序列）
  │
  ├─ ② e.target instanceof Node? ──→ 否 → return（不重置序列）
  │
  ├─ ③ isFormField(e.target)?
  │   ├─ 是 → target.id 存在?
  │   │   ├─ 否 → return（不重置序列）
  │   │   └─ 是 → DOM 中有 [data-hotkey-scope="target.id"]?
  │   │       ├─ 否 → return（不重置序列）
  │   │       └─ 是 → 继续匹配（只匹配带 scope 的绑定）
  │   └─ 否 → 继续匹配（只匹配不带 scope 的全局绑定）
  │
  ├─ t = c.get(eventToHotkeyString(e))    ← 从当前位置 c 的 children 中查找
  │
  ├─ ④ t 存在（truthy）?
  │   ├─ 是 →
  │   │   ├─ a.registerKeypress(e)        ← _path 推入按键 + 重启 1500ms 计时
  │   │   ├─ c = t                        ← 书签移动到子节点
  │   │   │
  │   │   ├─ ⑤ t instanceof Leaf?
  │   │   │   ├─ 是（序列完成）→ 反向遍历 children 找 scope 匹配
  │   │   │   │   ├─ 有匹配 → fireDeterminedAction + e.preventDefault()
  │   │   │   │   └─ 无匹配 → 什么都不做
  │   │   │   └─ a.reset()                ← 无条件重置！
  │   │   │         c = o, _path = []      （无论是否触发动作）
  │   │   │
  │   │   └─ ⑥ else（t 是 RadixTrie，中间节点）
  │   │       └─ 什么都不做！
  │   │         c 停在中间节点，_path 保留
  │   │         ⚠ 这就是"序列进行中，等待下一步"
  │   │
  │   └─ ⑦ else（t 为 undefined/falsy，完全不匹配）
  │       └─ a.reset()                    ← 立即重置！
  │             c = o, _path = []
  │             ⚠ 不是"进入中间态"，是直接清空
  │
  └─ 最终 preventDefault? → 仅在 ⑤ 中 scope 匹配到元素时为 true
          其余所有情况：false，原生行为保留
```

**核心区分 — 立即重置 vs 等待下一步：**

| 条件 | 触发分支 | 结果 | 序列状态 |
|---|---|---|---|
| `c.get(按键)` 返回 `undefined` | ⑦ | `a.reset()` 立即调用 | 序列清空，回到起点 |
| `c.get(按键)` 返回 `RadixTrie` 实例 | ④→⑥ | 什么都不做 | 序列继续，等待下一步 |
| `c.get(按键)` 返回 `Leaf` 实例 | ④→⑤ | 动作后 `a.reset()` | 序列完成，清空 |
| `isFormField` 提前 return | ③ | 不调用 reset | 序列冻结，计时继续 |

**判定顺序明确：** 先走 ④ 的 `if(t)`（存在性），再在其内部走 ⑤/⑥ 的类型判断（Leaf vs RadixTrie）。所以"不匹配"和"匹配到中间节点"是两条互斥分支——不可能同时发生。

### 5.2 `<dialog showModal()>` 打开时的 Escape 优先级

```
用户按 Escape（dialog 打开状态）
  │
  ├─ [1] keydown 事件 → @github/hotkey keyDownHandler
  │     ├─ RadixTrie 中有 "Escape" 绑定?（如 settings_nav）
  │     │   ├─ 是 → fireDeterminedAction → click() → e.preventDefault()
  │     │   │        ⚠ 原生 dialog 关闭被阻止！
  │     │   └─ 否 → 不 preventDefault
  │     └─ RadixTrie 中 "esc:DS--dialog#close" 不匹配 "Escape"
  │          → dialog 的 hotkey 绑定永远不触发
  │
  └─ [2] 浏览器 UA 层 → cancel 事件
        ├─ 若 keydown 被 preventDefault → cancel 不触发（dialog 不关闭）
        └─ 若 keydown 未被 preventDefault → cancel 触发 → dialog.close()
             ⚠ 绕过 DS--dialog#close()，reloadOnClose 不执行
```

### 5.3 各快捷键场景的边界条件速查表

| # | 场景 | 焦点位置 | isFormField | 快捷键是否生效 | 原因 |
|---|---|---|---|---|---|
| 1 | 页面空白处按 `j/k` | 普通元素 | false | ✅ 生效 | 匹配全局导航绑定 |
| 2 | 输入框内按 `j/k` | input/textarea | true | ❌ 禁用 | 无 `data-hotkey-scope`，直接 return |
| 3 | 输入框内按序列首键 | input | true | ❌ 无效且不重置 | isFormField 过滤 return，序列状态冻结，计时继续 |
| 4 | Dialog 内按 `j/k` | dialog 内普通元素 | false | ✅ 生效（如有绑定） | dialog 内元素不在 inert 区域 |
| 5 | Dialog 外按 `j/k` | inert 元素 | false | ⚠ 匹配但 click 可能无效 | RadixTrie 找到绑定但目标元素 inert |
| 6 | 设置页按 Escape | 普通元素 | false | ✅ 导航返回 | `data-hotkey="Escape"` 正确匹配 |
| 7 | 设置页 Dialog 开时按 Escape | dialog 内元素 | 视具体元素 | ⚠ 可能导航返回而非关 dialog | 外部 Escape 绑定在 RadixTrie 中仍存在，preventDefault 阻止原生关闭 |
| 8 | 非 Dialog 页面按 Escape | 普通元素 | false | ❌ 无响应 | 无匹配的 Escape 绑定，序列直接 reset |
| 9 | Dialog 内按 Escape | dialog 内普通元素 | false | ✅ 关闭 dialog | 原生 `<dialog>` UA 层 cancel 事件（非 hotkey） |
| 10 | Dialog 内输入框按 Escape | dialog 内 input | true | ⚠ 原生行为仍关 dialog | isFormField 阻止 hotkey 但不阻止 UA 层 cancel |
| 11 | Confirm Dialog 按 Escape | dialog 内元素 | 视具体元素 | ✅ 取消操作 | 原生关闭 → returnValue="" → 非确认 |
| 12 | Menu 内按 Escape | 菜单内元素 | 视具体元素 | ✅ 关菜单 + 可能连带全局 | Menu handleKeydown 先关菜单，冒泡后全局继续匹配 |
| 13 | Menu + Dialog 嵌套时按 Escape | 菜单内元素 | 视具体元素 | ⚠ 关菜单 + 可能阻止 dialog 关 | 全局 Escape 匹配 → preventDefault → 阻止原生 dialog 关闭 |
| 14 | 序列中间按不匹配的键 | 普通元素 | false | ✅ 立即重置 | `t` 为 falsy → `a.reset()`，序列清空 |
| 15 | 序列中间切换到输入框 | input | true | ⚠ 序列冻结计时中 | isFormField return 不重置，1500ms 后超时重置 |

---

## 六、关键文件索引

| 文件 | 职责 |
|---|---|
| [@github--hotkey.js](file:///d:/fz/0601-2/solo-dogfeeding/code/41-maybe/vendor/javascript/@github--hotkey.js) | 全局 keydown 监听、RadixTrie、expandHotkeyToEdges 解析、作用域匹配 |
| [importmap.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/41-maybe/config/importmap.rb#L10) | 声明 @github/hotkey v3.1.1 |
| [hotkey_controller.js](file:///d:/fz/0601-2/solo-dogfeeding/code/41-maybe/app/javascript/controllers/hotkey_controller.js) | Stimulus 包装，管理 install/uninstall 生命周期 |
| [dialog.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/41-maybe/app/components/DS/dialog.rb#L98-L110) | Dialog 组件 merged_opts，注入 hotkey + `esc:DS--dialog#close` |
| [dialog_controller.js](file:///d:/fz/0601-2/solo-dogfeeding/code/41-maybe/app/components/DS/dialog_controller.js) | showModal()、clickOutside、close()（含 reloadOnClose） |
| [dialog.html.erb](file:///d:/fz/0601-2/solo-dogfeeding/code/41-maybe/app/components/DS/dialog.html.erb) | Dialog 模板，使用原生 `<dialog>` |
| [menu_controller.js](file:///d:/fz/0601-2/solo-dogfeeding/code/41-maybe/app/components/DS/menu_controller.js#L57-L62) | 菜单局部 ESC 监听、焦点管理 |
| [list_keyboard_navigation_controller.js](file:///d:/fz/0601-2/solo-dogfeeding/code/41-maybe/app/javascript/controllers/list_keyboard_navigation_controller.js) | j/k/方向键导航列表项 |
| [chat_controller.js](file:///d:/fz/0601-2/solo-dogfeeding/code/41-maybe/app/javascript/controllers/chat_controller.js#L36-L41) | Chat 输入框 Enter/Shift+Enter 语义 |
| [confirm_dialog_controller.js](file:///d:/fz/0601-2/solo-dogfeeding/code/41-maybe/app/javascript/controllers/confirm_dialog_controller.js) | Turbo 自定义 confirm 弹窗 |
| [application.js](file:///d:/fz/0601-2/solo-dogfeeding/code/41-maybe/app/javascript/controllers/application.js#L9-L17) | Stimulus 启动 + Turbo.confirm 覆盖 |
| [_htmldoc.html.erb](file:///d:/fz/0601-2/solo-dogfeeding/code/41-maybe/app/views/layouts/shared/_htmldoc.html.erb#L18) | 全局快捷键（开发环境 `t t /` 切换主题） |
| [_container.html.erb](file:///d:/fz/0601-2/solo-dogfeeding/code/41-maybe/app/views/accounts/new/_container.html.erb#L23-L24) | 账户新建流程中的列表键盘导航 |
| [_settings_nav.html.erb](file:///d:/fz/0601-2/solo-dogfeeding/code/41-maybe/app/views/settings/_settings_nav.html.erb#L43) | 设置页 Escape 返回（唯一正确的 `data-hotkey="Escape"` 绑定） |
| [application.html.erb](file:///d:/fz/0601-2/solo-dogfeeding/code/41-maybe/app/views/layouts/application.html.erb#L140) | Chat 区域挂载 `chat hotkey` 双控制器 |
| [custom_confirm.rb](file:///d:/fz/0601-2/solo-dogfeeding/code/41-maybe/app/helpers/custom_confirm.rb) | confirm 弹窗数据构造 |
