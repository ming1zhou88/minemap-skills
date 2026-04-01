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
	position: [116.39, 39.9, 3000],
	pitch: 60,
	bearing: 0,
	roll: 0,
	minZoom: 1,
	maxZoom: 22,
	projection: minemap.ProjectionType.MERCATOR,
	renderBackend: "auto" // auto -> webgpu/webgl2/webgl1
});

map.on("load", () => {
	map.addControl(new minemap.Navigation(), "top-right");
	map.addControl(new minemap.Scale(), "bottom-left");
});
```

## Core Concepts

### 1) 初始化互斥规则与推荐相机表达

`Map` 构造要求：

-   用 `center + zoom`，或用 `position`（二选一）
-   不能同时传三者
-   `minZoom <= maxZoom`
-   `minPitch <= maxPitch`

但从 MineMap 4.x 的真实相机模型来看，**新代码更推荐优先使用 `position`，而不是 `center + zoom`**。

原因：

-   `position` 直接表达相机位置 `[lng, lat, height]`
-   和 3D 场景、地下模式、精确姿态控制更一致
-   更容易和后续 `target` 飞行、`setCameraPositionHeadingPitchRoll()`、三维模型调试联动
-   `center + zoom` 更像兼容传统 2D 地图思维，适合快速起图，不适合作为长期的 3D 主表达

推荐理解：

-   简单 2D 起图、临时 demo → `center + zoom`
-   正式 3D 项目、相机控制、场景漫游 → `position + bearing + pitch + roll`

```javascript
const map = new minemap.Map({
	container: "map",
	style,
	position: [116.39, 39.9, 1500],
	pitch: 55,
	bearing: 15,
	roll: 0
});
```

关于 `position`：

-   语义是 `[经度, 纬度, 距地表高度(米)]`
-   未开启地下模式时，不要随意给负高度
-   它表达的是“相机在哪里”，不是“地图中心在哪里”

### 2) `Map` 参数应该怎么分组理解

最常用的构造参数可以分成 5 组：

#### A. 容器与样式

-   `container`
-   `style`
-   `projection`

这是创建地图的必备入口。

#### B. 相机初始姿态

-   `position` 或 `center + zoom`
-   `bearing`
-   `pitch`
-   `roll`

这组参数决定你打开页面时“从哪里、以什么姿态看场景”。

#### C. 相机约束

-   `minZoom` / `maxZoom`
-   `minPitch` / `maxPitch`
-   `maxBounds`
-   `minHeightToDemSurface`

这组参数用来限制用户视角跑飞。

#### D. 交互开关

-   `scrollZoom`
-   `dragPan`
-   `dragRotate`
-   `doubleClickZoom`
-   `keyboard`
-   `firstPersonView`
-   `touchZoomRotate`
-   `touchPitch`

#### E. 渲染与三维环境

-   `renderBackend`
-   `renderBackendPreference`
-   `skyBox`
-   `earthSkin`
-   `fxaa`
-   `OIT`
-   `SSR`
-   `logDepth`

推荐做法：

-   基础项目先只配必需项和相机项
-   三维特效项按需逐个打开，不要初始化就全开
-   `renderBackend` 优先 `auto`
-   `logDepth`、`SSR` 这类高级项要结合专题 skill 再启用

### 3) 默认交互行为

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

### 4) 生命周期关键事件

-   `load`: 地图主体可用
-   `style.load`: 样式树完成，可安全加 source/layer
-   `data` / `dataloading`: 数据加载流
-   `idle`: 当前帧无动画/无请求（适合“全部稳定后”逻辑）

### 5) 销毁规范

页面卸载前调用 `map.remove()`，释放 WebGL/事件/worker 资源。

### 6) 全局配置与公开控件

`source/index.js` 还直接暴露了一组全局运行时配置：

-   `minemap.key`
-   `minemap.solution`
-   `minemap.dataVersion`
-   `minemap.domainUrl`
-   `minemap.dataDomainUrl`
-   `minemap.serverDomainUrl`
-   `minemap.serviceUrl`
-   `minemap.spriteUrl`
-   `minemap.fontsUrl`

建议：

-   在创建第一个 `Map` 之前完成这些配置
-   新代码优先配置 `key` + `solution`，不再把 `accessToken` 当主推荐入口
-   `domainUrl` 用于重定向静态资源根路径
-   `dataDomainUrl` / `serverDomainUrl` 只在你确实接管数据域、分析服务域时再改
-   不要在地图运行中频繁改这些全局值

兼容入口 `appKey` / `accessToken` 仍存在，但当前源码真实优先级是 `key > appKey > accessToken`。完整配置说明、`window.minemapCDN`、demo 取值方式和私有化部署边界，统一放到独立技能包 `minemap-global-configuration`。

公开控件除了常见的 `Navigation`、`Scale`、`Fullscreen`、`Attribution`、`Geolocate`，还包括：

-   `FPSControl`：性能观测控件
-   `Thumbnail`：鹰眼缩略导航控件
-   `ModelTransformationControl`：模型姿态调试/编辑控件
-   `BatchAddInstanceControl`：实例化模型批量落位控件
-   `EarthRotationControl`：地球自转演示控件

这些都走统一的 `map.addControl()` / `map.removeControl()` 生命周期。

控件协议、默认停靠位、自定义控件实现，以及 `Thumbnail` / `ModelTransformationControl` / `BatchAddInstanceControl` 这类工具型 widget 的边界，统一放到独立技能包 `minemap-widget-and-controls`。

### 7) 核心数据结构：图层、模型、geometry 到底分别是什么

MineMap 里最容易混淆的，不是 API 名字，而是对象层级。

可以把常见结构分成 4 层：

#### A. style / source / layer：二维地图数据层

这是地图底图和常规专题图层的主数据结构。

```javascript
const style = {
	version: 8,
	glyphs: "minemap://fonts/{fontstack}/{range}",
	sprite: "minemap://sprite/sprite",
	sources: {
		poi: {
			type: "geojson",
			data: { type: "FeatureCollection", features: [] }
		}
	},
	layers: [
		{
			id: "poi-circle",
			type: "circle",
			source: "poi",
			paint: { "circle-radius": 6, "circle-color": "#2d8cf0" }
		}
	]
};
```

这里的职责是：

-   `style`：整张地图的样式树
-   `source`：数据来源
-   `layer`：把某个 source 画成具体视觉结果

适合：

-   点线面专题图
-   栅格底图
-   矢量瓦片
-   普通业务可视化

#### B. scene component：模型 / 3D Tiles / 场景对象层

这是 MineMap 4.x 推荐的三维主入口。

```javascript
map.addSceneComponent({
	id: "building",
	type: "3d-model",
	data: modelUrl,
	position: [116.39, 39.9, 10],
	rotation: [0, 0, 0],
	scale: [1, 1, 1]
});

map.addSceneComponent({
	id: "cityTiles",
	type: "3d-tiles",
	urls: [{ url: tilesetUrl, name: "city" }]
});
```

这里常见对象有：

-   `3d-model` → `SceneModel`
-   `3d-tiles` → `SceneTileset`
-   `scene-object` → `Earth` / `Skybox` / `Panorama` / `AirLine` 等

适合：

-   glTF / glb 模型
-   BIM
-   倾斜摄影
-   3D Tiles
-   场景级三维对象

#### C. primitive：低层三维绘制对象

`Primitive` 是比 scene component 更底层的一层。

它通常由：

-   `geometry`
-   `material`
-   `modelMatrix`

共同构成。

```javascript
const geometry = new minemap.Geometries.BoxGeometry({
	width: 100,
	height: 60,
	depth: 40
});

const primitive = new minemap.Primitive({
	geometry,
	material: minemap.StandardMaterial.fromType("Color", {
		color: "green",
		opacity: 0.8
	}),
	modelMatrix: matrix
});

map.addPrimitive(primitive);
```

适合：

-   分析辅助体
-   自定义三维标绘
-   低层材质实验
-   不想走完整 model / tileset 管线时的自定义对象

#### D. geometry：几何结构本体

`geometry` 只负责“形状”，不负责颜色、贴图、光照和挂载。

常见有：

-   `BoxGeometry`
-   `PlaneGeometry`
-   `PolygonGeometry`
-   `PolylineGeometry`
-   `CircleGeometry`
-   `TubeGeometry`

要点：

-   `geometry` 不是可直接上屏的最终对象
-   通常要和 `material` 一起放进 `Primitive`
-   如果你的需求是“加载模型文件”，就不该从 geometry 下手，而该走 scene component

可用一个简单判断法：

-   业务专题地图 → layer
-   外部三维资产（glTF/3DTiles）→ scene component
-   手工三维分析体 / 调试体 → primitive + geometry + material

### 8) DOM 覆盖物：`Marker` 与 `Popup`

`Marker` 和 `Popup` 现在单独拆到了独立 skill。

在 fundamentals 里只记住 4 条：

-   它们都是 DOM 覆盖物，不是 layer / primitive / scene component
-   两者都支持 `distanceDisplayCondition`
-   两者都支持通过 `setAltitude()` 表达相对地表高度
-   `Popup` 支持高度，但**不支持深度测试**

完整参数、三维高度联动、地形拖拽和 `Marker + Popup` 绑定细节，统一看 `minemap-marker-and-popup`。

## Common Patterns

### 稳定启动顺序

1. 构造 `Map`
2. 监听 `load`
3. 在 `style.load` 后批量添加 source/layer 或 scene component
4. 用 `areTilesLoaded()` 或 `idle` 作为业务“可交互完成”信号

### 视角跳转与飞行动画：`jumpTo` / `easeTo` / `flyTo`

MineMap 的视角动画不要只理解成“地图平移一下”。

它实际上有 3 套常用入口：

-   `jumpTo()`：立即切换，没有过渡
-   `easeTo()`：常规缓动过渡，适合 UI 导航、轻量视角调整
-   `flyTo()`：飞行路径过渡，适合场景切换、模型聚焦、远近景切换

推荐原则：

-   初始化落位、无动画同步 → `jumpTo()`
-   面板点击、局部视角微调 → `easeTo()`
-   从全局飞到局部、从一个对象飞到另一个对象 → `flyTo()`

#### 动画目标推荐写法

对于动画接口，**更推荐用 `target`，而不是继续坚持 `center + zoom`**。

因为 `target` 能直接表达：

-   一个经纬高点 `[lng, lat, height]`
-   `Primitive`
-   `SceneModel`
-   `SceneTileset`
-   `BoundingSphere` / `BoundingBox`

这和 3D 场景更匹配。

```javascript
map.flyTo({
	target: [116.39, 39.9, 1200],
	bearing: 20,
	pitch: 60,
	duration: 2000
});
```

或者直接飞向一个场景对象：

```javascript
map.flyTo({
	target: map.getSceneComponent("cityTiles"),
	pitch: 55,
	bearing: 0,
	duration: 2500
});
```

#### 常用动画参数说明

-   `target`：目标对象或目标点，3D 场景首选
-   `center` + `zoom`：旧式地图视图表达，适合 2D 兼容场景
-   `bearing`：旋转角；对 `target` 模式来说，更接近相机航向角
-   `pitch`：俯仰角
-   `roll`：横滚角；只有相机姿态型场景才真的有意义
-   `duration`：动画时长，单位毫秒
-   `easing(t)`：缓动函数，输入 $t \in [0,1]$
-   `offset`：终点相对屏幕中心的偏移，适合让目标避开弹窗/侧边栏
-   `animate: false`：退化成无动画

`flyTo()` 额外常见参数：

-   `curve`：飞行弧线强度，默认 1.42；越大越“拱”
-   `speed`：飞行速度，默认 1.2
-   `screenSpeed`：按屏幕速度估算飞行时长；如果设了 `speed`，它会被忽略
-   `maxDuration`：限制最长飞行时间
-   `minZoom`：飞行过程中的最小缩放级别
-   `maximumHeight`：飞行过程中的最大高度约束
-   `flyOverLongitude` / `flyOverLongitudeWeight`：控制跨经线飞行路径

#### 什么时候该用 `position`，什么时候该用 `target`

-   `Map` 初始化参数 → 优先 `position`
-   动画 / 视角跳转参数 → 优先 `target`
-   传统 2D 地图兼容写法 → `center + zoom`

如果你做的是三维项目，长期最好形成这个习惯：

-   起图看相机位置 → `position`
-   飞行看目标对象 → `target`

#### 一个稳妥的飞行动画模板

```javascript
function focusSceneObject(id) {
	const target = map.getSceneComponent(id);
	if (!target) return;

	map.flyTo({
		target,
		pitch: 55,
		bearing: 15,
		duration: 2000,
		curve: 1.42,
		speed: 1.2,
		offset: [120, 0]
	});
}
```

这个模式适合：

-   点击树节点聚焦模型
-   点击列表定位到 3D Tiles 子场景
-   从总览飞到设备对象

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
-   飞行动画不要在高频 hover 事件里反复触发，`flyTo()` 更适合明确的导航动作
-   场景切换优先给稳定的 `duration`，不要在短时间内连续堆多个相机动画

## See Also

-   `minemap-official-resources-and-onboarding`
-   `minemap-global-configuration`
-   `minemap-widget-and-controls`
-   `minemap-marker-and-popup`
-   `minemap-style-and-data`
-   `minemap-scene-components`
-   `minemap-primitives-and-materials`
-   `minemap-events-and-picking`
-   `minemap-performance-and-backend`
