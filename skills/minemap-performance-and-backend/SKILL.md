---
name: minemap-performance-and-backend
description: MineMap 渲染后端偏好、当前 WebGL2/WebGL1 运行路径、降级策略、特效开关与性能调优；适用于核对设备实际后端和高成本渲染功能。
metadata:
  minemap-version: 4.28.2
---

# MineMap Performance and Backend

## Quick Start

```javascript
const map = new minemap.Map({
	container: "map",
	style,
	renderBackend: "auto",
	// 或细粒度：renderBackendPreference: ["webgl2", "webgl1"]
	preserveDrawingBuffer: false,
	failIfMajorPerformanceCaveat: false
});

map.on("load", () => {
	console.log("resolved backend:", map.renderBackendResolved);
	console.log("backend reasons:", map.renderBackendReasons);
});
```

## Core Concepts

### 1) 后端优先级

默认偏好数组包含 `webgpu -> webgl2 -> webgl1`，但 4.27.1 的 WebGPU 通道是预留占位，创建上下文时会跳过它；当前实际后端是 WebGL2，或在要求时使用 WebGL1。不要把 `renderBackend: "webgpu"` 描述为已启用 WebGPU 渲染。

构造参数：

-   `renderBackend`: `auto | webgpu | webgl2 | webgl1`
-   `renderBackendPreference`: 自定义数组优先级
-   兼容参数：`requestWebgl1`

### 2) 运行时结果

初始化后可读取：

-   `map.renderBackendResolved`
-   `map.renderBackendReasons`
-   `map.requestWebgl1`

业务代码在初始化完成后读取 `map.getRenderBackend()` / `map.renderBackendResolved`，并基于实际后端决定是否开放依赖 WebGL2 的功能。后端选项表示偏好，不代表浏览器与引擎已实现该图形 API。

### 3) 特效开关与性能成本

高成本项（按需开）：

-   `SSR`
-   `OIT`
-   阴影（`enableShadow`）
-   大气/云层
-   高频后处理链

### 4) WebGL1 兼容注意

WebGL1 下对高级能力（MRT、Transform Feedback、3D/2DArray 纹理）要走降级路径或关闭相关特效。

## Demo-backed Patterns

### 阴影是典型高成本开关

多个 demo 都采用同一套路：

1. `map.enableShadow()`
2. `map.setShadowType(...)`
3. 再通过 GUI 实时切换

这说明阴影不应默认无脑常开，而应作为明确的业务开关。

### `requestWebgl1` 只用于兼容兜底

demo 中也存在显式写 `requestWebgl1: true` 的场景，但这更适合兼容验证或特定环境兜底，不应成为新项目默认配置。

## Common Patterns

### 设备自适应策略

1. 默认 `renderBackend: "auto"`
2. 启动后读取 `renderBackendResolved`
3. 若是 `webgl1`，自动关闭 SSR/OIT、降低阴影与点云精度

### 大场景保守参数

-   3D Tiles `maximumScreenSpaceError` 提高到 `12~20`
-   控制 `maximumMemoryUsage`
-   降低同时显示的半透明对象

### 后处理链按需启用

`source/index.js` 公开了：

-   `PostProcessStageLibrary`
-   `PostProcessStage`
-   `PostProcessStageComposite`

而 `Map` 公开了 `map.postProcessStages`。

最稳妥的用法是优先用 `PostProcessStageLibrary` 产出官方组合阶段，再 add 到 `map.postProcessStages`：

```javascript
const stage = minemap.PostProcessStageLibrary.createNightVisionStage();
map.postProcessStages.add(stage);
```

自定义阶段或多阶段编排，再考虑 `PostProcessStage` / `PostProcessStageComposite`。

## Recommended Runtime Strategy

1. 默认 `renderBackend: "auto"`
2. 初始化后读取 `map.renderBackendResolved`
3. 若结果是 `webgl1`：

-   关闭或弱化阴影
-   关闭 SSR / OIT
-   收紧 3D Tiles 精度和内存
-   减少透明叠加与高频分析

4. 若结果是 `webgl2`：

-   再按场景逐步开启高成本能力

## Post-process Pipeline

适合启用后处理的前提：

-   当前后端至少稳定在 `webgl2` 或表现良好的 `webgl1`
-   主场景渲染已经达标，再叠加观感类效果

推荐顺序：

1. 先验证基础帧率
2. 再启用 `map.postProcessStages`
3. 优先使用 `PostProcessStageLibrary` 的内置 stage
4. 最后才进入自定义 shader stage 组合

不要把后处理当成默认模板常驻全开项。

## Performance Tips

-   首帧优先“可见”再“精致”：先关重特效，交互稳定后渐进开启
-   避免每帧触发对象创建和大数组分配
-   对 DEM/分析/拾取任务做节流
-   长时间运行场景定期清理无用 scene component 与 primitive

## Do Not Do This

-   不要把 `requestWebgl1: true` 当作默认模板
-   不要在大场景、阴影、分析、透明特效全开时再追求极低 `maximumScreenSpaceError`
-   不要在未读取后端解析结果前就写死整套高端渲染配置

## Demo References

-   `demo/html/DirectionalLight.html`
-   `demo/html/LightAndShadow.html`
-   `demo/html/SunLight.html`
-   `demo/html/editor-map.html`

## See Also

- [minemap-wind-field](../minemap-wind-field/SKILL.md)：浮点 GPU 计算能力与粒子预算。
- [minemap-framework-lifecycle](../minemap-framework-lifecycle/SKILL.md)：防止重复实例与定时器泄漏。

-   `minemap-scene-components`
-   `minemap-anti-aliasing`
-   `minemap-lighting-and-shadows`
-   `minemap-screen-space-reflection`
-   `minemap-water-surface-and-refraction`
-   `minemap-terrain-and-analysis`
-   `minemap-events-and-picking`
-   `minemap-post-process-and-render-pipeline`
