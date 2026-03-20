---
name: minemap-lighting-and-shadows
description: MineMap 光照与阴影规范。覆盖 `enableLight()`、`enableShadow()`、太阳时间、灯光对象、以及 `castShadow` / `receiveShadow` 的工程边界。
---

# MineMap Lighting and Shadows

## Quick Start

```javascript
map.enableLight();
map.enableShadow();
map.setShadowType(minemap.ShadowType.SHADOW_MAP_PCF);

map.setSunLightTime({
	year: 2024,
	month: 6,
	day: 6,
	hours: 10,
	minutes: 0,
	seconds: 0
});
```

## Core Concepts

### 1) 全局入口在 `Map`

MineMap 光照与阴影的正式入口是：

-   `map.enableLight()` / `map.disableLight()`
-   `map.enableShadow()` / `map.disableShadow()`
-   `map.setShadowType(...)`
-   `map.setSunLightTime(...)`
-   `map.getLights()`
-   `map.addLight(light)` / `map.removeLight(light)`

### 2) 太阳光是场景默认主光，不等于全部光照系统

`map.getLights()` 返回的核心结构包括：

-   `sunLight`
-   `ambientLight`
-   `directionalLights`
-   `pointLights`
-   `spotLights`

因此常见工作流是：

1. 先设 `sunLight`
2. 再按需加 `DirectionalLight` / `PointLight` / `SpotLight`

### 3) 阴影至少要满足三层条件

要看到阴影，通常至少要同时满足：

1. 全局开启 `map.enableShadow()`
2. 光源本身 `castShadow = true`
3. 对象侧正确配置 `castShadow` / `receiveShadow`

缺一项都可能导致“开了阴影但看不到结果”。

### 4) 阴影是成本项，不是默认常驻项

从 demo 和已有性能 skill 看，阴影应视为高成本能力，特别是叠加：

-   大场景
-   多光源
-   SSR
-   分析
-   半透明材质

时要更保守。

## Demo-backed Patterns

### 模式 1：太阳光 + 全局阴影

对应 demo：`demo/html/LightAndShadow.html`、`demo/html/SunLight.html`

最稳妥的主路径是：

1. `map.enableLight()`
2. `map.enableShadow()`
3. `map.setShadowType(...)`
4. `map.setSunLightTime(...)`
5. 再调 `sunLight.intensity`、`sunLight.castShadow`、`sunLight.shadow.*`

### 模式 2：自定义灯光对象加入场景

对应 demo：`demo/html/DirectionalLight.html`、`demo/html/PointLight.html`、`demo/html/MovingPointLight.html`

这条路径验证了：

-   `DirectionalLight`
-   `PointLight`
-   `SpotLight`
-   `AmbientLight`

都属于正式开发面。

### 模式 3：灯光与对象同时建模

`LightAndShadow.html` 还说明了一个很关键的工程点：

-   灯光对象本身往往需要辅助几何或场景对象一起调试
-   尤其是点光源、聚光灯，最好配可视化锚点或目标逻辑

## Recommended Workflow

1. 先只开 `enableLight()`
2. 再启 `enableShadow()`
3. 优先使用 `sunLight`
4. 确认模型或 primitive 已设 `castShadow` / `receiveShadow`
5. 最后再叠加额外 `DirectionalLight` / `PointLight` / `SpotLight`

## Shadow Tuning

常见高频参数：

-   `shadow.bias`
-   `shadow.normalBias`
-   `shadow.radius`
-   `shadow.focus`（聚光灯）

经验边界：

-   `bias` 主要用于压阴影痤疮
-   `normalBias` 主要用于压表面自阴影伪影
-   `radius` 主要影响软化观感

## Strict Constraints

### 1. 不要混淆“有光照”和“有阴影”

`enableLight()` 只代表参与光照计算，阴影还需要单独开。

### 2. `setSunLightTime(...)` 优先于旧 `setSunLight(...)`

旧接口已被标成废弃，不应继续扩散。

### 3. 多光源不是默认推荐路径

多灯光会明显抬高调试和性能成本。大多数业务先把 `sunLight + ambientLight` 跑顺，再加补光。

### 4. 阴影问题先查对象标志，不要只调 `shadowBias`

很多“没阴影”的根因是对象没开 `castShadow` / `receiveShadow`。

## Common Failure Cases

-   开了 `enableShadow()`，但光源没 `castShadow`
-   模型没 `receiveShadow`
-   只顾调阴影参数，忘了先 `enableLight()`
-   多点光 / 聚光灯同时开阴影，直接把场景拖重
-   继续使用废弃 `setSunLight(...)`

## Performance Discipline

-   默认优先单主光源
-   阴影只给关键场景开
-   与 SSR、后处理、大范围分析不要无差别叠加
-   `ShadowType` 先从更稳的基础模式试，再考虑更柔和的 PCF

## Demo References

-   `demo/html/LightAndShadow.html`
-   `demo/html/DirectionalLight.html`
-   `demo/html/AmbientLight.html`
-   `demo/html/PointLight.html`
-   `demo/html/MovingPointLight.html`
-   `demo/html/SunLight.html`

## See Also

-   `minemap-material-system-and-shading`
-   `minemap-primitives-and-materials`
-   `minemap-performance-and-backend`
