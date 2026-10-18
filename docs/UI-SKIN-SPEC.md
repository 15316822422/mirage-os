# 幻境 OS · UI 引擎规范

> 版本 2.0 · 适用于 `index.html`
> 一句话：**一份 HTML 文件就是一套界面**。载入后，引擎把它的 `<style>` 整段注入到页面最后，整站外观由它接管；引擎只负责结构与行为。

---

## 1. 四种载入方式

| 方式 | 操作 | 是否持久化 |
| --- | --- | --- |
| **同目录自动读取** | 把皮肤命名为 `skin.html` / `ui-skin.html` / `theme.html` / `mirage-skin.html` / `skin.css` / `ui.css`，放在 `index.html` 旁边，启动时自动套用 | 否（每次启动重新读取，改文件即生效） |
| **界面内载入** | 「界面皮肤」应用 → 选择皮肤文件，或直接把文件拖进桌面 | 是（记入 localStorage 与皮肤库） |
| **直接写入** | 「界面皮肤」应用 → 「直接写入」文本框，粘贴 CSS 或整份 HTML 后点应用 | 是 |
| **内嵌 / 导出** | 导出整个系统时，当前皮肤会一并写进导出文件 | 是（随文件走） |

**优先级**（从高到低）：内嵌于本文件 → 上次应用的皮肤 → 同目录文件 → 引擎默认外观。

> 想让同目录文件优先于旧记录：打开「界面皮肤」→「重新扫描同目录」，它会重新读取并覆盖当前记录。

### 本地文件（`file://`）限制

浏览器出于安全策略禁止 `fetch` 读取本地同目录文件，因此**双击打开 `index.html` 时自动读取会失效**，界面会提示一次。此时请用「界面皮肤 → 选择皮肤文件」手动载入，或把系统部署到任意静态服务器 / 打包进导出文件。

也可以在本文件的 `<head>` 里显式声明皮肤地址（仍需可被 `fetch` 读取）：

```html
<link rel="mirage-skin" href="skins/porcelain.html">
```

---

## 2. 文件格式

- 单个 `.html` 或 `.css` 文件，UTF-8 编码。
- 只读取 `<style>` 内容；**皮肤里的 `<script>` 不会执行**。
- 必须包含 `<style data-mirage-skin>`。若没有该属性，引擎会退而读取文件内所有 `<style>`。
- 体积上限 1.6 MB，建议 < 200 KB。
- 不要引用外链字体 / 图片，否则离线打开会缺样式。

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
<meta charset="utf-8">
<title>我的皮肤</title>

<script type="application/json" id="mirage-skin-manifest">
{
  "id": "my-skin",
  "name": "我的皮肤",
  "author": "你的名字",
  "version": "1.0",
  "description": "一句话描述",
  "theme": "dark",
  "accent": "#7c8cff",
  "tokens": { "--ui-radius": "12px" }
}
</script>

<style data-mirage-skin>
:root{
  --ui-surface: rgba(20,25,35,.5);
  --ui-radius: 14px;
}
</style>
</head>
<body></body>
</html>
```

---

## 3. 清单字段（可选）

`<script type="application/json" id="mirage-skin-manifest">` 内的字段全部可选：

| 字段 | 类型 | 作用 |
| --- | --- | --- |
| `id` | string | 皮肤标识，同时写入 `html[data-skin]`，用于作用域限定。缺省按内容哈希生成 |
| `name` | string | 显示名称。缺省取文件名 |
| `author` / `version` / `description` | string | 皮肤库展示用 |
| `theme` | `"dark"` \| `"light"` | 载入时同步切换主题模式 |
| `accent` | `#rrggbb` | 载入时同步设置强调色 |
| `wallpaper` | `aurora` \| `dusk` \| `ocean` \| `nebula` \| `mono` | 载入时切换壁纸 |
| `tokens` | object | 追加到 CSS 最前面的 `:root{}`，适合只改令牌的轻量皮肤 |

---

## 4. 设计令牌

在皮肤的 `:root` 里覆盖这些变量，就能整体改变观感。**令牌优先，组件其次**。

| 令牌 | 含义 |
| --- | --- |
| `--ui-bg` | 底层背景色 |
| `--ui-surface` | 窗口、任务栏的半透明底色 |
| `--ui-surface-2` | 开始菜单、浮层底色 |
| `--ui-surface-3` | 右键菜单、弹窗、通知底色 |
| `--ui-body` | 窗口内容区底色 |
| `--ui-elev` | 轻填充：输入框、卡片、按钮 |
| `--ui-elev-2` | 悬停 / 激活填充 |
| `--ui-border` | 常规描边 |
| `--ui-border-soft` | 分隔线与弱描边 |
| `--ui-text` | 主文字 |
| `--ui-dim` | 次要文字 |
| `--ui-dim-2` | 弱化文字 |
| `--ui-accent` / `--ui-accent-rgb` | 强调色；默认跟随设置面板 |
| `--ui-ink` | 强调色上的文字色 |
| `--ui-radius` / `--ui-radius-sm` / `--ui-radius-lg` | 圆角 |
| `--ui-blur` / `--ui-blur-lg` | 玻璃模糊半径（设 `0px` 即完全实心） |
| `--ui-shadow` / `--ui-shadow-sm` | 投影 |
| `--ui-scrim` | 弹窗遮罩 |
| `--ui-taskbar-h` | 任务栏高度 |
| `--ui-font` / `--ui-mono` | 字体 |
| `--ui-ease` / `--ui-dur` | 缓动与时长 |
| `--tile-bg-a` / `--tile-border-a` / `--tile-ink-l` | 图标底片的透明度与亮度 |
| `--label-bg` / `--label-ink` | 桌面图标文字标签 |

旧变量名（`--glass`、`--stroke`、`--text`、`--panel-1` 等）仍然可用，它们已指向新令牌。

---

## 5. 选择器契约

这些结构名是稳定契约，皮肤可以放心依赖。

| 选择器 | 对应区域 |
| --- | --- |
| `html[data-skin]` | 皮肤作用域（值为皮肤 `id`） |
| `#wallpaper` / `#desktop` | 壁纸层与桌面 |
| `.d-icon` / `.d-icon-art` / `.d-icon-label` | 桌面图标、底片、文字标签 |
| `#taskbar` / `.tb-btn` / `.tb-glyph` | 任务栏与其中按钮 |
| `#start` / `.start-search` / `.start-app` | 开始菜单 |
| `.win` / `.win-titlebar` / `.win-btn` / `.win-body` | 窗口外壳、标题栏、控制按钮、内容区 |
| `#ctx` / `.ctx-item` | 右键菜单 |
| `#modal-layer` / `.dialog` | 确认框、输入弹窗 |
| `.btn` / `.btn.primary` / `.icon-btn` | 按钮 |
| `#toasts` / `.toast` | 通知 |
| `.settings` / `.set-card` / `.set-row` | 设置页 |
| `.files` / `.file-row` | 应用库 |
| `.skin-app` / `.skin-card` / `.skin-item` | 皮肤管理页本身 |

> 桌面图标为 92 × 108 px，文字标签固定占两行（超长名称在第二行末尾省略）。若皮肤要改 `.d-icon` 的高度，请保证 ≥ 108 px（移动端 ≥ 102 px），否则标签第二行会被裁切。

---

## 6. 覆盖规则

1. 皮肤样式注入在基础样式**之后**，同优先级下后者胜出，**不需要 `!important`**。
2. 基础样式刻意保持低优先级，`.d-icon-art{}`、`.tb-btn .tb-glyph{}` 这类单层选择器即可覆盖。
3. 需要更保险时，用 `html[data-skin="你的id"]` 限定作用域，既提高优先级，也避免误伤。
4. 主题相关的差异通过 `--tile-*`、`--label-*` 等标量令牌表达，不必分别写深色 / 浅色两套选择器。
5. 覆盖壁纸这类带属性选择器的规则时，建议写成 `html[data-skin="id"] #wallpaper{}`。

---

## 7. 内置模板

`skins/` 目录下有三套可直接使用的完整皮肤，分别演示三种设计方向：

| 文件 | 名称 | 特征 |
| --- | --- | --- |
| `skins/porcelain.html` | 瓷白 | 浅色、纸面、零模糊、衬线标题、细描边 |
| `skins/neon.html` | 霓虹 | 近黑 + 荧光绿、锐角、等宽标签 |
| `skins/graphite.html` | 石墨 | 暖灰、直角、完全不透明、琥珀强调 |

把任意一个复制成 `index.html` 同级的 `skin.html`，刷新即可看到自动读取效果。

---

## 8. 安全边界

- 皮肤只能提供 CSS，不能执行 JavaScript。
- 不支持外链资源预加载，不做网络请求。
- 解析在 `DOMParser` 中完成，不插入任何皮肤 DOM 节点，只取 `<style>` 文本。
