---
name: minemap-wms-source
description: MineMap WMS 栅格数据源。用于通过 capabilities URL 自动构建 WMS 数据源、手工配置 GetMap 参数、瓦片调度、参数校验以及把 WMS 当作底图 / 图层使用。
---

# MineMap WMS Source

## 概述

WMS（Web Map Service）是 OGC 标准的栅格地图服务。MineMap 4.25 在 source 层新增 WMS 适配，可通过 `addSource({ type: 'wms', ... })` 直接挂载。

入口是统一的 `map.addSource`，与 `raster` / `vector` 风格保持一致：

```javascript
map.addSource("wms-source", {
	type: "wms",
	url: "https://your.wms.server/wms",
	layers: "layer-name",
	...
});
```

## Quick Start

### 通过 capabilities URL 自动构建

```javascript
map.addSource("basemap-wms", {
	type: "wms",
	url: "https://your.wms.server/wms?service=WMS&request=GetCapabilities"
});
```

### 手工配置 GetMap 参数

```javascript
map.addSource("basemap-wms", {
	type: "wms",
	url: "https://your.wms.server/wms",
	layers: "0",
	format: "image/png",
	transparent: true,
	version: "1.1.1",
	tileSize: 256
});

map.addLayer({
	id: "basemap",
	type: "raster",
	source: "basemap-wms"
});
```

## Architecture Positioning

-   WMS 是 `raster` 风格的 source，**不是**独立图层类型
-   引擎内部会按照 WMS 的 `GetMap` 协议发瓦片请求
-   与 `tileSize: 256` / `tileSize: 512` 都兼容，但建议按 WMS 服务侧约定对齐
-   4.25 增强了 WMS 参数校验，配置错误时会更早暴露问题（缺 layer、格式不匹配等）

## 关键参数

| 参数          | 说明                                            |
| ------------- | ----------------------------------------------- |
| `url`         | WMS 服务地址（GetCapabilities 或 GetMap）       |
| `layers`      | 必填，要请求的 WMS 图层标识，多个用英文逗号分隔 |
| `format`      | 图像格式，如 `image/png`、`image/jpeg`          |
| `transparent` | 是否请求透明背景                                |
| `version`     | WMS 协议版本，如 `1.1.1` / `1.3.0`              |
| `tileSize`    | 瓦片大小，建议与 WMS 服务侧输出尺寸一致         |
| `bbox`        | 可选，限制请求范围 `[west, south, east, north]` |
| `crs`         | 可选，坐标系标识，如 `EPSG:4326`                |

## Common Patterns

### 模式 1：WMS 作为单一底图

```javascript
map.addSource("wms", {
	type: "wms",
	url: "https://server/wms",
	layers: "basemap",
	format: "image/png"
});

map.addLayer({
	id: "wms-base",
	type: "raster",
	source: "wms"
});
```

### 模式 2：WMS 与其他栅格叠加

```javascript
map.addSource("wms-base", {
	type: "wms",
	url: "https://server/wms",
	layers: "base",
	format: "image/png",
	tileSize: 512
});

map.addSource("wms-overlay", {
	type: "wms",
	url: "https://server/wms",
	layers: "overlay,labels",
	format: "image/png",
	tileSize: 512
});

map.addLayer({ id: "wms-base", type: "raster", source: "wms-base" });
map.addLayer({ id: "wms-overlay", type: "raster", source: "wms-overlay" });
```

### 模式 3：WMS 限定地理范围

```javascript
map.addSource("wms-region", {
	type: "wms",
	url: "https://server/wms",
	layers: "city-layer",
	format: "image/png",
	bbox: [112, 28, 114, 30]
});
```

## Strict Constraints

### 1. `layers` 必填

WMS 不知道你要画哪一层图，没填 `layers` 不会自动拿第一层，会被 4.25 的参数校验拦下。

### 2. `tileSize` 要与服务对齐

如果服务固定输出 256，但 source 写 512，会导致请求失败或拉伸。建议先在浏览器直接访问 `GetMap` URL 看实际瓦片大小。

### 3. 投影与服务一致

WMS 服务的 SRS / CRS 要与 `Map` 的 `projection` 对齐。

-   `projection: 'MERCATOR'` 配 `EPSG:3857` 服务
-   `projection: 'GLOBE'` 或经纬度投影配 `EPSG:4326` 服务

### 4. WMS 与 `raster` 的区别

不要试图用 WMS source 做矢量查询。WMS 永远是栅格瓦片，能用 `queryRenderedFeatures` 命中 raster layer 拿到的也只是瓦片级 hit。

## Failure Cases

-   整张图变灰瓦片 → 检查 `layers` / `format` / `tileSize` 是否对齐服务
-   404 / 参数错误 → 在浏览器里直接打开拼接好的 `GetMap` URL 看真实报错
-   切片位置错位 → 检查 `bbox` / `projection` / `crs`
-   透明背景全黑 → `transparent: true` 是否传了

## See Also

-   `minemap-style-and-data`
-   `minemap-layer-system`
-   `minemap-global-configuration`
