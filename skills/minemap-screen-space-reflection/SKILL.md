---
name: minemap-screen-space-reflection
description: MineMap 屏幕空间反射（SSR）规范。覆盖 `map.SSR` 开关、运行时调参边界、与水面材质和反射对象的关系。
---

# MineMap Screen Space Reflection

## Quick Start

```javascript
const map = new minemap.Map({
	container: "map",
	style,
	SSR: true
});

map.on("load", () => {
	map.SSR = true;
});
```

## Core Concepts

### 1) 公开入口只有 `map.SSR`

对业务代码来说，SSR 的正式开关是：

-   构造参数 `SSR: true | false`
-   运行时属性 `map.SSR = true | false`

这是当前最稳定的开发入口。

### 2) `map.painter.ssrPass` 属于 demo 可见但不宜泛化的内部调参面

demo 确实直接改了：

-   `map.painter.ssrPass.ssrOpacity`
-   `map.painter.ssrPass.ssrMaxDistance`
-   `map.painter.ssrPass.ssrThickness`
-   `map.painter.ssrPass.ssrDelta`
-   `map.painter.ssrPass.ssrSigma`
-   `map.painter.ssrPass.ssrStepSize`

但这类写法应视为：

-   已被 demo 证明可用
-   但更接近调优接口，而不是通用业务第一入口

因此推荐顺序：

1. 先只用 `map.SSR`
2. 只有在明确要做视觉调优时，才少量使用 `map.painter.ssrPass.*`

### 3) SSR 不是后处理库 stage

MineMap 里的 SSR 不是 `map.postProcessStages` 家族的一员。

它更接近渲染主链中的一个高成本反射 pass，与：

-   水面材质
-   模型 / primitive 的反射表现
-   当前主场景深度信息

直接耦合。

### 4) SSR 是高成本特效，不应默认常开

现有源码和性能 skill 的结论一致：SSR 与阴影、OIT、分析叠加后，成本会明显上升。

因此它更适合：

-   高端设备
-   关键展示模式
-   局部场景验收后再常驻

## Demo-backed Patterns

### 模式 1：模型场景显式开启 SSR

对应 demo：`demo/html/GltfAnimationsTestSSR.html`

它验证了两件事：

1. 构造时可直接 `SSR: true`
2. 运行时可切换 `map.SSR`

这是当前最干净的 SSR 入口。

### 模式 2：水面场景联动调 SSR 参数

对应 demo：`demo/html/WaterPrimitive.html`

该 demo 说明：

-   SSR 常与 `WaterMaterial` 一起调
-   水面视觉效果往往需要和 `ssrOpacity`、`ssrThickness`、`ssrMaxDistance` 联动验收

这说明 MineMap 里 SSR 的主要高价值场景之一就是水体反射。

## Recommended Workflow

1. 先确认当前场景是否真的需要反射增强
2. 默认保持 `map.SSR = false`
3. 只在高质量模式里开启 `map.SSR = true`
4. 若视觉仍不理想，再微调 `map.painter.ssrPass.*`
5. 若后端或设备较弱，则优先关闭 SSR，而不是硬调参数

## Strict Constraints

### 1. 先开关，后调参

不要一上来就暴露一堆 `ssrPass` 滑杆。先验证 `map.SSR` 打开后值不值得保留。

### 2. 不要把 `map.painter` 当常规公开 API 扩散

只有在 demo 已明确这样组织的 SSR 调优场景里，才接触 `map.painter.ssrPass`。

### 3. WebGL1 或弱设备要优先关闭

这条和性能 skill 一致。SSR 是优先级很靠后的观感增强项，不应与兼容兜底冲突。

### 4. SSR 不替代环境贴图或真实镜面材质策略

它解决的是屏幕空间反射增强，不是完整物理反射系统。

## Common Tuning Parameters

### `ssrOpacity`

-   控制 SSR 贡献度
-   适合先从较低值往上试

### `ssrMaxDistance`

-   控制反射追踪距离
-   值越大通常越重，不应盲目拉高

### `ssrThickness`

-   影响相交容差
-   过小容易断裂，过大容易污染

### `ssrDelta` / `ssrSigma` / `ssrStepSize`

-   更偏滤波与步进细调
-   只应在已有画面对比基线后再调

## Failure Cases

-   在低端设备默认常开 SSR
-   把 `map.painter.ssrPass` 当稳定业务 API 大范围封装
-   SSR、阴影、分析、透明叠加同时全开
-   把水面反射问题全部归咎于 `ssrOpacity`，却忽略水面材质和允许反射对象配置

## Performance Discipline

-   SSR 作为高成本视觉增强项，默认后置考虑
-   先确认 `renderBackendResolved` 再决定是否保留
-   与阴影、OIT、复杂分析不要无差别同时常开

## Demo References

-   `demo/html/GltfAnimationsTestSSR.html`
-   `demo/html/WaterPrimitive.html`

## See Also

-   `minemap-performance-and-backend`
-   `minemap-water-surface-and-refraction`
-   `minemap-primitives-and-materials`
