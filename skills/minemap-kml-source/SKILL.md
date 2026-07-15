---
name: minemap-kml-source
description: MineMap KML 数据源。用于加载 KML / KMZ 文件，解析 LineStyle / IconStyle / LabelStyle / PolyStyle 等样式继承，并把 KML 属性映射为 line / circle / fill / symbol 派生图层。
---

# MineMap KML Source

## 概述

KML（Keyhole Markup Language）是 OGC 标准的地理标记格式，常用于地铁线路、POI 集合、行政区标注等。

MineMap 4.25 公开支持把 KML 文件直接作为 `geojson`-style source：

-   解析纯 JS XML 解析器，无需依赖 `DOMParser`，因此也支持 Worker
-   自动把 `LineStyle` / `IconStyle` / `LabelStyle` / `PolyStyle` 映射到 feature properties
-   `IconStyle` 里的图片资源会自动 `addImage` 进 sprite，引用名是 `icon-id`

入口：

```javascript
map.addSource("kml", {
	type: "kml",
	data: "../data/your.kml",
	maxzoom: 16
});
```

## Quick Start

```javascript
map.on("load", () => {
	map.addSource("kml-subway", {
		type: "kml",
		data: "../data/kml/2015beijingsubway.kml",
		maxzoom: 16
	});

	// LineString ← KML LineStyle
	map.addLayer({
		id: "kml-line",
		type: "line",
		source: "kml-subway",
		filter: ["==", "$type", "LineString"],
		paint: {
			"line-color": ["get", "stroke"],
			"line-width": ["get", "stroke-width"],
			"line-opacity": ["get", "stroke-opacity"]
		}
	});

	// Point 有 icon 时 ← KML IconStyle（图片已自动 addImage）
	map.addLayer({
		id: "kml-icon",
		type: "symbol",
		source: "kml-subway",
		filter: ["all", ["==", "$type", "Point"], ["has", "icon-id"]],
		layout: {
			"icon-image": ["get", "icon-id"],
			"icon-size": ["get", "icon-scale"],
			"icon-allow-overlap": true
		}
	});

	// Polygon ← KML PolyStyle
	map.addLayer({
		id: "kml-fill",
		type: "fill",
		source: "kml-subway",
		filter: ["==", "$type", "Polygon"],
		paint: {
			"fill-color": ["get", "fill"],
			"fill-opacity": ["get", "fill-opacity"]
		}
	});
});
```

## Architecture Positioning

-   KML 是 source 类型，**不是** layer 类型
-   内部解析完成后会被表达成 GeoJSON-like 的 `$type` 区分几何：
    -   `LineString`
    -   `Polygon`
    -   `Point`
-   KML 样式通过 properties 暴露，**style 层只需要读取 `get` 表达式即可**

## KML 样式到 properties 的映射

| KML 字段              | properties key              | 说明                  |
| --------------------- | --------------------------- | --------------------- |
| `LineStyle.color`     | `stroke`                    | 十六进制颜色          |
| `LineStyle.width`     | `stroke-width`              | 像素                  |
| `LineStyle.color.a`   | `stroke-opacity`            | 0..1                  |
| `PolyStyle.color`     | `fill`                      | 十六进制颜色          |
| `PolyStyle.color.a`   | `fill-opacity`              | 0..1                  |
| `IconStyle.Icon.href` | 自动 `addImage` + `icon-id` | 引擎自动注册到 sprite |
| `IconStyle.scale`     | `icon-scale`                | 图标缩放              |
| `LabelStyle.color`    | `text-color`                | 文字颜色              |
| `LabelStyle.color.a`  | `text-opacity`              | 0..1                  |
| `LabelStyle.scale`    | `text-size`                 | 文字大小              |
| `Placemark.name`      | `name`                      | 一般用于 text-field   |

## Common Patterns

### 模式 1：地铁线路（线 + 站点 + 站名）

对应 demo：`demo/html/KMLSource.html`

```javascript
// 线
map.addLayer({
	id: "kml-subway-line",
	type: "line",
	source: "kml-subway",
	filter: ["==", "$type", "LineString"],
	paint: {
		"line-color": ["get", "stroke"],
		"line-width": ["get", "stroke-width"],
		"line-opacity": ["get", "stroke-opacity"]
	}
});

// 站点（带图标）
map.addLayer({
	id: "kml-subway-icon",
	type: "symbol",
	source: "kml-subway",
	filter: ["all", ["==", "$type", "Point"], ["has", "icon-id"]],
	layout: {
		"icon-image": ["get", "icon-id"],
		"icon-size": ["get", "icon-scale"],
		"icon-allow-overlap": true
	}
});

// 站名
map.addLayer({
	id: "kml-subway-label",
	type: "symbol",
	source: "kml-subway",
	filter: ["==", "$type", "Point"],
	layout: {
		"text-field": ["get", "name"],
		"text-anchor": "top",
		"text-allow-overlap": false,
		"text-size": ["get", "text-size"]
	},
	minzoom: 11
});
```

### 模式 2：监听图标加载

KML 里的 icon 图片会异步注册到 sprite，可以监听 `kmlIconsLoaded` 事件：

```javascript
map.getSource("kml-subway").on("kmlIconsLoaded", (e) => {
	console.log(
		"registered icons:",
		e.icons.map((i) => i.id)
	);
});
```

### 模式 3：KML 与 `queryRenderedFeatures`

KML 解析后与 GeoJSON 行为一致，可以直接用 `queryRenderedFeatures` 做交互：

```javascript
map.on("click", (e) => {
	const features = map.queryRenderedFeatures(e.point, {
		layers: ["kml-subway-line", "kml-subway-icon"]
	});
	if (features.length) {
		console.log(features[0].properties);
	}
});
```

## Strict Constraints

### 1. KML 是 source，不是 layer

不要写 `addLayer({ type: 'kml' })`。必须 `addSource({ type: 'kml' })` + `addLayer` 用 `line` / `symbol` / `fill` / `circle` 派生。

### 2. 图标资源要等 `kmlIconsLoaded`

KML 里的 `IconStyle.Icon.href` 是异步下载并 `addImage` 的，**不要**在 KML 加载完立刻假设 sprite 已经有这个 id。要么监听 `kmlIconsLoaded`，要么用 `has-image` filter 兜底。

### 3. `data` 必须是可访问的 URL

`data` 字段接受字符串 URL；目前 KML source 不会把内嵌 KML 字符串直接解析，需要先把 KML 写成本地 / 远端文件。

### 4. 字段值是 KML 原始字符串或数字

例如 `stroke-width` 直接来自 KML `<width>` 节点（KML 默认单位是像素），不需要二次换算。

## Failure Cases

-   图标全不显示 → 没等 `kmlIconsLoaded`；或 KML 里的 `IconStyle.Icon.href` 指向了不可访问的 URL
-   线没有颜色 → KML 里没有 `LineStyle`，properties 里 `stroke` 为空，给 paint 加 fallback 默认值
-   Polygon 区域全部画在最上层挡住其他 layer → 检查 `fill-opacity` 是否被 KML `PolyStyle.color` 里的 alpha 覆盖
-   `text-field` 拿不到 `name` → KML 中 `Placemark` 没有 `<name>` 节点，或 `styleUrl` 引用了未定义的 Style

## See Also

-   `minemap-style-and-data`
-   `minemap-layer-system`
-   `minemap-events-and-picking`
