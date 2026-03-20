---
name: minemap-anti-aliasing
description: MineMap 抗锯齿与边缘平滑规范。用于 MSAA、FXAA、TAA、fill-antialias 的选型、启停与性能取舍。
---

# MineMap Anti-aliasing

## Architecture Positioning

MineMap 里“抗锯齿”不是单一开关，而是四类不同层级的能力：

1. `map.MSAASamples`：多重采样抗锯齿，偏底层渲染管线
2. `map.fxaa`：屏幕后处理 FXAA，偏全屏图像平滑
3. `map.taa`：时间累积抗锯齿，偏相邻帧稳定化
4. `fill-antialias`：fill 图层边缘抗锯齿，只作用于填充轮廓

因此不要把它们混成同一个抽象：“地图抗锯齿已开启”。业务要先判断自己要解决的是：

-   三维模型边缘锯齿
-   屏幕空间整体锯齿
-   静态视角下的细边闪烁
-   矢量 fill 边界轮廓的 1px 边缘处理

## Source-backed Entry Surface

源码和文档已确认以下正式入口：

-   `new minemap.Map({ MSAASamples, fxaa })`
-   `map.MSAASamples`
-   `map.fxaa`
-   `map.taa`
-   fill layer paint: `fill-antialias`

对应依据：

-   `source/api/map.js`
-   `source/renderer/Painter.js`
-   `source/renderer/post-process/PostProcessStageLibrary.js`
-   `source/style/style_layer/fill_style_layer.js`

## Demo-backed Patterns

### 模式 1：MSAA 作为三维场景的首选硬件抗锯齿

对应 demo：`demo/html/MSAA.html`

这个 demo 直接把 `MSAASamples` 暴露成运行时参数：

-   `MSAASamples: 8`
-   运行时再切 `map.MSAASamples = 1 / 2 / 4 / 8`

说明 MineMap 把 MSAA 设计成正式的地图级画质参数，而不是私有调试项。

适合：

-   3D 模型
-   3D Tiles
-   线框 / 几何边缘较多的三维场景

### 模式 2：FXAA 作为全屏后处理抗锯齿

对应 demo：`demo/html/PostProcessFXAA.html`

demo 的正式用法不是手搓 stage，而是直接：

-   `map.fxaa = true`

这说明业务层若只想要“开关 FXAA”，优先走 `map.fxaa`，不要默认直接操作内部 post-process stage。

适合：

-   希望快速平滑整体画面
-   后期补救屏幕空间锯齿
-   不想单独为每种几何配置抗锯齿策略

### 模式 3：TAA 适合相机较稳定的观察场景

对应 demo：`demo/html/TAA.html`

demo 开启方式同样很直接：

-   `map.taa = true`

而 `Painter.js` 里的实现还能看到一个关键约束：

-   只有在 `taaPass.enabled && !viewChanged` 时，才给投影矩阵加入细微抖动

这意味着 TAA 更适合：

-   静态观察
-   低速漫游
-   BIM / 模型查看器

不适合被理解成“任何快速运动视角下都能稳定提升边缘质量”。

### 模式 4：fill-antialias 只管 fill 边缘

对应 demo：

-   `demo/html/FillStencilTestWithTerrain.html`
-   `demo/html/WaterFillLayer.html`

源码文档已明确：

-   `fill-antialias` 是 fill 图层 paint 属性
-   默认值为 `true`

demo 也给出两种真实取值：

-   常规填充：`"fill-antialias": true`
-   水面 fill：`"fill-antialias": false`

这说明它不是“全局画质开关”，而是图层局部轮廓策略。

## Recommended Selection Strategy

### 1. 三维模型 / Tiles 优先看 MSAA

如果你主要问题是：

-   建筑边缘发锯齿
-   模型轮廓硬
-   倾斜摄影 / 模型边界不平滑

优先从 `MSAASamples` 入手。

### 2. 需要快速全屏平滑时开 FXAA

如果你要的是“快速整体变柔和一些”，优先：

-   `map.fxaa = true`

### 3. 视角较稳定时再评估 TAA

如果业务画面经常停留、缓动、慢速查看，TAA 才更有意义。

### 4. fill 图层边缘单独看 `fill-antialias`

如果只是一层行政区、矢量面、水面边界问题，不要上升到全局 FXAA / TAA，先检查 `fill-antialias`。

## Strict Constraints

### 1. `MSAASamples` 不是任意整数都照单全收

`Context` 里已明确：

-   小于 `1` 会被重置为 `1`
-   大于 `ContextLimits.maximumSamples` 会被钳制到最大值

因此业务不要假设：

-   `16x` 一定可用
-   输入多少就一定生效多少

### 2. FXAA 属于后处理，不是几何级抗锯齿

FXAA 的正式实现来自后处理链。它的优点是简单，但也意味着它处理的是最终画面，不是重建真实几何边缘。

### 3. TAA 对相机状态敏感

`Painter.js` 中 `viewChanged` 逻辑已经说明：

-   TAA 的价值和相机是否稳定直接相关

所以不要把它当成“高速相机运动时的万能解”。

### 4. `fill-antialias` 不替代全局抗锯齿

它只影响 fill 图层边缘轮廓，不会替代：

-   模型边缘抗锯齿
-   后处理 FXAA
-   时间累积 TAA

## Failure Cases

### 失败场景 1：所有抗锯齿一起常驻全开

MSAA、FXAA、TAA、SSR、阴影、Bloom 全开会迅速抬高成本。抗锯齿应该是“按场景选主方案”，不是“能开的都开”。

### 失败场景 2：只会调全局，不会调图层

如果问题只是 fill 边缘发虚或边界过硬，优先从 `fill-antialias` 处理，而不是直接改全局渲染策略。

### 失败场景 3：把 TAA 当成高速运动场景的稳定方案

源码已经显示 TAA 和相机静稳度强相关。快速移动、频繁转向的场景，效果并不会像静态视角那样稳定。

## Professional Guidance

### 方案 A：三维质量优先

-   首选 `MSAASamples: 4`
-   不够再评估 `8`
-   必要时再叠加轻量后处理

### 方案 B：成本敏感的全屏平滑

-   先 `MSAASamples: 1`
-   再 `map.fxaa = true`

### 方案 C：静态浏览 / 方案查看

-   可评估 `map.taa = true`
-   但要接受其对运动状态更敏感

### 方案 D：矢量填充边缘控制

-   默认保留 `fill-antialias: true`
-   若是特殊水面、需要干净边缘控制，再按 demo 评估关掉

## Performance Discipline

-   `MSAASamples` 先从 `2` 或 `4` 起，不要默认拉满
-   FXAA / TAA 只开一个主方案再观察，不要无脑叠加
-   fill 图层问题优先局部属性解决
-   低端设备或 `webgl1` 兼容环境要更保守地选择抗锯齿策略

## Demo References

-   `demo/html/MSAA.html`
-   `demo/html/PostProcessFXAA.html`
-   `demo/html/TAA.html`
-   `demo/html/FillStencilTestWithTerrain.html`
-   `demo/html/WaterFillLayer.html`

## See Also

-   `minemap-performance-and-backend`
-   `minemap-post-process-and-render-pipeline`
-   `minemap-style-and-data`
-   `minemap-lighting-and-shadows`
