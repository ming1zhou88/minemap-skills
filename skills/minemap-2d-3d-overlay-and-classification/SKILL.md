---
name: minemap-2d-3d-overlay-and-classification
description: MineMap 二三维叠加规范。覆盖矢量贴地、贴模型、对象接受/拒绝贴合、symbol 自动赋高、以及 raster 深度测试与图层顺序控制。
---

# MineMap 2D/3D Overlay and Classification

## Quick Start

```javascript
map.on("load", () => {
	map.addSource("route", {
		type: "geojson",
		data: routeGeoJSON,
		adaptTerrain: true
	});

	map.addLayer({
		id: "route-line",
		type: "line",
		source: "route",
		paint: {
			"line-width": 6,
			"line-color": "#00ffff"
		}
	});

	map.addLayer(
		{
			id: "base-raster",
			type: "raster",
			source: "base-raster",
			paint: {
				"depth-test": true
			}
		},
		"route-line"
	);
});
```

## Core Concepts

### 1) 先区分“顺序叠加”与“空间贴合”

MineMap 里的二三维叠加不是单一机制，而是四类机制组合：

-   图层顺序：`before` / `moveLayer(...)`
-   矢量贴地：`source.adaptTerrain`
-   Primitive 贴地：`primitive.adaptTerrain`
-   分类控制：`classificationType`
-   symbol 自动赋高与遮挡：`auto-height`、`auto-height-offset`、`hiddenType`

先问清业务目标：

-   只是想让二维图层别被栅格压住 → 先处理 `before`
-   想让线面沿 DEM 起伏 → 用 `adaptTerrain`
-   想让对象拒绝或接受贴合 → 配 `classificationType`
-   想让 symbol 跟三维场景一起工作 → 用 `auto-height` / `hiddenType`
-   想让矢量强制压在模型之上 → 用 `FORCE_VECTOR_OVER_MODEL`

### 2) 推荐主路径：source 级矢量贴地

当前版本更推荐在数据源上开启：

-   `GeoJSONSource.adaptTerrain: true`
-   对大批量线面数据优先这样做

这条路径适合：

-   路线贴地
-   面域贴地
-   贴地热力、贴地图案、贴地 sprite 等 style-layer 方案

注意边界：源码注释把这条能力描述为**贴 DEM 地形**的主路径，不应把它理解成“自动贴 3D Tiles 或 SceneModel 表面”。

### 3) Primitive 贴地是另一条路

如果你不是走 style layer，而是走低层三维对象：

-   `PolylineGeometry`
-   `PolygonGeometry`
-   其他 `Primitive`

则用：

-   `primitive.adaptTerrain = true`

源码中 `Primitive` 在开启 `adaptTerrain` 时，会自动把 `classificationType` 处理到贴地所需状态。它更适合：

-   少量高精度业务几何
-   需要自己控制材质、几何结构、交互的对象

而不是大批量常规专题面/线渲染。

### 4) 接收侧与拒绝侧由 `classificationType` 控制

二三维叠加不只看发送方，还要看接收方。

关键语义：

-   `minemap.ClassificationType.NONE`
    -   常用于对象正常参与接收/显示，不主动拒绝分类影响
-   `minemap.ClassificationType.IGNORECLASSIFICATION`
    -   显式拒绝贴合或分类影响
-   `minemap.ClassificationType.FORCE_VECTOR_OVER_MODEL`
    -   强制让矢量压在模型之上
-   `minemap.ClassificationType.ALL`
    -   贴地/分类相关的全量模式；低层对象内部会用到

其中最容易误用的是：

-   `FORCE_VECTOR_OVER_MODEL` 解决的是“压模显示”
-   不是“真正贴附到模型表面”

### 5) symbol 叠加三维场景，核心不在 `adaptTerrain`

symbol 与 line/fill 的思路不同。

围绕三维场景叠加 symbol，常见组合是：

-   `layout["auto-height"] = true`
-   `layout["auto-height-offset"] = xxx`
-   `hiddenType: minemap.SymbolHiddenType.SCENE`
-   必要时叠加 `classificationType: minemap.ClassificationType.FORCE_VECTOR_OVER_MODEL`

适用场景：

-   车标、站点、设备图标需要随着场景高度抬升
-   既要参与场景遮挡，又要避免完全埋进模型

### 6) raster 不是背景板，深度与顺序会直接影响结果

MineMap demo 已明确暴露两个常见坑：

-   raster 默认顺序不对，会压住矢量
-   primitive 贴地时，底层 raster 若未开启 `"depth-test": true`，贴地结果会异常

因此二三维叠加里，raster 至少要检查：

-   是否通过 `before` 放到了正确位置
-   是否需要 `paint["depth-test"] = true`

### 7) 旧路径不要再当默认方案

历史上有过直接依赖 layer 级 `classificationType` 的贴地写法，但当前版本更稳妥的默认方案是：

1. style layer 体系优先 `source.adaptTerrain`
2. 低层几何优先 `primitive.adaptTerrain`
3. 是否接受/拒绝叠加，再看 receiver 的 `classificationType`
4. symbol 的高度和遮挡单独处理

## Demo-backed Patterns

### 模式 1：GeoJSON 线面贴 DEM

对应 demo：`demo/html/VectorMapAdaptTerrain.html`、`demo/html/VectorMapAdaptAll.html`

做法：

-   source 上 `adaptTerrain: true`
-   line/fill layer 正常渲染
-   必要时给 raster 指定 `before`

这是最推荐的二维专题贴地入口。

### 模式 2：发送方 / 接收方双向控制

对应 demo：`demo/html/VectorAdaptGroupExample.html`

该 demo 很关键，因为它把两侧角色拆清楚了：

-   发送方：GeoJSON source、Primitive line、Primitive polygon
-   接收方：SceneTileset、SceneModel、Primitive、Raster
-   接收方可通过 `classificationType` 在接受/拒绝之间切换

这说明 MineMap 的“叠加”本质上是一个 sender / receiver 机制，不只是画面层级问题。

### 模式 3：强制矢量压模型

对应 demo：`demo/html/SymtrackingOverModel.html`

做法：

-   对 line 或 symtracking 类表达设置
-   `classificationType: minemap.ClassificationType.FORCE_VECTOR_OVER_MODEL`

适用于：

-   轨迹必须清晰可见
-   标识线不能被模型遮没

但这不是贴地，更不是贴模型几何。

### 模式 4：symbol 自动赋高

对应 demo：`demo/html/SymbolAutoHeight.html`

组合特征：

-   `hiddenType: minemap.SymbolHiddenType.SCENE`
-   `auto-height: true`
-   `auto-height-offset`
-   可叠加 `FORCE_VECTOR_OVER_MODEL`

适合：

-   三维园区 POI
-   模型上方图标
-   跟随场景高度变化的点状标识

### 模式 5：非典型贴地 layer 也有 shipped demo

对应 demo：

-   `demo/html/SpriteAdaptTerrain.html`
-   `demo/html/HeatMapAdaptTerrain.html`
-   `demo/html/LinePatternAdaptTerrain.html`
-   `demo/html/FillPatternAdaptTerrain.html`

这说明 shipped demo 中确实存在 sprite、heatmap、pattern 参与贴地的组织方式。

但源码注释对 `GeoJSONSource.adaptTerrain` 的描述更保守，重点仍放在线/面主路径上。实际项目里应把：

-   line/fill 视为最稳基线
-   sprite/heatmap/pattern 视为“可参考 demo、但要逐场景验收”的扩展写法

## Recommended Workflow

1. 先判断表达层次：style layer 还是 primitive
2. 如果是大批量专题线面，优先 `source.adaptTerrain`
3. 如果是少量精细几何，优先 `primitive.adaptTerrain`
4. 再确定 receiver 是否要 `IGNORECLASSIFICATION`
5. 如果诉求是“永远别被模型挡住”，才用 `FORCE_VECTOR_OVER_MODEL`
6. symbol 另走 `auto-height` / `hiddenType`
7. 最后检查 raster 的 `before` 和 `depth-test`

## Strict Constraints

### 1. `source.adaptTerrain` 不等于贴任意三维对象

它的主语义是贴 DEM 地形。不要把它写成“自动贴合全部 3D Tiles / 模型表面”的通用方案。

### 2. `FORCE_VECTOR_OVER_MODEL` 不等于真实几何贴附

它解决显示优先级，不解决几何拟合。

### 3. 不要把 symbol 方案套到 line/fill

-   line/fill 主要看 `adaptTerrain`
-   symbol 主要看 `auto-height` / `hiddenType`

两套机制不能混成一个抽象概念。

### 4. raster 的顺序和深度测试必须显式检查

尤其是：

-   raster 在错误顺序时会遮住矢量
-   raster 缺少 `depth-test` 时会影响 primitive 贴地

### 5. 不要把旧版 layer-classification 贴地写法当默认模板

当前文档与 demo 的推荐方向，已经转向 source / primitive 两条主路径。

## Failure Cases

### 失败场景 1：只改 `before`，却以为已经实现贴地

`before` 只解决图层顺序，不解决空间贴合。

### 失败场景 2：对模型上方路线使用 `FORCE_VECTOR_OVER_MODEL`，却声称“贴模型成功”

这只是视觉压盖，不是沿模型表面拟合。

### 失败场景 3：symbol 仍按普通 2D 点图层处理

在 DEM、3D Tiles、SceneModel 混合场景里，不配 `auto-height` / `hiddenType`，往往会出现埋地、漂浮或遮挡错误。

### 失败场景 4：primitive 开启贴地，却忽略 raster `depth-test`

这会导致看起来“贴地失效”或结果不稳定。

## Common Patterns

### GeoJSON 贴地线

```javascript
map.addSource("route", {
	type: "geojson",
	data: routeGeoJSON,
	adaptTerrain: true
});

map.addLayer({
	id: "route-line",
	type: "line",
	source: "route",
	paint: {
		"line-width": 4,
		"line-color": "#00ffff"
	}
});
```

### 对象拒绝接受贴合

```javascript
map.addSceneComponent({
	id: "tileset-01",
	type: "3d-tiles",
	url: tilesetUrl,
	classificationType: minemap.ClassificationType.IGNORECLASSIFICATION
});
```

### symbol 自动赋高

```javascript
map.addLayer({
	id: "poi-symbol",
	type: "symbol",
	source: "poi-source",
	hiddenType: minemap.SymbolHiddenType.SCENE,
	layout: {
		"icon-image": "poi",
		"auto-height": true,
		"auto-height-offset": 20
	}
});
```

### 强制矢量压模型

```javascript
map.addLayer({
	id: "tracking-line",
	type: "line",
	source: "tracking-source",
	classificationType: minemap.ClassificationType.FORCE_VECTOR_OVER_MODEL,
	paint: {
		"line-width": 6,
		"line-color": "#ff5500"
	}
});
```

## Performance Discipline

-   大批量贴地线面优先 source 方案，不要全改成 primitive
-   `FORCE_VECTOR_OVER_MODEL` 只给关键图层，不要全图默认开启
-   symbol 自动赋高要控制图标数量与碰撞策略
-   DEM、3D Tiles、raster、贴地矢量同时存在时，先做分层验收，再做综合验收

## Demo References

-   `demo/html/VectorMapAdaptTerrain.html`
-   `demo/html/VectorMapAdaptAll.html`
-   `demo/html/VectorAdaptGroupExample.html`
-   `demo/html/SymtrackingOverModel.html`
-   `demo/html/SymbolAutoHeight.html`
-   `demo/html/SpriteAdaptTerrain.html`
-   `demo/html/HeatMapAdaptTerrain.html`
-   `demo/html/LinePatternAdaptTerrain.html`
-   `demo/html/FillPatternAdaptTerrain.html`

## See Also

-   `minemap-style-and-data`
-   `minemap-scene-components`
-   `minemap-primitives-and-materials`
-   `minemap-primitive-adapt-terrain`
-   `minemap-terrain-and-analysis`
-   `minemap-events-and-picking`
