---
name: minemap-style-and-data
description: MineMap 样式规范（style v8）、source/layer 管理、图标资源、GeoJSON 更新与数据源组织。
---

# MineMap Style and Data

## Quick Start

```javascript
map.on("style.load", () => {
	map.addSource("poi", {
		type: "geojson",
		data: { type: "FeatureCollection", features: [] }
	});

	map.addLayer({
		id: "poi-circle",
		type: "circle",
		source: "poi",
		paint: {
			"circle-radius": 5,
			"circle-color": "#2d8cf0"
		}
	});
});
```

## Core Concepts

### 1) Style 版本与结构

MineMap 使用 style spec `version: 8`，核心字段：`sources`、`layers`、`sprite`、`glyphs`。

### 2) Source 类型

常用：

-   `vector`
-   `raster`
-   `geojson`
-   `image`

说明：历史上的 `3d-model` / `3d-tiles` source/layer 方式已不推荐，优先使用 `addSceneComponent()`。

### 3) Layer 基础操作

-   `addLayer(layer, before?)`
-   `removeLayer(id)`
-   `moveLayer(id, beforeId?)`
-   `setFilter(layerId, filter)`
-   `setLayoutProperty` / `setPaintProperty`

### 4) GeoJSON 聚合与批量合并

可用 `addSourceGroup(id, source)` 处理多份 GeoJSON 异步合并；引擎会先注入临时数据，避免 `addLayer` 时 source 不存在。

### 5) 图像资源

-   `loadImage(url, cb)` + `addImage(id, image, options)`
-   `updateImage(id, image)`
-   `hasImage(id)` / `removeImage(id)` / `listImages()`

### 6) WMTS、RTL 与辅助数据工具

`source/index.js` 暴露了几个容易漏掉、但确实属于正式开发面的数据辅助入口：

-   `WMTSCapabilities`
-   `optionsFromCapabilities(...)`
-   `setRTLTextPlugin(...)`
-   `getRTLTextPluginStatus()`
-   `minemap.util.getJSON(...)`
-   `minemap.util.getString(...)`

它们适合放在“数据接入层”，而不是散落在业务组件里。

## Common Patterns

### 样式加载后再添加数据

```javascript
map.on("style.load", () => {
	if (!map.getSource("lineSource")) {
		map.addSource("lineSource", { type: "geojson", data: geojson });
	}
	if (!map.getLayer("lineLayer")) {
		map.addLayer({ id: "lineLayer", type: "line", source: "lineSource" });
	}
});
```

### 无闪烁数据刷新

-   更新 GeoJSON source 数据（而不是 remove/add layer）
-   动态路况可按 source 刷新机制做定时更新

### WMTS 能力解析

```javascript
const parser = new minemap.WMTSCapabilities();
const capabilities = parser.read(xmlText);
const wmtsSource = minemap.optionsFromCapabilities(capabilities, {
	layer: "img",
	matrixSet: "EPSG:4326",
	format: "image/png"
});

map.addSource("wmts", wmtsSource);
```

适用场景：

-   第三方 WMTS 服务参数不想手写
-   需要从 capabilities 自动提取 tile matrix、style、format

### RTL 插件配置

```javascript
minemap.setRTLTextPlugin("https://example.com/rtl-text.js");
```

约束：

-   应在地图初始化前或非常早期调用
-   不要重复调用，源码里会直接抛错
-   `getRTLTextPluginStatus()` 只用于状态判定，不替代初始化逻辑

## Performance Tips

-   高频变化优先改 `paint/layout/filter`，少做整套 `setStyle`
-   大 GeoJSON 数据考虑切片、聚合、分级加载
-   `queryRenderedFeatures` 指定 `layers` 范围，减少遍历

## See Also

-   `minemap-fundamentals`
-   `minemap-anti-aliasing`
-   `minemap-layer-system`
-   `minemap-style-system`
-   `minemap-2d-3d-overlay-and-classification`
-   `minemap-water-surface-and-refraction`
-   `minemap-scene-components`
-   `minemap-events-and-picking`
