---
name: minemap-heatmap
description: MineMap 热力图（heatmap）专题。用于二维 / 三维热力图、贴地热力、聚合权重、热力色阶与 3D 显示模式，重点说明 paint/layout 参数边界与典型失败场景。
---

# MineMap HeatMap

## 概述

热力图在 MineMap 里走的是 **style-layer 路径**，类型是 `heatmap`，不是 Primitive。

要点：

-   数据源 `type` 推荐 `"geojson"`，每个 Feature 是个点
-   字段 `mag` 是 demo 默认的权重字段名
-   2D / 3D 模式通过 `layout.display-mode` 切换：`"2d"`（默认）/ `"3d"`
-   3D 模式下需要 `paint.heatmap-height` / `paint.heatmap-height-factor`
-   贴地热力（沿 DEM 起伏）通过 source 的 `adaptTerrain: true` 实现

## Quick Start：二维热力图

对应 demo：`demo/html/HeatMap.html`

```javascript
map.on("load", function () {
	map.addSource("IMAGERY", {
		type: "raster",
		tiles: ["https://.../satellite/{z}/{r}/{c}/{z}_{x}_{y}.jpg"],
		tileSize: 512
	});
	map.addLayer({
		id: "IMAGERY",
		type: "raster",
		source: "IMAGERY",
		minzoom: 0,
		maxzoom: 23
	});

	map.addSource("heatmapSource", {
		type: "geojson",
		data: "../data/heatmap_point.json" // 业务路径
	});

	map.addLayer({
		id: "heatmapLayer",
		type: "heatmap",
		source: "heatmapSource",
		layout: { visibility: "visible" },
		paint: {
			"heatmap-radius": 50,
			"heatmap-weight": {
				property: "mag",
				stops: [
					[0, 0],
					[10, 1]
				]
			},
			"heatmap-intensity": 1,
			"heatmap-color": [
				"interpolate",
				["linear"],
				["heatmap-density"],
				0,
				"rgba(0,0,255,0)",
				0.1,
				"royalblue",
				0.3,
				"cyan",
				0.5,
				"lime",
				0.7,
				"yellow",
				1,
				"red"
			],
			"heatmap-opacity": 1
		}
	});
});
```

## Quick Start：三维热力图

对应 demo：`demo/html/HeatMap3D.html`

3D 模式核心差异：

-   `layout.display-mode = "3d"`
-   `paint.heatmap-height` / `paint.heatmap-height-factor` 控制每根热力柱的高度

```javascript
map.addLayer({
	id: "heatmapLayer",
	type: "heatmap",
	source: "heatmapSource",
	layout: {
		visibility: "visible",
		"display-mode": "3d" // 默认 "2d"
	},
	paint: {
		"heatmap-radius": 100,
		"heatmap-weight": {
			property: "mag",
			stops: [
				[0, 0],
				[10, 1]
			]
		},
		"heatmap-intensity": 1,
		"heatmap-color": [
			"interpolate",
			["linear"],
			["heatmap-density"],
			0,
			"rgba(0,0,255,0)",
			0.1,
			"royalblue",
			0.3,
			"cyan",
			0.5,
			"lime",
			0.7,
			"yellow",
			1,
			"red"
		],
		"heatmap-opacity": 1,
		"heatmap-height": 200, // 每根热力柱的目标高度（米）
		"heatmap-height-factor": 1 // 0..1 高度权重因子
	}
});
```

## Quick Start：贴地热力（沿 DEM 起伏）

对应 demo：`demo/html/HeatMapAdaptTerrain.html`、`demo/html/HeatmapAdaptTerrain1.html`

要让热力柱沿地形起伏，需要：

1. 加载地形（`map.setTerrain`）
2. 数据源上声明 `adaptTerrain: true`

```javascript
map.setTerrain({
	tiles: ["https://.../dem/{z}/{r}/{c}/{z}_{x}_{y}.terrain"],
	availabilities: ["https://.../dem/layer.json"],
	tileSize: 512
});

map.addSource("heatmapSource", {
	type: "geojson",
	data: "../data/xi_an_heatmap.json",
	adaptTerrain: true
});

map.addLayer({
	id: "heatmapLayer",
	type: "heatmap",
	source: "heatmapSource",
	layout: { visibility: "visible" },
	paint: {
		"heatmap-radius": 20,
		"heatmap-weight": 1,
		"heatmap-intensity": 1,
		"heatmap-color": [
			"interpolate",
			["linear"],
			["heatmap-density"],
			0,
			"rgba(0,0,255,0)",
			0.1,
			"#36D56B",
			0.3,
			"#F8EA52",
			0.8,
			"#F6663A",
			1,
			"#B21710"
		],
		"heatmap-opacity": 1
	}
});
```

## 公开参数

### `layout`

| 参数           | 说明                                          |
| -------------- | --------------------------------------------- |
| `visibility`   | `"visible"` / `"none"`                        |
| `display-mode` | `"2d"`（默认）/ `"3d"`；3D 模式需配合高度参数 |

### `paint`

| 参数                    | 说明                                                   |
| ----------------------- | ------------------------------------------------------ |
| `heatmap-radius`        | 单点模糊半径（像素），默认 30，>= 1                    |
| `heatmap-weight`        | 单点权重，可以是数字或 `{ property, stops }` 插值      |
| `heatmap-intensity`     | 全局强度缩放，默认 1                                   |
| `heatmap-color`         | 颜色阶梯表达式（`interpolate` over `heatmap-density`） |
| `heatmap-opacity`       | 不透明度，默认 1，0..1                                 |
| `heatmap-height`        | 3D 模式柱高（米），`display-mode = "3d"` 时生效        |
| `heatmap-height-factor` | 3D 模式高度权重，0..1                                  |

> 业务上修改 `heatmap-color` 走 `map.setPaintProperty("heatmapLayer", "heatmap-color", expr)`，源码会重建 color ramp。

## Architecture Positioning

-   热力图 = `addLayer({ type: "heatmap" })`，**不是** Primitive，也**不是**独立类
-   数据源 `type` 推荐 `"geojson"`，demo 也用 `vector`，但 `vector` 上跑 `heatmap-weight.property` 时要保证字段在 vector tile 里
-   贴地逻辑在 source 上声明，不在 layer 上
-   颜色阶梯用 `interpolate(["linear"], ["heatmap-density"], ...)`，没有其它表达式更稳定

## Common Patterns

### 模式 1：按字段权重（人口 / 流量 / 评分）

```javascript
"heatmap-weight": { property: "mag", stops: [[0, 0], [10, 1]] }
```

字段名 `mag` 跟数据保持一致；切换权重字段要同步改 source。

### 模式 2：3D 模式 + 地形起伏

```javascript
// source 上开 adaptTerrain
// layer 开 display-mode = "3d"
// paint 同时给 height / height-factor
```

业务上常见的"流量柱状图沿地形起伏"就是这套组合。

### 模式 3：动态改色阶

```javascript
map.setPaintProperty("heatmapLayer", "heatmap-color", [
	"interpolate",
	["linear"],
	["heatmap-density"],
	0,
	"rgba(33,102,172,0)",
	0.2,
	"rgb(103,169,207)",
	0.4,
	"rgb(209,229,240)",
	0.6,
	"rgb(253,219,199)",
	0.8,
	"rgb(239,138,98)",
	1,
	"rgb(178,24,43)"
]);
```

参考 `HeatMap.html` 的 `changeColor()` 写法。

## Strict Constraints

### 1. 走 `addLayer`，不要 new 类

热力图是 style-layer，**没有** `minemap.HeatmapLayer` 之类的类；不要 new。

### 2. 2D / 3D 模式必须显式声明

`display-mode` 默认是 `"2d"`。要柱状 3D 必须写 `"3d"`，并配 `heatmap-height` / `heatmap-height-factor`，否则会"看起来是 2D 但没颜色"或"3D 高度为 0"。

### 3. 贴地必须 `setTerrain` + `adaptTerrain`

`adaptTerrain: true` 单独写在 source 上不会自动加载地形；少了 `setTerrain` 就会贴在 WGS84 椭球面上，**不是**用户想要的 DEM 起伏。

### 4. `heatmap-color` 表达式必须用 `heatmap-density`

字段是 `heatmap-density`，不是 `density` / `value`。改字段名会变成"颜色全黑"。

### 5. `heatmap-weight.property` 字段必须存在

字段在 data 里没有时，全部点权重都是默认 1；不会出现显式错误，但视觉上分不出强弱。

## Failure Cases

-   热力图完全看不到 → 检查 source 数据是否有点；检查 `visibility`、`opacity`
-   全黑 / 全透明 → 颜色阶梯里 `heatmap-density = 0` 的颜色透明度被吃掉了
-   3D 模式没柱高 → 没配 `heatmap-height` 或 `display-mode` 仍是 `"2d"`
-   贴地但没起伏 → 没 `setTerrain` 或地形瓦片地址 404
-   改色后没生效 → 改的是 `paint-property` 但属性名拼错（注意是 `heatmap-color`）

## Demo References

-   `demo/html/HeatMap.html`：二维热力图 + 改色
-   `demo/html/HeatMap3D.html`：三维热力图（`display-mode = "3d"` + `heatmap-height`）
-   `demo/html/HeatMapAdaptTerrain.html`：贴地热力（`setTerrain` + `adaptTerrain: true`）
-   `demo/html/HeatmapAdaptTerrain1.html`：贴地热力的另一种数据 / 配色变体

## See Also

-   `minemap-layer-system`：layer 类型总览，heatmap 在其中的位置
-   `minemap-style-and-data`：style / source / layer 数据工作流
-   `minemap-terrain-and-analysis`：`setTerrain` 与 DEM 查询
