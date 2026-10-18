# 幻境 OS 插件规范 · 高级（全权限）插件

> 适用版本：幻境 OS **2.2.0+** · 插件 API **2**
> 普通（沙箱）插件见 [PLUGIN-SPEC.md](PLUGIN-SPEC.md)

---

## 0. 这是什么

高级插件**不运行在沙箱里**。它的代码由宿主用 `new Function` 直接在主页面上下文执行，因此：

- 能拿到真正的 `window` / `document` / `localStorage`；
- 能拦截并改写内核函数（`openApp`、`toast`、`applySettings` …）；
- 能修改任意界面、读取任意数据、管理其他插件；
- 不受心跳与启动看门狗约束。

代价是**没有任何隔离**。所以它需要：

1. 用户在设置或插件中心**打开「高级模式」总开关**；
2. 用户对**这一份源码**逐项授权权限；
3. 源码一旦修改，授权自动失效，需要重新确认。

> 高级插件只应该用来运行你自己写的、或你完全信任的代码。

---

## 1. 与普通插件的差异

| 维度 | 普通（沙箱） | 高级（全权限） |
|---|---|---|
| 清单 `mode` | `"sandbox"` 或不写 | `"advanced"` |
| 执行方式 | iframe `srcdoc` + 桥接 RPC | `new Function('Mirage', code)` |
| 执行上下文 | opaque origin | 主页面 |
| `document` | 只能操作自己的视图 | 全页面 |
| `localStorage` | 命名空间代理，异步 | 原生，同步 |
| 内核钩子 | ❌ | ✅ 8 个函数 |
| UI | `render()` 返回 HTML 字符串 | `render(root)` 直接操作 DOM |
| 定时器 | 原生，随 iframe 销毁 | `ctx.interval` / `ctx.timeout`，停用时自动清理 |
| 授权 | 无 | 逐项授权，绑定源码指纹 |
| 心跳 / 看门狗 | 参与 | 豁免 |
| 崩溃默认策略 | 禁用 + 重载 | 禁用 + 重载，可改为只记录 |
| 出现位置 | 桌面图标 / 开始菜单 | 同上（桌面布局不变，没有额外状态栏） |

---

## 2. 运行模型

```
┌────────────────────── 宿主页面（幻境 OS）──────────────────────┐
│                                                                │
│  Mirage.register(def)                                          │
│        │                                                       │
│        ▼                                                       │
│  new Function('Mirage', 插件源码)  ← 直接在主上下文执行            │
│        │                                                       │
│        ├── ctx.app / ctx.ui.*        → 注册窗口与界面             │
│        ├── ctx.kernel.override(...)  → 进入内核钩子链             │
│        ├── ctx.advanced.system.*     → 管理插件 / 系统            │
│        └── document / window         → 任意 DOM 操作              │
│                                                                │
│  错误 → window.onerror → pluginError → 日志 / 禁用               │
│  心跳 → 不写入运行时标记（豁免）                                   │
└────────────────────────────────────────────────────────────────┘
```

### 2.1 执行细节

- 源码中的每个 `<script>` 会被抽出，依次执行：
  ```js
  const fn = new Function('Mirage', code + '\n//# sourceURL=mirage-plugin://<id>');
  fn(window.Mirage);
  ```
- `sourceURL` 让 DevTools 里能按插件 id 定位堆栈；
- `<style>` 会作为全局样式注入（`data-mirage-plugin="<id>"`），停用时移除；
- `<body>` 内的静态 HTML 会挂到 `document.body` 末尾，同样在停用时移除。

### 2.2 启动前置条件

`startAdvancedPlugin()` 会依次检查：

```
插件存在且已启用
  → 不在安全模式
  → 高级模式总开关已打开
  → 当前源码已有有效授权（指纹一致且权限覆盖）
  → 才执行代码
```

任一条件不满足就**静默跳过**，不写崩溃记录，只在插件中心显示「待授权」。

---

## 3. 清单与权限声明

```html
<meta name="mirage-plugin" content='{
  "id": "com.example.advanced",
  "name": "高级示例",
  "version": "1.0.0",
  "author": "你",
  "description": "拦截 openApp 并记录调用历史",
  "mode": "advanced",
  "permissions": ["kernel", "dom", "storage"],
  "crashPolicy": "disable",
  "priority": 30
}'>
```

| 字段 | 必填 | 说明 |
|---|---|---|
| `mode` | ✅ | 必须是 `"advanced"`（或 `"advanced": true`） |
| `permissions` | ✅ | 权限数组，见第 5 节；写 `["*"]` 表示全部 |
| `crashPolicy` | 否 | `"disable"`（默认）/ `"ignore"` |
| 其余字段 | 否 | 与普通插件一致 |

> 不声明 `permissions` 时，授权弹窗会按最小集 `kernel + dom` 处理，但建议显式写全，让用户知道自己在授权什么。

---

## 4. 高级模式与状态呈现

### 4.1 总开关

三个入口，效果一致：

| 入口 | 位置 |
|---|---|
| 设置页 | 设置 → 插件与内核 → 高级模式 |
| 插件中心 | 运行模式 → 高级模式 |
| API | `Mirage.advanced.set(true)` |

开启时会弹出确认框，说明风险。开启后：

- `localStorage['mirage.advanced.v1'] = '1'`；
- 已授权且已启用的高级插件会被自动拉起；
- **桌面布局不变**：不会新增任务栏、状态条或任何占位元素。

关闭时：

- 所有高级插件立即停止（`stopPlugin`）；
- 已授权记录**保留**，下次开启无需重新授权（源码未变的前提下）。

### 4.2 状态在哪里看（没有独立插件栏）

高级模式是一种「运行身份」，不是一块界面。它不占用任何屏幕空间，所以：

- `workArea()` 只减去任务栏高度，最大化窗口与普通模式完全一致；
- 通知（toast）位置、窗口吸附范围、移动端布局都不因高级模式而改变；
- 不存在 `#adv-bar` 这类元素，也不存在 `body.has-adv-bar` 这个 class。

状态收敛到两处：

| 位置 | 显示内容 |
|---|---|
| 插件中心 →「运行模式」卡片 | 开关状态 + 「已开启 · 已授权的高级插件会自动载入，当前运行 N 个」 |
| 设置页 →「高级模式」一行 | 同一个开关与同一句状态文案 |
| 插件中心 →「已安装」列表 | 每条记录的模式徽章（`沙箱` / `高级`）、权限标签、授权状态 |

需要确认「现在到底有没有高级插件在跑」，看插件中心列表里的绿点即可；也可以用 API：

```js
Mirage.advanced.list()   // [{ id, name, granted, running }]
```

> 如果你的插件想在界面上提示用户「高级模式已开启」，请自行在窗口或任务栏组件里画，系统不会代劳。

### 4.3 状态 API

```js
Mirage.advanced.enabled      // boolean，只读
Mirage.advanced.set(true)    // Promise<boolean>，会弹确认框
Mirage.advanced.list()       // [{ id, name, granted, running }]
Mirage.advanced.grants()     // { [id]: { hash, permissions, ts } }
```

---

## 5. 权限系统

### 5.1 权限表

| key | 名称 | 等级 | 说明 |
|---|---|---|---|
| `kernel` | 内核覆写 | 危险 | 拦截并改写 `openApp` / `closeWindow` / `toast` / `applySettings` 等核心函数 |
| `dom` | 页面完全控制 | 危险 | 直接读写页面 DOM、注入样式、替换任意界面元素 |
| `storage` | 全部本地数据 | 危险 | 读取和修改系统设置、笔记、应用、皮肤以及其他插件的数据 |
| `plugins` | 插件管理 | 危险 | 启用、禁用、安装、卸载其他插件 |
| `system` | 系统操作 | 危险 | 重置系统、导出快照、修改全局设置、触发重新加载 |
| `network` | 网络访问 | 敏感 | 向任意地址发起请求（仍受浏览器同源策略约束） |
| `clipboard` | 剪贴板 | 敏感 | 读写剪贴板内容 |

`"permissions": ["*"]` 等价于全部七项。

### 5.2 授权流程

```
用户点击「授权…」/ 首次启用
        │
        ▼
检查高级模式是否开启 ──否──▶ toast「高级模式未开启」+ 去开启
        │是
        ▼
检查是否已有有效授权 ──是──▶ 直接启动
        │否
        ▼
弹出权限清单（逐项勾选状态默认全选）
        │
   勾选「我已阅读并理解…」
        │
        ▼
写入 grants：{ hash: 源码指纹, permissions, ts }
        │
        ▼
启动插件
```

### 5.3 源码指纹

```js
pluginSourceHash(source) === hash36(source)
```

- 授权记录里保存的是源码哈希；
- 每次启动前重新计算，不一致 → 授权失效；
- 因此**升级插件 = 重新授权**，这是有意的：防止「先授权一个无害版本，再偷偷替换成恶意代码」。

### 5.4 授权记录

```js
localStorage['mirage.plugins.grants.v1'] = {
  "com.example.advanced": {
    "hash": "k3f9x2a8b1",
    "permissions": ["kernel", "dom", "storage"],
    "ts": 1790000000000
  }
}
```

### 5.5 撤销

| 方式 | 效果 |
|---|---|
| 插件中心 → 撤销授权 | 停止插件 + 删除授权记录 |
| `Mirage.plugins.revoke(id)` | 同上（不弹确认） |

撤销后该插件保持「已安装、已禁用」，直到重新授权。

### 5.6 授权相关 API

```js
Mirage.plugins.grant(id)      // Promise<boolean>，未授权时弹窗
Mirage.plugins.hasGrant(id)   // boolean
Mirage.plugins.revoke(id)     // true
```

---

## 6. 插件上下文 `ctx`

`Mirage.register(def)` 后，`def.start(ctx)` / `def.setup(ctx)` / `def.stop(ctx)` 都会拿到同一个 `ctx`。

### 6.1 基础字段

| 成员 | 说明 |
|---|---|
| `ctx.id` | 插件 id |
| `ctx.apiVersion` | `2` |
| `ctx.manifest` | 清单对象 |
| `ctx.plugin` | 插件记录（含 id/name/version/crashes…） |
| `ctx.document` | 宿主 `document` |
| `ctx.log(...args)` | 带前缀的 console.log |

### 6.2 应用与窗口

```js
const app = ctx.app({
  id: 'demo',
  name: '高级示例',
  sub: '全权限',
  icon: 'shield',
  desktop: true,
  w: 760, h: 540,
  render(root, win) {
    root.innerHTML = '<div>直接操作宿主 DOM</div>';
    root.querySelector('button').addEventListener('click', () => ctx.toast('ok'));
  }
});

app.key      // plugin.<id>.demo
app.open()   // 打开窗口
app.close()  // 关闭窗口
app.window   // 当前窗口实例或 null
```

```js
const handle = ctx.openWindow({
  title: '临时窗口',
  html: '<div>...</div>',
  width: 720, height: 520,
  icon: 'window',
  onClose() { /* ... */ }
});
handle.close();
handle.body;   // 窗口内容容器，可直接 appendChild
```

与普通插件的区别：`render` 拿到的是真实的 DOM 容器，可以直接 `innerHTML`、`addEventListener`、挂 React/Vue 也行。

### 6.3 UI 注入

```js
ctx.ui.desktopIcon({ id, name, icon, appKey, onOpen });
ctx.ui.menuItem({ label, icon, order, onClick });
ctx.ui.taskbarWidget({ render(el, ctx) });     // el 是任务栏里的真实 DOM
ctx.ui.settingsSection({ id, title, desc, render(card, ctx) });
```

全部返回一个「取消注册」函数：

```js
const off = ctx.ui.menuItem({ label: '我的菜单', onClick: () => {} });
off();  // 立即移除
```

### 6.4 事件

```js
const off = ctx.on('app-opened', (detail) => ctx.log('打开了', detail.key));
```

事件与普通插件一致（`app-opened` / `app-closed` / `settings-changed` / `theme-changed` / `skin-changed` / `plugin-started` / `plugin-stopped` 等），但 `detail` 是**原始对象**，包含窗口实例等不可序列化的内容。

### 6.5 定时器与清理

```js
const h = ctx.interval(() => { /* 每 5 秒 */ }, 5000);
ctx.timeout(() => { /* 3 秒后 */ }, 3000);
ctx.clearTimer(h);

ctx.cleanup(() => { /* 插件停止时执行 */ });
```

- 定时器回调执行期间会标记 `activePlugin`，便于错误归属；
- 插件停止时，`ctx.interval` / `ctx.timeout` 创建的定时器会被统一清除；
- `ctx.cleanup` 注册的函数会在停止时**逆序**执行。

### 6.6 样式与 DOM

```js
ctx.css('.my-class{color:red}');                 // 全局样式，停用时移除
const node = ctx.mount(document.body, '<div>hi</div>');  // 挂到任意位置，停用时移除
```

`ctx.css` 注入的是**全局**样式，会影响整个系统界面——这正是高级插件的能力，也是风险。

### 6.7 存储

```js
ctx.storage.set('history', [...]);
const list = ctx.storage.get('history', []);
ctx.storage.remove('history');
ctx.storage.keys();
```

- 同步 API；
- 实际 key 为 `mirage.plugin.<id>.<key>`；
- 与普通插件的命名空间相同，可以互相读取（同一插件）。

### 6.8 交互

```js
ctx.toast('消息', 'success');
ctx.notify('消息');
const ok = await ctx.confirm({ title, message, confirmText, danger });
const value = await ctx.prompt({ title, message, value });
```

### 6.9 内核

```js
ctx.kernel.override(name, fn);   // 返回取消函数
ctx.kernel.restore(name);        // 移除本插件在该函数上的所有钩子
ctx.kernel.list();               // { openApp: 2, toast: 1, ... }
ctx.kernel.api();                // 拿到 state / settings / BUILTINS / bus 等
```

详见第 7 节。

### 6.10 高级能力

```js
ctx.advanced.document           // === window.document
ctx.advanced.window             // === window
ctx.advanced.permissions        // ['kernel','dom',...]
ctx.advanced.has('kernel')      // boolean
ctx.advanced.kernel             // 同 ctx.kernel
ctx.advanced.eval('return 1+1') // 在宿主上下文执行代码（需 dom 权限）
```

---

## 7. 内核钩子

### 7.1 可拦截的函数

| 名称 | 签名 | 触发时机 |
|---|---|---|
| `openApp` | `(key) => win` | 打开任意应用 |
| `closeWindow` | `(win) => void` | 关闭窗口 |
| `renderDesktopIcons` | `(animateKey?) => void` | 重绘桌面图标 |
| `renderStartApps` | `(query?) => void` | 重绘开始菜单 |
| `applySettings` | `() => void` | 应用设置到 DOM |
| `setSettings` | `(patch, opts) => void` | 修改设置 |
| `toast` | `(message, type, opts) => void` | 弹通知 |
| `showMenu` | `(items, x, y) => void` | 显示右键菜单 |

### 7.2 钩子签名

```js
ctx.kernel.override('openApp', (next, key) => {
  // 前置逻辑
  const win = next(key);   // 调用下一个钩子 / 原函数
  // 后置逻辑
  return win;              // 返回值会原样传回调用方
});
```

- 多个插件可以拦截同一个函数，按**注册顺序**串行执行；
- 不调用 `next()` 就等于「吞掉」这次调用（可用于拦截/改写）；
- 钩子内抛错会被捕获，记录错误后自动 `next()` 继续，不会卡死系统；
- 返回 `next()` 的返回值，调用方才能拿到正确结果。

### 7.3 示例：记录所有窗口打开

```js
ctx.kernel.override('openApp', (next, key) => {
  history.unshift({ key, ts: Date.now() });   // 先记录，再放行
  ctx.storage.set('history', history);
  return next(key);
});
```

### 7.4 示例：拦截敏感操作

```js
ctx.kernel.override('setSettings', (next, patch, opts) => {
  if (patch && patch.wallpaper === 'custom') {
    ctx.toast('壁纸修改已被插件拦截', 'warn');
    return;
  }
  return next(patch, opts);
});
```

### 7.5 恢复

```js
ctx.kernel.restore('openApp');   // 移除本插件在 openApp 上的全部钩子
```

---

## 8. 系统级 API（`ctx.advanced.system`）

```js
ctx.advanced.system.listPlugins()
// [{ id, name, mode, enabled, running, crashes }]

ctx.advanced.system.enablePlugin(id)     // Promise<boolean>
ctx.advanced.system.disablePlugin(id)    // Promise<void>
ctx.advanced.system.reloadPlugin(id)     // Promise<boolean>
ctx.advanced.system.uninstallPlugin(id)  // Promise<void>

ctx.advanced.system.getSettings()        // 设置副本
ctx.advanced.system.setSettings(patch)   // 修改设置
ctx.advanced.system.resetSystem()        // 清空本机数据（会弹确认）
ctx.advanced.system.exportHtml()         // 生成导出 HTML 字符串
ctx.advanced.system.reload()             // 重新加载页面
```

### 8.1 全局存储

```js
ctx.advanced.storage.get('mirage.settings.v1', null);
ctx.advanced.storage.set('my.key', { a: 1 });
ctx.advanced.storage.remove('my.key');
ctx.advanced.storage.keys();   // 全部 key，包括系统与其他插件的
```

### 8.2 任意代码执行

```js
const result = ctx.advanced.eval('return document.title');
```

在宿主上下文执行，拥有与插件相同的权限。**这是最高级别能力，仅在你完全掌控代码时使用。**

### 8.3 事件总线

```js
ctx.advanced.emit('my-event', { a: 1 });
ctx.advanced.bus;   // 原始 EventTarget
```

---

## 9. 生命周期与心跳豁免

### 9.1 生命周期

```
setup(ctx)   ── 可选，注册时执行一次
   │
start(ctx)   ── 启动时执行（可 async，宿主会 await）
   │
   ├── 运行中：定时器、钩子、窗口、事件
   │
stop(ctx)    ── 停止时执行（可 async）
   │
   └── 之后：应用/窗口/图标/菜单/样式/定时器/钩子全部清理
```

### 9.2 心跳豁免

普通插件在回调执行期间会写 `mirage.plugins.runtime.v1` 标记，宿主据此判断「是否卡死」。高级插件**完全不写这个标记**，因此：

| 机制 | 普通插件 | 高级插件 |
|---|---|---|
| 运行时标记 | ✅ 写入 | ❌ 不写 |
| 主线程卡顿检测（>12s） | ✅ 判定卡死 | ❌ 跳过 |
| 启动看门狗（15s） | ✅ 判定超时 | ❌ 跳过 |
| `window.onerror` 捕获 | ✅ | ✅ |
| 15 秒 3 次错误禁用 | ✅ | ✅（除非 `crashPolicy: "ignore"`） |

理由：高级插件可能本来就在做长任务（批量处理、同步大量数据），用「卡顿」去判定它崩溃会误杀。

### 9.3 崩溃策略

| `crashPolicy` | 启动抛错 | 运行抛错 |
|---|---|---|
| `"disable"`（默认） | 禁用 + 重载页面 | 15 秒内 3 次则禁用 + 重载 |
| `"ignore"` | 记录 + 停止，但**不禁用**、不重载 | 只记录，永不禁用 |

`ignore` 适合常驻型插件：崩了下次启动会自己重试，不需要用户手动重新启用。

---

## 10. 安全边界

### 10.1 能做什么

理论上，**系统能做的它都能做**：

- 读写任意 DOM、替换界面、注入全局样式；
- 读写全部 `localStorage` / IndexedDB；
- 拦截任意内核函数；
- 管理其他插件（含卸载）；
- 修改设置、导出系统、重置数据；
- 发网络请求、读写剪贴板（受浏览器权限约束）。

### 10.2 不能做什么

| 限制 | 说明 |
|---|---|
| 突破浏览器同源策略 | 不能读其他域的页面 |
| 绕过系统确认框 | `resetSystem` / `uninstall` 等仍会弹确认 |
| 逃避错误记录 | 所有异常都会进入插件中心日志 |
| 在安全模式运行 | `?safe=1` 时一律不加载 |

### 10.3 权限的真相

高级插件与宿主同上下文，**运行期技术上无法真正拦截**一次 `document.body.innerHTML = ''`。所以权限系统的作用是：

- **安装时**让用户知情：逐项列出这份代码会做什么；
- **启动前**做一次闸门：没授权就不执行；
- **源码变更**时强制重新确认。

它不是一个运行时沙箱。请把它理解为「安装软件时的权限清单」，而不是「手机 App 的沙箱」。

### 10.4 建议

- 只运行自己写的、或逐行读过的代码；
- 不确定时，先用普通（沙箱）模式实现；
- 长期不用时，关闭高级模式或撤销授权；
- 怀疑出问题时，用 `?safe=1` 启动，再逐个排查。

---

## 11. 调试

1. **DevTools → Sources**：插件代码带 `//# sourceURL=mirage-plugin://<id>`，会单独出现在 Sources 面板里，可直接下断点。
2. **Console**：`ctx.log()` 输出前缀 `[plugin:<id>]`。
3. **插件中心 → 崩溃与恢复**：最近 40 条错误，含阶段（start / runtime / hang）。
4. **重载**：插件中心点「重载」，等价于 `stop()` → 重新执行源码，不需要刷新页面。
5. **断点调试启动流程**：在 `startAdvancedPlugin` 里下断点，或在 `ctx.kernel.override` 里打日志。

---

## 12. 完整示例

见同目录 `plugins/kernel-demo.html`。核心结构：

```html
<meta name="mirage-plugin" content='{
  "id":"com.mirage.demo.kernel",
  "name":"内核观测台",
  "version":"2.0.0",
  "mode":"advanced",
  "permissions":["kernel","dom","storage"],
  "crashPolicy":"disable",
  "priority":30
}'>
<script>
Mirage.register({
  id: 'com.mirage.demo.kernel',
  start(ctx) {
    const history = ctx.storage.get('history', []) || [];

    const app = ctx.app({
      id: 'kernel',
      name: '内核观测台',
      icon: 'shield',
      desktop: true,
      w: 780, h: 560,
      render(root) {
        root.innerHTML = '<div class="kd">...</div>';
        root.querySelector('[data-kd="clear"]').addEventListener('click', () => {
          history.length = 0;
          ctx.storage.set('history', []);
        });
      }
    });

    ctx.kernel.override('openApp', (next, key) => {
      history.unshift({ key, ts: Date.now() });
      ctx.storage.set('history', history);
      return next(key);
    });

    ctx.css('body.kernel-demo-ready .win{border-color:rgba(52,211,153,.45)!important}');
    document.body.classList.add('kernel-demo-ready');
    ctx.cleanup(() => document.body.classList.remove('kernel-demo-ready'));

    ctx.advanced.system.listPlugins();
    app.open();
  },
  stop(ctx) { ctx.log('已停止'); }
});
</script>
```

---

## 13. 从 API 1 迁移

2.1.0 的插件全部是「全权限」写法，没有 `mode` 字段。升级到 2.2.0 后：

| 旧写法 | 现在 |
|---|---|
| 不写 `mode` | 会被当成**沙箱**插件，`ctx.*` 不存在 |
| `ctx.app({ render(root) {...} })` | 沙箱：`render()` 返回 HTML；高级：保持原样 |
| `ctx.kernel.override` | 仅高级可用 |
| `ctx.document` | 仅高级可用 |
| `Mirage.register({ start(ctx) })` | 两种模式都支持 |

### 迁移方案 A：保持全权限

在清单里加：

```json
"mode": "advanced",
"permissions": ["kernel", "dom", "storage"]
```

安装后需要在插件中心授权一次。

### 迁移方案 B：改成沙箱插件（推荐）

1. 把 `render(root)` 改成 `render()` 返回 HTML 字符串；
2. 把 `root.querySelector(...).addEventListener(...)` 改成 `data-mirage-action` + `onAction`；
3. 把 `ctx.kernel.*`、`ctx.document`、`ctx.css` 相关逻辑删掉（沙箱不支持）；
4. `ctx.storage` 改成 `await Mirage.storage.*`；
5. `ctx.toast` → `Mirage.toast`。

改造后无需授权、崩溃隔离、随装随用。

---

## 14. 常见问题

**Q：为什么插件装了却不运行？**
A：依次检查：高级模式是否开启 → 是否已授权 → 源码是否被改过（指纹失效）→ 是否在安全模式。

**Q：为什么授权后又提示「待授权」？**
A：源码变了。任何字符改动都会让指纹失效，包括格式化、换行。

**Q：高级插件崩溃会把系统搞坏吗？**
A：不会。异常会被捕获并记录；默认策略下插件被禁用并重载页面，其他插件与用户数据不受影响。只有插件主动执行的破坏性 DOM 操作才可能影响界面，重载即可恢复。

**Q：能让高级插件开机自动运行吗？**
A：可以。高级模式开启 + 授权有效 + `enabled: true`，开机就会自动加载，不会弹窗。

**Q：`crashPolicy: "ignore"` 会不会导致死循环重载？**
A：不会。`ignore` 不触发重载，只在日志里记录。
