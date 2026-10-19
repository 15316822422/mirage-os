# 忘忧 OS 插件规范 · 普通（沙箱）插件

> 适用版本：忘忧 OS **3.0.0+** · 插件 API **2**（自 2.2.0 起）
> 高级（全权限）插件见 [ADVANCED-PLUGIN-SPEC.md](ADVANCED-PLUGIN-SPEC.md)

> 兼容说明：3.0 完成品牌更名，插件清单标记 `mirage-plugin`、`Mirage` API 与 `mirage.*` 存储键全部沿用旧名，2.x 插件可直接继续使用。

---

## 0. 两种插件模式

忘忧 OS 的插件分两条完全不同的运行路径。**默认、也是绝大多数情况下应该使用的，是沙箱模式。**

| | 普通插件（本文档） | 高级插件 |
|---|---|---|
| 清单声明 | `"mode": "sandbox"` 或不写 | `"mode": "advanced"` |
| 运行位置 | 独立 sandbox iframe | 主页面上下文 |
| 同源 | ❌ opaque origin（`origin === "null"`） | ✅ 与系统同源 |
| 宿主 DOM | 读不到（SecurityError） | 完全可读写 |
| localStorage / IndexedDB / Cookie | 读不到 | 完全可读写 |
| 内核钩子 | 无此能力 | `ctx.kernel.override()` |
| UI 形式 | 返回 HTML 字符串，宿主渲染 | 直接操作 DOM |
| 授权 | 不需要，安装即用 | 必须逐项授权 |
| 心跳 / 看门狗 | 参与 | 豁免 |
| 适合 | 工具、面板、小组件、演示 | 改内核、深度定制、自己写的系统级扩展 |

> 一句话：**普通插件只能「请求」系统做事，高级插件可以「替」系统做事。**

---

## 1. 运行模型

```
┌──────────────────────── 宿主页面（忘忧 OS）────────────────────────┐
│                                                                    │
│   pluginState（记录 / 授权 / 错误）                                  │
│   窗口管理 · 桌面图标 · 右键菜单 · 设置页 · 任务栏                     │
│        ▲                                      │                    │
│        │ postMessage RPC                      │ 注册元数据 + HTML     │
│        ▼                                      ▼                    │
│   ┌──────────── 插件主实例 iframe ────────────┐                      │
│   │ sandbox="allow-scripts allow-forms ..."   │                      │
│   │ 无 allow-same-origin → opaque origin      │                      │
│   │ 桥接脚本定义 window.Mirage                  │                      │
│   │ 插件脚本在这里长期驻留，保存自己的状态         │                      │
│   └───────────────────────────────────────────┘                      │
│        │ render() 返回 HTML 字符串                                    │
│        ▼                                                            │
│   ┌──── 视图 iframe（窗口 / 设置块 / 任务栏组件）────┐                  │
│   │ 同样是 sandbox，独立 origin                      │                  │
│   │ 只负责显示；点击通过 data-mirage-action 回传        │                  │
│   └────────────────────────────────────────────────┘                  │
└────────────────────────────────────────────────────────────────────┘
```

### 1.1 三种 iframe 角色

| 角色 | 数量 | 尺寸 | 作用 |
|---|---|---|---|
| 主实例 | 每个插件 1 个 | 2×2 px，移出视口 | 执行插件脚本、保存状态、响应宿主请求 |
| 窗口视图 | 每个打开的插件窗口 1 个 | 铺满窗口 | 显示 `render()` 返回的 HTML |
| 设置 / 任务栏视图 | 每个区块 1 个 | 固定或自适应高度 | 显示设置区块、任务栏小组件 |

所有 iframe 共用同一份 `sandbox` 属性：

```
allow-scripts allow-forms allow-modals allow-popups allow-downloads
```

**故意不包含 `allow-same-origin`**，所以：

- `parent.document` → `SecurityError`
- `localStorage` → `SecurityError`
- `document.cookie` → 空
- `location.origin` → `"null"`
- 无法读取宿主的任何 JS 变量、函数、原型链

### 1.2 为什么 UI 要返回字符串

跨窗口 `postMessage` 使用结构化克隆，**函数无法传递**。所以插件不能像普通库那样把 `render(root)` 回调交给宿主调用。

本系统的做法是反过来：宿主向插件发一个 `render` 请求，插件在自己的 iframe 里执行 `render()`，把结果（HTML 字符串）回传。这样：

- 插件代码与状态始终只存在于自己的 iframe 里；
- 宿主永远只拿到字符串，不会执行插件函数；
- 插件仍然可以写 `render() { return '...' }` 这种直观的代码。

---

## 2. 文件格式

### 2.1 HTML 插件（推荐）

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
<meta charset="utf-8">
<meta name="mirage-plugin" content='{
  "id": "com.example.hello",
  "name": "示例插件",
  "version": "1.0.0",
  "author": "你",
  "description": "一句话说明",
  "mode": "sandbox",
  "priority": 10
}'>
<style data-mirage-plugin>
  /* 只会注入插件自己的视图，不会影响系统界面 */
</style>
</head>
<body>
<script data-mirage-plugin>
  /* 插件代码 */
</script>
</body>
</html>
```

### 2.2 纯 JS 插件

文件扩展名 `.js`，首行写清单注释：

```js
// @mirage-plugin {"id":"com.example.js","name":"JS 插件","version":"1.0.0","mode":"sandbox"}
Mirage.app({ /* ... */ });
```

### 2.3 清单字段

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `id` | string | 建议 | 唯一标识，`[a-z0-9._-]`，最长 80。缺省时按源码哈希生成 `plug_xxxxxxxxxx` |
| `name` | string | 建议 | 显示名称 |
| `version` | string | 否 | 语义化版本，默认 `1.0.0` |
| `author` | string | 否 | 作者 |
| `description` | string | 否 | 插件中心里的一句话描述 |
| `mode` | string | 否 | `"sandbox"`（默认）或 `"advanced"` |
| `permissions` | string[] | 高级必填 | 高级插件权限，见高级文档；沙箱插件写了也只作展示 |
| `crashPolicy` | string | 否 | 仅高级插件：`"disable"`（默认）/ `"ignore"` |
| `priority` | number | 否 | 启动顺序，越小越先启动，默认 0 |

### 2.4 大小与解析

- 单个插件源码上限 **2 MB**，超过直接拒绝安装。
- 带 `<meta name="mirage-plugin">` 或 `Mirage.register(` 的文件才会被识别为插件。
- `<style>` 会被抽取并注入到该插件的每个视图；`<body>` 内的静态 HTML 会注入主实例（一般留空即可）。
- `<script>` 会被抽取，在桥接脚本之后按顺序执行。

---

## 3. 生命周期

```
安装 ──▶ 写入 IndexedDB ──▶ 启用
                            │
                            ▼
                 创建主实例 iframe（srcdoc）
                            │
                     桥接脚本执行，定义 Mirage
                            │
                     插件脚本执行（可异步）
                            │
                     load 事件 → 自动发 ready
                            │
                 宿主收到 ready → 标记 running
                            │
        ┌───────────────────┼────────────────────┐
        ▼                   ▼                    ▼
   打开窗口            注册菜单/图标         设置区块渲染
        │
        ▼
   禁用 / 重载 / 卸载 ──▶ 销毁 iframe（定时器、监听器随文档一起消失）
```

| 阶段 | 宿主行为 | 插件可做的事 |
|---|---|---|
| 加载 | 创建 iframe，写入 `srcdoc` | 执行顶层代码、注册 UI、读存储 |
| ready | 6 秒内未收到 ready 视为启动失败 | 无需手动调用，桥接层自动发 |
| 运行 | 转发事件、响应 RPC | 打开窗口、更新视图、读写存储 |
| 停止 | 移除 iframe、清理注册项 | 无 `stop` 回调（文档销毁即结束） |

> 因为没有 `stop` 回调，**不要依赖「停止时保存」**。状态请在有变化时立即 `Mirage.storage.set()`。

---

## 4. 桥接协议（了解即可）

插件脚本不需要手写协议，桥接层已经封装成 `Mirage.*`。以下内容用于排错和理解限制。

### 4.1 消息格式

```js
{
  __mirage: 1,
  token: "sbx_xxx",        // 每个插件实例独有，宿主据此识别来源
  kind: "call" | "reply" | "invoke" | "event" | "action" | "resize",
  seq: 12,                 // RPC 序号
  method: "toast",
  params: { ... },
  ok: true,
  value: null,
  error: ""
}
```

### 4.2 超时

| 方向 | 超时 | 结果 |
|---|---|---|
| 插件 → 宿主（`Mirage.*`） | 15 秒 | Promise reject `调用超时：xxx` |
| 宿主 → 插件（`render` 等） | 8 秒 | 窗口显示「插件渲染失败」 |

### 4.3 安全校验

宿主对每条消息做三件事：

1. `data.__mirage === 1`；
2. `token` 能匹配到正在运行的沙箱插件；
3. `method` 在白名单内（见第 6 节）。

任何一条不满足就丢弃。**插件无法伪造别的插件的 token，也无法调用未实现的方法。**

---

## 5. 快速开始

最小可用插件：

```html
<meta name="mirage-plugin" content='{"id":"com.example.mini","name":"最小插件","mode":"sandbox"}'>
<script data-mirage-plugin>
Mirage.app({
  id: 'mini',
  name: '最小插件',
  icon: 'spark',
  desktop: true,
  w: 640, h: 460,
  render() {
    return '<div style="padding:24px">Hello, 忘忧。</div>';
  }
});
</script>
```

装进系统后：桌面出现图标 → 双击打开窗口 → 窗口里就是那段 HTML。

---

## 6. API 全表

### 6.1 元信息

| 成员 | 类型 | 说明 |
|---|---|---|
| `Mirage.apiVersion` | number | 当前为 `2` |
| `Mirage.mode` | string | 固定 `"sandbox"` |
| `Mirage.pluginId` | string | 当前插件 id |

### 6.2 应用窗口

```js
const app = Mirage.app({
  id: 'hello',            // 应用短名，最终 key 为 plugin.<插件id>.hello
  name: '插件示例',        // 窗口标题 / 桌面图标名
  sub: '副标题',           // 可选
  icon: 'spark',          // 内置图标名，或一段 <svg> 字符串
  g1: '#6c8cff', g2: '#a78bfa',  // 可选，图标渐变
  w: 720, h: 520,         // 默认窗口尺寸
  desktop: true,          // 是否在桌面放图标（默认 false）
  hidden: false,          // 是否隐藏于开始菜单（默认 false）
  render() { return '<div>...</div>'; },       // 可 async
  onAction(action, value, form) { /* ... */ }  // 可选
});

app.key      // 窗口 key
app.open()   // Promise<void>
app.close()  // Promise<void>
```

- `render()` 的返回值会写进窗口 iframe 的 `<body>`。
- 每次 `Mirage.setView()` 都会重新执行 `render()`（或直接使用传入的 HTML）。
- 窗口 iframe 里可以写 `<script>`，会正常执行（仍然在沙箱里）。

### 6.3 桌面图标

```js
Mirage.icon({
  id: 'quick',
  name: '快捷入口',
  icon: 'spark',
  app: app.key,     // 点击打开哪个应用；不填则发 icon 事件
  g1: '#34d399', g2: '#0ea5e9'
});
```

不填 `app` 时，点击会触发：

```js
Mirage.on('icon', (detail) => { /* detail.id === 'quick' */ });
```

### 6.4 右键菜单

```js
Mirage.menuItem({ id: 'open', label: '打开插件示例', icon: 'spark', order: 10 });
Mirage.on('menu', (detail) => { if (detail.id === 'open') app.open(); });
```

菜单项出现在桌面空白处与任务栏的右键菜单里，点击时向插件发 `menu` 事件。

### 6.5 设置区块

```js
Mirage.settingsSection({
  id: 'about',
  title: '插件示例',
  desc: '这一段由插件渲染。',
  height: 72,                      // 初始高度，渲染后按内容自动调整
  render() { return '<div>...</div>'; }
});
```

- 宿主在设置页创建一个独立沙箱 iframe 渲染这段 HTML。
- 内容高度变化时，桥接层会通过 `ResizeObserver` 上报，宿主自动调整 iframe 高度（40–900 px）。
- 区块里同样可以放 `<script>`。

### 6.6 任务栏组件

```js
Mirage.widget({
  id: 'clock',
  width: 128,
  height: 34,
  title: '插件时钟',
  render() { return '<div>...</div>'; }
});
```

显示在任务栏右侧（`#tb-plugin-widgets`）。移动端（≤720px）会自动隐藏。

### 6.7 通知与对话框

```js
Mirage.toast('保存成功', 'success');   // info | success | warn | error
Mirage.notify('普通提示');

const ok = await Mirage.confirm({
  title: '确认操作',
  message: '这段文字支持换行',
  confirmText: '继续',
  cancelText: '取消',
  danger: true
});                                    // → boolean

const name = await Mirage.prompt({
  title: '重命名',
  message: '请输入新名称',
  value: '默认值',
  placeholder: '名称'
});                                    // → string | null
```

### 6.8 命名空间存储

```js
await Mirage.storage.set('clicks', 3);
const n = await Mirage.storage.get('clicks', 0);
await Mirage.storage.remove('clicks');
const keys = await Mirage.storage.keys();
```

- 实际写入宿主 `localStorage` 的 `mirage.plugin.<插件id>.<key>`。
- 值会自动 `JSON.stringify` / `parse`，可存对象与数组。
- 只读写自己的命名空间，看不到系统设置、笔记、其他插件的数据。
- 卸载插件**不会**自动删除这些数据；需要清理时调用 `remove` 或「清空本机数据」。

### 6.9 打开应用

```js
await Mirage.openApp('settings');
await Mirage.openApp(app.key);
```

白名单：

```
files · skin · plugins · notes · settings · about · welcome
plugin.<自己的插件id>.*
```

其余 key 会被拒绝，Promise reject：

```
沙箱插件无权打开该应用：app.secret
```

### 6.10 视图控制

```js
await Mirage.setView(app.key, '<div>新内容</div>');  // 重新写入窗口 body
await Mirage.setTitle(app.key, '新标题');            // 改窗口标题
await Mirage.closeView(app.key);                    // 关闭窗口
```

`setView` 会重建窗口 iframe（等价于刷新视图），因此：

- 窗口内 `<script>` 会重新执行；
- 之前窗口内的局部变量会丢失，状态请存在插件主实例或 `Mirage.storage` 里。

### 6.11 事件

```js
const off = Mirage.on('theme-changed', (detail) => { /* ... */ });
off();  // 取消监听
```

| 事件 | 触发时机 | detail |
|---|---|---|
| `action` | 任意视图里点击了 `data-mirage-action` | `{ key, action, value, form }` |
| `menu` | 点击了插件注册的右键菜单项 | `{ id }` |
| `icon` | 点击了没有绑定应用的桌面图标 | `{ id }` |
| `app-opened` | 任意应用窗口打开 | `{ key }` |
| `app-closed` | 任意应用窗口关闭 | `{ key }` |
| `settings-changed` | 系统设置变更 | `{ patch }` |
| `theme-changed` | 主题 / 皮肤变更 | `{}` |
| `skin-changed` | 皮肤切换 | `{}` |
| `stop` | 插件即将被停止 | `{}` |

> `detail` 会经过结构化克隆，宿主对象（如窗口实例）无法传递，只保留可序列化字段。

### 6.12 日志

```js
Mirage.log('状态', 1, { a: 2 });
```

输出到宿主控制台，前缀 `[plugin:<id>]`。

---

## 7. 视图与交互

### 7.1 用 `data-mirage-action` 绑定交互

视图 iframe 的桥接脚本会拦截所有带该属性的元素：

```html
<button data-mirage-action="save" data-mirage-value="42">保存</button>

<form data-mirage-action="submit">
  <input name="title">
  <select name="kind"><option value="a">A</option></select>
  <input type="checkbox" name="done">
  <button type="submit">提交</button>
</form>
```

点击 / 提交时，宿主收到：

```js
{
  key: 'plugin.com.example.x.hello',
  action: 'save',
  value: '42',
  form: { title: '...', kind: 'a', done: true }
}
```

`form` 会收集同一 `<form>` 内所有带 `name` 的 `input / select / textarea`：

- `checkbox` → boolean
- `radio` → 选中项的值
- 其余 → 字符串

然后分两路投递：

1. 如果 `key` 对应某个应用，调用该应用的 `onAction(action, value, form)`；
2. 同时以 `action` 事件广播给 `Mirage.on('action')`。

### 7.2 主题变量

视图文档会自动注入宿主当前的全部 CSS 自定义属性（`--ui-*`、`--glass-*` 等）。所以插件可以直接写：

```css
.card{background:var(--ui-elev);color:var(--ui-text);border:1px solid var(--ui-border-soft)}
.accent{color:var(--ui-accent)}
```

切换主题 / 皮肤后，宿主会重建视图或发送 `theme-changed`，颜色自动跟随。

### 7.3 背景

- 窗口视图：透明背景，直接透出窗口本体。
- 设置区块 / 任务栏组件：由宿主指定纯色背景（`--ui-surface-2` / `--ui-bg`），避免浏览器把无同源 iframe 画成白底。

插件自己的 CSS 可以覆盖，但建议保持透明或使用主题变量。

### 7.4 尺寸

- 窗口视图：100% × 100%，内部自行滚动。
- 设置区块：初始 `height`，之后按内容自动调整。
- 任务栏组件：`width × height` 固定，超出部分裁切。

---

## 8. 能力边界

### 8.1 能做

- 注册应用窗口、桌面图标、右键菜单、设置区块、任务栏组件
- 显示通知、确认框、输入框
- 读写自己的命名空间存储
- 打开白名单内的系统应用
- 更新自己的窗口内容与标题
- 监听系统事件（主题、设置、窗口开关）
- 在视图里执行任意前端代码（仍在该视图的沙箱内）

### 8.2 不能做

| 想做的事 | 结果 |
|---|---|
| `parent.document` | `SecurityError` |
| `localStorage` / `indexedDB` / `cookie` | `SecurityError` / 空 |
| 调用 `Mirage.kernel.*` | 不存在该 API |
| 打开任意应用 | 白名单外被拒绝 |
| 读取其他插件的数据 | 命名空间隔离 |
| 修改系统设置 | 无 API |
| 跳出 iframe 导航顶层页面 | 被 sandbox 拦截 |
| 向宿主传函数 | 结构化克隆丢弃 |

### 8.3 网络

插件在沙箱里可以 `fetch()`，但因为是 opaque origin，请求属于跨域，受浏览器 CORS 限制。需要读取同源文件时，请让用户通过系统导入，而不是插件自己抓。

---

## 9. 崩溃与恢复

### 9.1 错误上报

| 来源 | 方式 |
|---|---|
| 顶层脚本抛错 | 桥接层 `window.onerror` |
| Promise 未处理拒绝 | `unhandledrejection` |
| `onAction` 内抛错 | 桥接层 try/catch |
| `render()` 抛错 | 宿主收到 reject，窗口显示「插件渲染失败」 |
| 6 秒内未 ready | 宿主判定启动失败 |

### 9.2 处理策略

| 情况 | 处理 |
|---|---|
| 单次运行错误 | 记录到插件中心日志 + toast 提示 |
| 15 秒内累计 3 次错误 | 自动禁用该插件并重载页面 |
| 启动超时 / 启动即崩溃 | 禁用 + 重载 |
| 主线程阻塞 > 12 秒 | 判定卡死，禁用 + 重载 |
| 上次运行标记残留 | 下次启动时禁用该插件（疑似卡死） |

禁用只影响出问题的插件，其他插件照常加载。用户数据（应用、设置、皮肤、插件存储）不会被清空。

### 9.3 安全模式

- 启动参数 `?safe=1`；
- 插件中心「进入安全模式」；
- 连续两次启动失败自动进入。

安全模式下不加载任何插件，插件中心仍可打开，可退出或逐个禁用可疑插件。

---

## 10. 自动发现

打开页面时会依次尝试：

1. `<link rel="mirage-plugin" href="...">`
2. `plugins.json` / `mirage-plugins.json`
3. 同目录 `plugin.html` / `plugin.js`

`plugins.json` 格式：

```json
{
  "version": 1,
  "plugins": [
    { "src": "plugins/hello.html", "enabled": true },
    { "src": "plugins/clock.html", "enabled": false }
  ]
}
```

规则：

- 按 `priority` 升序启动；
- 已存在的同 id 记录不会被覆盖，只更新源码与元信息；
- 自动发现的记录 `origin: "auto"`；
- `file://` 下浏览器禁止读取同目录文件，自动发现会跳过并提示，需要手动拖入或粘贴。

---

## 11. 安装、持久化与导出

### 11.1 安装方式

| 方式 | 入口 |
|---|---|
| 拖放文件 | 把 `.html` / `.js` 拖进桌面 |
| 文件选择 | 插件中心 → 选择文件 |
| 粘贴源码 | 插件中心 → 编辑器 → 安装 |
| 自动发现 | `plugins.json` 或同目录 `plugin.html` |
| API | `Mirage.plugins.install(text, filename)` |

### 11.2 持久化

- 记录写入 IndexedDB `mirage-os` 的 `plugins` store；
- 不可用时回退 `localStorage['mirage.plugins.v1']`；
- 启用状态、崩溃次数、最近错误、模式、权限都会保存。

### 11.3 导出

「导出整个系统」会把**已启用**的插件源码、模式、权限一起写进单文件 HTML。导出文件在 `file://` 下打开时：

- 插件以 `origin: "embedded"` 载入；
- 沙箱插件正常运行；
- 高级插件因为导出环境没有授权，会被跳过（这是有意的安全设计）。

---

## 12. 调试

1. 打开插件中心，底部「崩溃与恢复」会列出最近的错误（含时间与消息）。
2. 浏览器 DevTools → Console，插件日志前缀 `[plugin:<id>]`。
3. DevTools → Elements → 选中 `iframe.mirage-sandbox`，在 Console 顶部的执行上下文里切换到该 frame，即可直接调试插件代码。
4. 修改代码后，在插件中心点「重载」即可重新执行，无需刷新整个页面。

---

## 13. 兼容性

| 项目 | 值 |
|---|---|
| 插件 API | 2 |
| 最低系统版本 | 3.0.0 |
| 浏览器 | 支持 `iframe[sandbox]`、`srcdoc`、`postMessage`、`ResizeObserver` 的现代浏览器 |
| 移动端 | 支持；任务栏组件在 ≤720px 隐藏 |

API 1（2.1.0）的全权插件不会被静默降级：它们没有 `mode` 字段，会被当作沙箱插件载入，若代码里用了 `ctx.*` 会报错。迁移方式见高级文档最后一节。

---

## 14. 完整示例

见同目录 `plugins/hello.html`：应用窗口 + 计数存储 + 右键菜单 + 设置区块 + 主题跟随。

```js
const app = Mirage.app({
  id: 'hello',
  name: '插件示例',
  icon: 'spark',
  desktop: true,
  w: 720, h: 520,
  async render() {
    const n = await Mirage.storage.get('clicks', 0);
    return '<div style="padding:24px">点击次数：' + n +
      '<button data-mirage-action="inc">+1</button></div>';
  },
  async onAction(action) {
    if (action !== 'inc') return;
    const n = (await Mirage.storage.get('clicks', 0)) + 1;
    await Mirage.storage.set('clicks', n);
    await Mirage.setView(app.key, await app.render());
  }
});
```

---

## 15. 常见问题

**Q：为什么我的 `render` 里 `this` 不对？**
A：`render` 是以 `app.render()` 形式调用的，`this` 指向应用定义对象。建议用闭包变量而不是 `this`。

**Q：为什么窗口里的按钮点了没反应？**
A：按钮必须带 `data-mirage-action="xxx"`，且对应应用实现了 `onAction`（或你监听了 `Mirage.on('action')`）。

**Q：为什么设置区块高度不对？**
A：桥接层按 `document.documentElement.scrollHeight` 上报。如果内容绝对定位或高度为 0，需要给容器一个实际高度。

**Q：能不能直接 `import` 外部脚本？**
A：可以，但脚本必须能从网络或同源地址加载；沙箱 iframe 仍受 CORS 限制。单文件插件建议内联。

**Q：卸载插件后数据还在吗？**
A：在。存储按插件 id 命名空间保存，卸载不会删除。需要清理请自行 `remove` 或使用系统「清空本机数据」。
