---
name: minemap-style-and-data
description: MineMap 样式规范（style v8）、source/layer 管理、图标资源、GeoJSON 更新与数据源组织，重点补全 VectorTileSource 参数与约束。
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

MineMap 使用 style spec `version: 8`，核心字段仍然是：

-   `sources`
-   `layers`
-   `sprite`
-   `glyphs`

### 2) 常用 source 类型

-   `vector`
-   `raster`
-   `geojson`
-   `image`

补充结论：

-   历史上的 `3d-model` / `3d-tiles` source/layer 方式已不推荐
-   新三维对象统一优先 `map.addSceneComponent()`

### 3) Layer 基础操作

-   `addLayer(layer, before?)`
-   `removeLayer(id)`
-   `moveLayer(id, beforeId?)`
-   `setFilter(layerId, filter)`
-   `setLayoutProperty()` / `setPaintProperty()`

### 4) GeoJSON 聚合与批量合并

可用 `addSourceGroup(id, source)` 处理多份 GeoJSON 异步合并；引擎会先注入临时数据，避免 `addLayer` 时 source 不存在。

### 5) 图像资源

-   `loadImage(url, cb)` + `addImage(id, image, options)`
-   `loadImages([{ id, url, imageOptions? }], cb, options)`
-   `updateImage(id, image)`
-   `hasImage(id)` / `removeImage(id)` / `listImages()`

最近版本补充结论：

-   `loadImages` 当前主路径是描述符数组，不再推荐只传 `string[]`
-   描述符模式下会在内部自动执行 `addImage(...)`
-   单项结果按数组返回，每项包含 `id`、`url`、`isOk`、`image/error`
-   每个图片项可带 `imageOptions`，会透传到内部 `addImage(...)`

兼容说明：

-   `string[]` 仍兼容，但已进入弃用兼容模式
-   兼容模式下不会自动 `addImage(...)`
-   回调第二个参数是以原始 URL 为 key 的图片对象，而不是新数组结果结构

### 6) WMTS、RTL 与辅助数据工具

`source/index.js` 暴露了几类容易漏掉、但确实属于正式开发面的数据入口：

-   `WMTSCapabilities`
-   `optionsFromCapabilities(...)`
-   `setRTLTextPlugin(...)`
-   `getRTLTextPluginStatus()`
-   `minemap.util.getJSON(...)`
-   `minemap.util.getString(...)`

这些更适合放在“数据接入层”。

## VectorTileSource 深入说明

这一节以 `source/data-source/source-cache/VectorTileSource.js` 为准，不按旧博客或旧版 Mapbox 经验脑补。

### 推荐定位

`VectorTileSource` 是 MineMap 里最重要的业务 source 之一，典型用于：

-   行政区面
-   路网线
-   建筑拉伸
-   POI / 标注
-   大体量贴地线面

### 创建方式

```javascript
map.addSource("road-source", {
	type: "vector",
	tiles: ["https://example.com/mvt/{z}/{x}/{y}.pbf"],
	minzoom: 1,
	maxzoom: 20
});
```

### 参数总表

#### 必备或常见字段

-   `type: 'vector'`
-   `url`
-   `tiles`
-   `scheme`
-   `projection`
-   `attribution`
-   `tileSize`
-   `minzoom`
-   `maxzoom`
-   `bounds`
-   `adcode`
-   `promoteId`
-   `adaptTerrain`

### `url` 和 `tiles`

两者二选一即可：

-   `url`：通过 TileJSON 地址间接拿瓦片描述
-   `tiles`：直接给瓦片模板 URL 数组

业务上更常见的是直接给 `tiles`。

### `scheme`

注释给的标准值：

-   `xyz`
-   `tms`

默认是 `xyz`。

### `projection`

注释写法是字符串，如 `MERCATOR`；源码里实际走投影类型判断。

实际建议：

-   非必要不要乱改
-   除非明确知道服务端输出不是常规墨卡托瓦片

### `tileSize`

这是最容易被写错的点。

源码构造器里直接做了硬校验：

-   `vector tile sources must have a tileSize of 512`

也就是说：

-   `VectorTileSource` 的 `tileSize` 必须是 `512`
-   不是“推荐 512”，而是“不是 512 就抛错”

### `minzoom` / `maxzoom`

源码真实默认值来自 `LayerZoomConfig.VECTOR_DEFAULT_ZOOM`：

-   `minzoom = 0`
-   `maxzoom = 20`

这点比一些注释旧文本更重要，因为运行时实际就是这么取默认值。

使用建议：

-   如果你的服务有 20 级以上更细数据，必须主动把 source 的 `maxzoom` 往上提
-   否则 layer 再设更大 `maxzoom` 也只是继续放大旧层级

### `bounds`

格式：

-   `[west, south, east, north]`

作用：

-   限制 source 的有效空间范围
-   避免在全球范围无意义发请求

### `adcode`

这是 MineMap 自己扩的重点参数。

含义：

-   按行政区划编码过滤数据

源码还实现了 `adcode` 的 setter：

-   修改后会触发 source reload

因此它不是静态元数据，而是可以作为运行时切区工具使用。

前提：

-   服务端必须真的支持按行政区划编码返回数据

### `promoteId`

作用：

-   指定哪个属性作为 feature id，用于 feature-state 之类的能力

可写法：

-   直接字符串：对所有 source-layer 统一用同一字段
-   对象：按 source-layer 分别指定字段

这是做 `setFeatureState()`、交互高亮、运行时状态染色时的重要前置。

### `adaptTerrain`

这是 MineMap 的重点扩展能力。

作用：

-   让 vector source 级别的数据贴合 DEM 地形

源码注释明确写了边界：

-   在 source 上设置后，layer 上无需再设
-   这样只能贴合 terrain / DEM
-   不能贴合 3D Tiles 和模型
-   当前主要适用于 `LineStyleLayer`、`FillStyleLayer`

这点很关键：

-   贴地 terrain 和贴 3D Tiles / 模型不是一回事
-   不要把 `adaptTerrain` 当成“万能贴所有三维表面”

### `clippingPlanes`

源码里 `VectorTileSource` 也挂了 `clippingPlanes` 字段，但这不是常规对外主能力，技能库不把它当作常规 vector source 首选参数。

### 推荐写法

```javascript
map.addSource("road-source", {
	type: "vector",
	tiles: ["https://example.com/mvt/{z}/{x}/{y}.pbf"],
	tileSize: 512,
	minzoom: 1,
	maxzoom: 20,
	bounds: [112, 28, 114, 30],
	promoteId: "id"
});
```

### 贴地推荐写法

```javascript
map.addSource("line-source", {
	type: "vector",
	tiles: ["https://example.com/mvt/{z}/{x}/{y}.pbf"],
	tileSize: 512,
	adaptTerrain: true,
	minzoom: 1,
	maxzoom: 20
});
```

### 什么时候优先 vector，而不是 GeoJSON

优先 `vector` 的典型场景：

-   数据量大
-   需要分层级加载
-   同一数据有多个 `source-layer`
-   想把贴地、大范围路网、行政区、建筑分层统一进瓦片服务

优先 `geojson` 的典型场景：

-   小规模临时数据
-   前端实时生成数据
-   只做局部交互实验

## Common Patterns

## Common Patterns

### `loadImages` 新主路径

```javascript
map.loadImages(
	[
		{ id: "cat", url: "https://example.com/cat.png", imageOptions: { pixelRatio: 2 } },
		{ id: "dog", url: "https://example.com/dog.png" }
	],
	(error, result) => {
		if (error) console.error(error);
		console.log(result);
	},
	{
		onProgress: (loaded, total) => console.log(loaded, total)
	}
);
```

这条路径的优点是：

-   加载完成后自动 `addImage(...)`
-   图片 id 明确，不再依赖 URL 推导
-   单项失败不会把成功项的结果结构抹平

### `loadImages` 兼容旧写法时要知道的边界

```javascript
const legacyUrls = ["https://example.com/cat.png"];
map.loadImages(legacyUrls, (error, images) => {
	if (!error) {
		map.addImage("cat", images[legacyUrls[0]]);
	}
});
```

这是兼容路径，不是当前推荐路径。

### `addSource` / `addLayer` 先后添加的当前行为

最近 master 有专门修复 `addSource`、`addLayer` 场景下“不加载”的问题，但该修复主要落在历史 `3d-model` source/layer 路径。

对 style/source/layer 的常规结论是：

-   当前版本 `map.addSource(...)` 会先执行 `_lazyInitEmptyStyle()`
-   然后 `style.addSource(...)`
-   最后 `_update(true)`

因此在空样式、晚绑定样式或运行时动态加 source/layer 的场景里，现版本比旧版本更稳。

但推荐顺序仍然不变：

-   常规二维图层尽量放在 `style.load` 后添加
-   不要因为修复了历史问题，就把所有时序都写成“随便什么时候加都行”

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

对于 vector source，如果服务端数据更新而 source id 不变，优先触发 reload 或切换服务参数，不要反复销毁整套 layer/source。

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
-   大规模业务数据优先 `VectorTileSource`，不要把所有内容一次性灌进 GeoJSON
-   `VectorTileSource` 的 `bounds`、`minzoom`、`maxzoom` 要收紧
-   `promoteId` 提前设计好，避免后期 feature-state 很难补
-   `queryRenderedFeatures` 指定 `layers` 范围，减少遍历
-   `loadImages` 批量图标场景优先用 `maxConcurrent`、`timeout`、`retryCount` 做受控加载，不要手搓一层并发队列

## See Also

-   `minemap-fundamentals`
-   `minemap-anti-aliasing`
-   `minemap-layer-system`
-   `minemap-style-system`
-   `minemap-2d-3d-overlay-and-classification`
-   `minemap-water-surface-and-refraction`
-   `minemap-scene-components`
-   `minemap-events-and-picking`
