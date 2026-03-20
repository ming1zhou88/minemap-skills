---
name: minemap-water-surface-and-refraction
description: MineMap 水面效果规范。覆盖 `WaterMaterial`、fill 水面图层、`waterRenderingMode`、以及对象是否允许水面折射/反射的控制。
---

# MineMap Water Surface and Refraction

## Quick Start

```javascript
const map = new minemap.Map({
	container: "map",
	style,
	waterRenderingMode: "normal"
});

const water = new minemap.Primitive({
	geometry: new minemap.Geometries.PolygonGeometry({
		polygonHierarchy: {
			positions: minemap.Math.Vector3.fromDegreesArrayHeights([
				116.39, 39.9, 10, 116.4, 39.9, 10, 116.4, 39.91, 10, 116.39, 39.91, 10
			])
		},
		perPositionHeight: true
	}),
	material: new minemap.WaterMaterial({
		depth: 256,
		reflectivity: 1.0,
		flowSpeed: 5,
		flowScale: 8
	})
});

map.addPrimitive(water);
```

## Core Concepts

### 1) 水面有两条正式开发路径

MineMap 当前可验证的水面能力主要分两类：

-   `WaterMaterial` + `Primitive`
-   `fill` 图层 + `paint["fill-water"] = "water"`

前者更适合：

-   自定义水面几何
-   精细材质调节
-   低层三维对象场景

后者更适合：

-   面状矢量水域
-   仍想保留 style/source/layer 工作流

### 2) 全局模式用 `map.waterRenderingMode`

公开入口：

-   构造参数 `waterRenderingMode: "normal" | "simplified"`
-   运行时属性 `map.waterRenderingMode`

语义：

-   `normal`
    -   正常渲染
    -   包含反射、折射、SSR、散射等完整效果
-   `simplified`
    -   精简渲染
    -   关闭一部分复杂特性，优先稳性能

### 3) 水面效果不只看水本身，还看谁允许被折射/反射

MineMap 在多个对象层面都暴露了：

-   `allowWaterRefract`
-   `allowWaterReflect`

这些控制可作用于：

-   `Material`
-   `Primitive` 使用的材质
-   `SceneModel`
-   `SceneTileset`
-   style layer

因此“水面看起来不对”常常不是 `WaterMaterial` 一个对象的问题，而是参与反射/折射的对象配置不对。

### 4) style layer 水面不是 `WaterMaterial` 的简单别名

`fill` 图层路径使用的是 style spec：

-   `paint["fill-water"] = "water"`
-   `fill-water-texture`
-   `fill-water-speed`
-   `fill-water-wave-size`
-   顶层图层参数中的 `depth`
-   `reflectivity`
-   `flowSpeed`
-   `flowScale`
-   `waterAbsorptionCoeff`
-   `heightOffset`

这是一条专门为矢量面水域准备的能力，不应强行改写成 primitive-only 模式。

## Demo-backed Patterns

### 模式 1：Primitive 水面材质

对应 demo：`demo/html/WaterPrimitive.html`

特点：

-   用 `PolygonGeometry` 定义水面
-   用 `WaterMaterial` 承载法线、流动、散射、反射参数
-   与 terrain、raster、模型共同验收

这是低层水面开发的主路径。

### 模式 2：Primitive 水面 + 对象折射/反射控制

对应 demo：`demo/html/WaterPrimitiveControl.html`

它验证了：

-   primitive 的材质可切 `allowWaterRefract`
-   `SceneModel` 可切 `allowWaterRefract`
-   水体效果不是只调水本身，还要控制哪些对象参与折射/反射

### 模式 3：fill 图层水域

对应 demo：`demo/html/WaterFillLayer.html`

做法：

-   `type: "fill"`
-   `paint["fill-water"] = "water"`
-   配合 `depth`、`reflectivity`、`flowSpeed`、`flowScale`、`waterAbsorptionCoeff`、`heightOffset`

同时 demo 还说明：

-   其他 line / symbol 图层可用 `allowWaterRefract: false` 避免被水面折射干扰

## `WaterMaterial` Recommended Parameters

常用参数：

-   `depth`
-   `reflectivity`
-   `normalMapA`
-   `normalMapB`
-   `flowMap`
-   `flowDirection`
-   `flowSpeed`
-   `flowScale`
-   `fresnelPower`
-   `fresnelScale`
-   `waterAbsorptionCoeff`
-   `scatteringScale`
-   `shallowScatterColor`
-   `deepScatterColor`

推荐顺序：

1. 先只调 `depth`、`reflectivity`、`flowSpeed`、`flowScale`
2. 再调 `waterAbsorptionCoeff`
3. 最后才微调法线、散射、菲涅尔

## Strict Constraints

### 1. `waterRenderingMode` 只有两个合法值

-   `normal`
-   `simplified`

其他值源码会直接抛错。

### 2. fill 水面和 primitive 水面不要混成同一抽象

-   矢量水域优先 fill 水面
-   复杂三维几何水面优先 `WaterMaterial`

### 3. 对象默认是否参与水面效果并不一致

源码里不同对象的默认值并不完全相同，不能想当然：

-   `SceneModel`：`allowWaterRefract` 默认 `false`，`allowWaterReflect` 默认 `false`
-   `SceneTileset`：`allowWaterRefract` 默认 `false`，`allowWaterReflect` 默认 `true`
-   普通非 raster/style layer 默认通常不参与

因此必须按对象显式验收。

### 4. `simplified` 模式是性能兜底，不是视觉优化工具

它的目标是降复杂度，不是让效果更“高级”。

### 5. 水面效果强依赖深度与场景组织

terrain、raster、模型、矢量层顺序不合理时，先排查场景组织，不要只盯材质参数。

## Common Patterns

### 设置全局水面模式

```javascript
map.waterRenderingMode = "simplified";
```

### fill 图层水域

```javascript
map.addLayer({
	id: "water-fill",
	type: "fill",
	source: "water-source",
	paint: {
		"fill-color": "rgb(64, 164, 223)",
		"fill-opacity": 1,
		"fill-water": "water"
	},
	depth: 50,
	reflectivity: 1,
	flowSpeed: 20,
	flowScale: 2,
	waterAbsorptionCoeff: [0.3, 0.075, 0.03],
	heightOffset: 3
});
```

### 模型不参与水面折射

```javascript
map.addSceneComponent({
	id: "harbor",
	type: "3d-model",
	data: "../data/test-gltf/harbor.glb",
	allowWaterRefract: false,
	allowWaterReflect: true
});
```

### 普通图层不被水面折射干扰

```javascript
map.addLayer({
	id: "route-line",
	type: "line",
	source: "route",
	allowWaterRefract: false
});
```

## Failure Cases

-   把所有对象都默认设成允许水面反射/折射
-   只调 `WaterMaterial`，却忽略 scene component 或图层的 `allowWaterRefract` / `allowWaterReflect`
-   用 `fill-water` 做复杂三维水面几何
-   在弱设备仍坚持 `waterRenderingMode = "normal"` + SSR 全开
-   把 `heightOffset` 当作通用空间贴合方案使用

## Performance Discipline

-   默认先用 `simplified` 验证性能基线
-   只有在展示模式里再切 `normal`
-   SSR 与高质量水面通常联动抬成本，要一起评估
-   法线图、流动图优先复用，不要在高频回调里重复装载

## Demo References

-   `demo/html/WaterPrimitive.html`
-   `demo/html/WaterPrimitiveControl.html`
-   `demo/html/WaterFillLayer.html`

## See Also

-   `minemap-primitives-and-materials`
-   `minemap-style-and-data`
-   `minemap-scene-components`
-   `minemap-screen-space-reflection`
-   `minemap-performance-and-backend`
