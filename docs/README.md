# 忘忧 OS 文档

当前版本：**3.0.0**

| 文档 | 内容 | 适合 |
|---|---|---|
| [PLUGIN-SPEC.md](PLUGIN-SPEC.md) | 普通（沙箱）插件：清单、桥接协议、API、视图交互、崩溃恢复 | 想写一个能安全分享的插件 |
| [ADVANCED-PLUGIN-SPEC.md](ADVANCED-PLUGIN-SPEC.md) | 高级（全权限）插件：权限系统、`ctx` 成员、内核钩子、心跳豁免 | 想深度改造系统 |
| [UI-SKIN-SPEC.md](UI-SKIN-SPEC.md) | UI 引擎：载入方式、设计令牌、选择器契约、覆盖规则 | 想换掉整套界面外观 |

## 快速入口

- 第一次写插件：先读 [PLUGIN-SPEC.md](PLUGIN-SPEC.md) 的「五分钟写一个」。
- 需要直接操作宿主 DOM 或内核：读 [ADVANCED-PLUGIN-SPEC.md](ADVANCED-PLUGIN-SPEC.md) 的权限与授权部分。
- 只想改配色、圆角、阴影：读 [UI-SKIN-SPEC.md](UI-SKIN-SPEC.md) 的设计令牌表。

## 兼容说明

3.0 完成品牌更名（幻境 OS → 忘忧 OS），但以下内部标识保持不变：

- 插件清单：`<meta name="mirage-plugin">`
- 插件 API：`Mirage.*`
- 本地存储：`mirage.*`
- 皮肤样式标记：`data-mirage-skin`

2.x 的插件、皮肤与本地数据可以继续在 3.0 中使用。

## 截图

`docs/screenshots/` 保存 README 与文档使用的界面截图。截图应从当前 `index.html` 实际运行界面生成。
