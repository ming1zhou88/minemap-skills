---
name: minemap-fundamentals
description: MineMap 地图初始化、基础相机参数、生命周期事件、控件添加与清理。用于新建地图、初始化交互、规范化启动流程。
---

# MineMap Fundamentals

## Quick Start

```javascript
const style = {
	version: 8,
	glyphs: "minemap://fonts/{fontstack}/{range}",
	sprite: "minemap://sprite/sprite",
	sources: {},
	layers: []
};

const map = new minemap.Map({
	container: "map",
	style,
	center: [116.39, 39.9],
	zoom: 12,
	pitch: 45,
	bearing: 0,
	projection: minemap.ProjectionType.MERCATOR,
	renderBackend: "auto" // auto -> webgpu/webgl2/webgl1
});

map.on("load", () => {
	map.addControl(new minemap.Navigation(), "top-right");
	map.addControl(new minemap.Scale(), "bottom-left");
});
```

## Core Concepts

### 1) 初始化互斥规则

`Map` 构造要求：

-   用 `center + zoom`，或用 `position`（二选一）
-   不能同时传三者
-   `minZoom <= maxZoom`
-   `minPitch <= maxPitch`

### 2) 默认交互行为

默认开启：`scrollZoom`、`dragPan`、`dragRotate`、`doubleClickZoom`、`keyboard`、`firstPersonView`、触摸缩放/俯仰。

可按需关闭：

```javascript
const map = new minemap.Map({
	container: "map",
	style,
	dragRotate: false,
	firstPersonView: false,
	touchPitch: false
});
```

### 3) 生命周期关键事件

-   `load`: 地图主体可用
-   `style.load`: 样式树完成，可安全加 source/layer
-   `data` / `dataloading`: 数据加载流
-   `idle`: 当前帧无动画/无请求（适合“全部稳定后”逻辑）

### 4) 销毁规范

页面卸载前调用 `map.remove()`，释放 WebGL/事件/worker 资源。

### 5) 全局配置与公开控件

`source/index.js` 还直接暴露了一组全局运行时配置：

-   `minemap.accessToken`
-   `minemap.appKey`
-   `minemap.key`
-   `minemap.solution`
-   `minemap.dataVersion`
-   `minemap.domainUrl`
-   `minemap.dataDomainUrl`
-   `minemap.serverDomainUrl`

建议：

-   在创建第一个 `Map` 之前完成这些配置
-   `domainUrl` 用于重定向静态资源根路径
-   `dataDomainUrl` / `serverDomainUrl` 只在你确实接管数据域、分析服务域时再改
-   不要在地图运行中频繁改这些全局值

公开控件除了常见的 `Navigation`、`Scale`、`Fullscreen`、`Attribution`、`Geolocate`，还包括：

-   `FPSControl`：性能观测控件
-   `Thumbnail`：鹰眼缩略导航控件
-   `ModelTransformationControl`：模型姿态调试/编辑控件
-   `BatchAddInstanceControl`：实例化模型批量落位控件
-   `EarthRotationControl`：地球自转演示控件

这些都走统一的 `map.addControl()` / `map.removeControl()` 生命周期。

控件协议、默认停靠位、自定义控件实现，以及 `Thumbnail` / `ModelTransformationControl` / `BatchAddInstanceControl` 这类工具型 widget 的边界，统一放到独立技能包 `minemap-widget-and-controls`。

### 6) DOM 覆盖物：`Marker` 与 `Popup`

`Marker` 和 `Popup` 已包含在公开导出里，但目前不单独拆独立 skill，统一挂在 fundamentals 下处理。

它们的定位是：

-   `Marker`：地图上的 DOM 标注物，适合少量业务点位、交互把手、编辑锚点
-   `Popup`：地图上的 DOM 信息窗体，适合点击反馈、悬浮信息、说明卡片

源码上的关键约束：

-   两者本质都是 DOM，不是图层对象，也不是 primitive
-   `Popup` 支持高度/海拔语义，但**不支持深度测试**
-   `Marker` / `Popup` 都支持 `distanceDisplayCondition`
-   `Marker` 支持 `draggable`、`rotation`、`pitchAlignment`、`rotationAlignment`
-   `Popup` 支持 `closeOnClick`、`closeOnMove`、`offset`、`trackPointer()`

推荐边界：

-   少量、高交互、需要自定义 DOM 内容 → `Marker` / `Popup`
-   大量点位展示 → 优先 symbol/circle 图层，不要堆大量 DOM marker

## Common Patterns

### 稳定启动顺序

1. 构造 `Map`
2. 监听 `load`
3. 在 `style.load` 后批量添加 source/layer 或 scene component
4. 用 `areTilesLoaded()` 或 `idle` 作为业务“可交互完成”信号

### UI 框架中防抖更新

-   对频繁状态变更使用 `setFilter` / `setPaintProperty`，避免频繁 `setStyle`
-   大规模样式切换才用 `setStyle`

### 控件分层使用建议

-   产品级常驻控件：`Navigation`、`Scale`、`Fullscreen`、`Geolocate`、`Attribution`
-   调试级控件：`FPSControl`、`ModelTransformationControl`
-   演示或编辑级控件：`BatchAddInstanceControl`、`EarthRotationControl`、`Thumbnail`

调试级和演示级控件不要默认常驻线上业务页面。

### `Marker` + `Popup` 基础用法

```javascript
const marker = new minemap.Marker({
	color: "red",
	draggable: true,
	offset: [0, -18]
})
	.setLngLat([116.39, 39.9])
	.addTo(map);

const popup = new minemap.Popup({
	closeButton: false,
	closeOnClick: false,
	offset: [0, -30]
})
	.setHTML("<div>设备点位</div>")
	.setLngLat([116.39, 39.9]);

marker.setPopup(popup).togglePopup();
```

### 地形场景下的 marker

MineMap 源码和 demo 都表明：`Marker` 支持地形场景下的拖拽和高度控制。

常见链路：

-   `setLngLat(...)`
-   `setAltitude(...)`
-   `setDraggable(true)`
-   `setPopup(...)`

适合编辑点、观测点、设备点位等少量对象。

### `Popup` 单独使用

```javascript
new minemap.Popup({ closeOnMove: true }).setLngLat(e.lngLat).setHTML("<strong>picked</strong>").addTo(map);
```

如果是跟鼠标走的信息提示，可用 `trackPointer()`；但不要和固定 `setLngLat(...)` 的逻辑混在一套状态里反复切换。

## Performance Tips

-   低端设备优先降低初始 `pitch` 与特效开关（SSR、OIT、阴影）
-   控制地图同时在线数据源数量
-   避免每帧触发高成本查询（如全图 `queryRenderedFeatures`）
-   `Marker` / `Popup` 数量要受控，海量点位不要走 DOM 覆盖物

## See Also

-   `minemap-widget-and-controls`
-   `minemap-style-and-data`
-   `minemap-events-and-picking`
-   `minemap-performance-and-backend`
