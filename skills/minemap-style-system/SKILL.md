---
name: minemap-style-system
description: MineMap style 系统规范。用于 style v8 结构、`setStyle()`、diff/keepUserInfo、sprite/glyphs 与样式切换工作流。
---

# MineMap Style System

## Architecture Positioning

MineMap 的 style 不只是“底图 JSON”，而是整个地图渲染状态的组织入口。

从 `doc-style.js` 与 `source/api/map.js` 可以确认，style 的核心职责包括：

-   定义 `sources`
-   定义 `layers`
-   定义 `sprite`
-   定义 `glyphs`
-   可选地携带默认相机参数（`center` / `zoom` / `bearing` / `pitch`）

因此，style 变更往往不只是“换皮肤”，而是重建或差量更新地图渲染结构。

## Source-backed Entry Surface

style 相关正式入口主要是：

-   `map.setStyle(style, options?)`
-   `map.getStyle()`
-   `map.isStyleLoaded()`
-   `style.load` 事件

`setStyle()` 的关键选项：

-   `diff`
-   `keepUserInfo`
-   `useCustomCamera`
-   `localIdeographFontFamily`
-   `projection`

## Style-backed Core Model

### 1. MineMap 使用 style spec v8

`doc-style.js` 已明确：

-   `version: 8`
-   结构主干是 `sources` + `layers`

### 2. style 可来自 URL 或 JSON

`setStyle()` 的实现明确支持：

-   URL
-   JSON 对象
-   `null`

### 3. `setStyle()` 不一定等于整图清空重建

`source/api/map.js` 里可见：

-   若 `options.keepUserInfo` 或 `options.diff` 生效，会走 `_diffStyle()` / `_updateDiff()`
-   否则会清场再 `_updateStyle()`

这意味着 style 切换有两种完全不同的成本模型：

-   差量更新
-   全量重建

## Demo-backed / Source-backed Patterns

### 模式 1：`style.load` 后再补业务 source/layer

这是 MineMap 里最重要的 style 工作流规则之一。

源码与已有 demo 都反复说明：

-   style 还没好时，不要抢着加业务 layer

因此业务代码应优先组织为：

-   `map.on('style.load', ...)`

### 模式 2：业务样式与用户增量内容分开

`setStyle(style, { keepUserInfo: true })` 的存在，说明 MineMap 已明确区分：

-   基础 style
-   用户运行时追加内容

这对这些场景很重要：

-   切换底图主题
-   切换白天/夜间风格
-   仍保留业务临时叠加层

### 模式 3：需要 diff 时才用 diff

`setStyle(..., { diff: true })` 不是默认无脑选项。

源码里若差量失败，会 fallback 到完整重建，并发出警告。这说明 diff 是“优化路径”，不是“绝对正确路径”。

### 模式 4：style 相机会覆盖地图默认相机，除非显式关闭

`setStyle()` 选项里有：

-   `useCustomCamera`

这说明 style 切换时，相机是否跟随 style 配置，是一项必须显式考虑的架构问题。

### 模式 5：style 不只是视觉，还包含资源协议

`doc-style.js` 已明确：

-   `sprite`
-   `glyphs`

因此样式切换时不能只想着“颜色和图层”，还要考虑：

-   图标资源是否一致
-   字体资源是否一致

## Recommended Workflow

1. 先把基础底图 style 视为“渲染基线”
2. 业务 source/layer 放在 `style.load` 后挂载
3. 若是换主题但要保留业务层，优先评估 `keepUserInfo`
4. 若只是小范围变更，才评估 `diff`
5. 若风格切换涉及相机保持，显式处理 `useCustomCamera`

## Strict Constraints

### 1. 不要在 style 未完成时添加业务 layer

这是 MineMap 里最常见的时序问题之一。

### 2. `diff` 不是永远安全

源码里已经有 fallback 逻辑，说明复杂 style 变化未必总能稳定 diff 成功。

### 3. style 切换会影响整个渲染结构

它不是单纯改几个颜色字段，可能涉及：

-   source 结构
-   layer 顺序
-   资源引用
-   相机状态

### 4. `keepUserInfo` 只在真的有“用户追加内容”时才有意义

不要默认把所有 style 切换都绑定到它，否则会让运行时状态越来越难推断。

## Failure Cases

### 失败场景 1：把 style 当成普通配置对象随便覆盖

style 是地图渲染主状态，不是一般 UI 配置。

### 失败场景 2：切 style 后忘记业务层重挂载策略

若没有 `keepUserInfo`，很多运行时附加 source/layer 会被清掉。

### 失败场景 3：style 切换后相机突然变了，却不知道为什么

这通常和 style 自带相机参数、`useCustomCamera` 配置有关。

## Performance Discipline

-   小改动优先 property 更新，不优先 `setStyle()`
-   需要换整套底图风格时，再考虑 style 切换
-   差量更新只在收益明确时使用
-   将业务增量图层与基础 style 拆层管理

## Evidence References

-   `source/api/map.js`
-   `docs/src-cn/doc-style.js`
-   `docs/src-cn/doc-source.js`
-   `docs/src-cn/doc-style-layer-migrate.js`
-   `demo/html/BeiDouCodecDemo.html`

## See Also

-   `minemap-style-and-data`
-   `minemap-layer-system`
-   `minemap-fundamentals`
-   `minemap-scene-components`
