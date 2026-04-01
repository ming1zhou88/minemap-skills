---
name: minemap-widget-and-controls
description: MineMap 控件与 widget 体系。涵盖 IControl 协议、map.addControl/removeControl/hasControl、自定义控件，以及 Navigation/Scale/Fullscreen/Geolocate/Attribution、Thumbnail、FPSControl、ModelTransformationControl、BatchAddInstanceControl 的使用边界。
---

# MineMap Widget And Controls

## Quick Start

```javascript
const map = new minemap.Map({
	container: "map",
	style,
	center: [116.39, 39.9],
	zoom: 12
});

const navigation = new minemap.Navigation();
const scale = new minemap.Scale({ unit: "metric" });
const thumbnail = new minemap.Thumbnail({
	url: "https://example.com/thumbnail.jpg",
	clickCallback: ({ lnglat }) => {
		map.easeTo({ center: lnglat, zoom: 2, duration: 1500 });
	}
});

map.on("load", () => {
	if (!map.hasControl(navigation)) map.addControl(navigation, "top-right");
	if (!map.hasControl(scale)) map.addControl(scale, "bottom-left");
	if (!map.hasControl(thumbnail)) map.addControl(thumbnail, "bottom-right");
});
```

## Architecture Positioning

### 1) 控件统一走 `IControl` 协议

MineMap 的地图控件不是图层，也不是 scene component，而是挂在地图容器 DOM 上的一类 UI 组件。

标准控件协议来自 `IControl`：

-   `onAdd(map)`：控件挂载时调用，必须返回一个 `HTMLElement`
-   `onRemove(map)`：控件移除时调用，负责解绑事件与清理 DOM
-   `getDefaultPosition()`：返回默认停靠位

合法位置只有四个：

-   `top-left`
-   `top-right`
-   `bottom-left`
-   `bottom-right`

### 2) 生命周期入口只有三个

控件管理主入口都在 `Map` 上：

-   `map.addControl(control, position?)`
-   `map.removeControl(control)`
-   `map.hasControl(control)`

源码约束非常直接：

-   传入对象如果没有 `onAdd` / `onRemove`，`addControl()` / `removeControl()` 会抛出错误事件
-   `position` 不传时，优先调用控件自己的 `getDefaultPosition()`
-   重复添加前应先用 `hasControl()` 判断，避免重复 DOM 和重复监听器

补充一个源码细节：

-   控件如果没有实现 `getDefaultPosition()`，`map.addControl()` 会默认放到 `top-right`

所以 `Navigation`、`Fullscreen`、`Geolocate` 这类没显式默认停靠位的方法，实际默认都落在右上。

### 3) 产品级控件 vs 调试/编辑控件

产品级常驻控件：

-   `Navigation`
-   `Scale`
-   `Fullscreen`
-   `Geolocate`
-   `Attribution`

调试/编辑/演示控件：

-   `FPSControl`
-   `Thumbnail`
-   `ModelTransformationControl`
-   `BatchAddInstanceControl`

推荐把后者看成“工作台工具”，不要默认常驻线上页面。

### 4) `EarthRotationControl` 不是标准 `IControl` widget

虽然 `EarthRotationControl` 也从 `source/index.js` 对外导出，但它的真实用法不是 `map.addControl()`。

它的构造方式是：

```javascript
const rotationController = new minemap.EarthRotationControl(map, options);
rotationController.start();
rotationController.stop();
```

也就是说：

-   它是“地图行为控制器”
-   不是返回 DOM 的标准控件
-   不要把它当 `IControl` 传给 `map.addControl()`

## Built-in Controls Cheat Sheet

### `Navigation`

定位：缩放按钮 + 指南针。

构造参数：

-   `showCompass = true`
-   `showZoom = true`
-   `visualizePitch = false`

特点：

-   不实现 `getDefaultPosition()`，所以默认 `top-right`
-   `visualizePitch = true` 时，指南针会同时可视化 pitch，并调用 `resetNorthPitch()`
-   默认模式下点击指南针调用 `resetNorth()`

### `Scale`

定位：比例尺。

构造参数：

-   `maxWidth = 100`
-   `unit = 'metric' | 'imperial' | 'nautical'`

公开方法：

-   `setUnit(unit)`

默认位置：

-   `bottom-left`

### `Fullscreen`

定位：切换地图容器全屏。

特点：

-   无构造参数
-   不实现 `getDefaultPosition()`，默认 `top-right`
-   会自动检测浏览器全屏能力
-   不支持时会隐藏按钮并给出警告

### `GeolocateControl`

定位：浏览器地理定位按钮。

构造参数：

-   `positionOptions = { enableHighAccuracy: false, timeout: 6000 }`
-   `fitBoundsOptions = { maxZoom: 15 }`
-   `trackUserLocation = false`
-   `showUserLocation = true`

特点：

-   不实现 `getDefaultPosition()`，默认 `top-right`
-   `trackUserLocation = true` 时会进入持续跟踪模式
-   `showUserLocation = true` 时会在地图上放一个用户位置 marker

### `Attribution`

定位：显示归因、数据来源、自定义版权片段。

构造参数：

-   `compact`
-   `customAttribution`：字符串或字符串数组

公开方法：

-   `addAttribution(attribution)`
-   `removeAttribution(attribution)`

默认位置：

-   `bottom-right`

### `Logo`

定位：Logo 水印控件。

特点：

-   默认位置 `bottom-left`
-   `map.addControl(new minemap.Logo(), position, logoUrl)` 里的第三个参数可传 logo 跳转地址
-   更偏引擎级 / 品牌级控件，业务层通常不需要手写管理

### `Thumbnail`

定位：鹰眼缩略图控件。

构造参数：

-   `width = 100`
-   `height = 100`
-   `url`
-   `clickCallback`

默认位置：

-   `bottom-right`

特点：

-   主图移动时会同步更新缩略图视口
-   点击可反向驱动主图定位

### `ModelTransformationControl`

定位：模型位置 / 旋转 / 缩放编辑控件。

构造参数：

-   `onTransform`
-   `onTransformEnd`

默认位置：

-   `top-right`

特点：

-   通过地图点击选择模型
-   回调返回 `{ position, rotation, scale }`
-   更适合编辑器和调试场景

### `BatchAddInstanceControl`

定位：实例化模型批量布设工具。

默认位置：

-   `top-right`

能力边界：

-   支持点 / 线 / 面等方式生成实例位置
-   可结合 `addModel(options)` 直接批量挂实例化模型
-   常见参数包括 `data`、`modelFolder`、`rotations`、`scales`、`positions`、`instanceNames`、`id`

### `FPSControl`

定位：帧率观测。

特点：

-   更适合本地调试
-   在 `map.js` 内部也按控件方式接入
-   无需把它当正式产品 UI

## Complete Control Selection Guide

### 线上常驻优先级

优先常驻：

-   `Navigation`
-   `Scale`
-   `Fullscreen`
-   `Attribution`

按业务决定：

-   `GeolocateControl`
-   `Thumbnail`

默认不常驻：

-   `FPSControl`
-   `ModelTransformationControl`
-   `BatchAddInstanceControl`

### 控件停靠位建议

-   `top-right`：操作性控件，如 `Navigation`、`Fullscreen`、`GeolocateControl`
-   `bottom-left`：环境类控件，如 `Scale`、`Logo`
-   `bottom-right`：归因、鹰眼、说明性控件

如果同时放多个控件：

-   不要把重交互控件和大面积面板全塞在同一个角落
-   `bottom-right` 已经有 `Attribution` 时，再放 `Thumbnail` 要注意遮挡

## Common Patterns

### 常规地图工具栏

```javascript
map.on("load", () => {
	map.addControl(new minemap.Navigation(), "top-right");
	map.addControl(new minemap.Fullscreen(), "top-right");
	map.addControl(new minemap.Scale({ unit: "metric" }), "bottom-left");
});
```

适合常规业务地图页面。`ScaleControl.html` 和常规 demo 都按这个模式组织。

### `Scale` 的稳定用法

`Scale` 是标准 `IControl`，默认位置是 `bottom-left`，支持：

-   `maxWidth`
-   `unit: 'imperial' | 'metric' | 'nautical'`
-   运行时 `setUnit(...)`

推荐：

-   国内业务默认 `metric`
-   海事或航运专题再用 `nautical`
-   不要自己手写比例尺 DOM 去同步缩放

### `Thumbnail` 作为鹰眼/导航 widget

`Thumbnail` 是一个完整控件，不是 demo 私有脚本。

源码和 demo 已验证的关键参数：

-   `url`
-   `width`
-   `height`
-   `clickCallback`

典型用途：

-   显示全球或区域缩略图
-   点击缩略图后联动主地图 `easeTo(...)`
-   在主图移动时同步更新视口指示

```javascript
const thumbnail = new minemap.Thumbnail({
	url: thumbnailUrl,
	clickCallback: ({ lnglat }) => {
		map.easeTo({ center: lnglat, zoom: 0, pitch: 0, bearing: 0, duration: 2000 });
	}
});

map.addControl(thumbnail, "bottom-right");
```

### `ModelTransformationControl` 作为模型编辑 widget

这个控件适合运行时调整三维对象的位置、旋转、缩放。

源码与 demo 共同表明：

-   支持 `onTransform(result)`
-   支持 `onTransformEnd(result)`
-   默认停靠位是 `top-right`
-   应配合可拾取目标使用，至少要保证被编辑对象能被交互命中

```javascript
const transformControl = new minemap.ModelTransformationControl({
	onTransformEnd(result) {
		console.log(result.position, result.rotation, result.scale);
	}
});

map.on("load", () => {
	map.addSceneComponent({
		id: "building",
		type: "3d-model",
		data: modelUrl,
		position: [116.39, 39.9, 10],
		allowPick: true
	});

	map.addControl(transformControl, "top-left");
});
```

适用场景：

-   模型摆位校准
-   编辑器内姿态修正
-   调试阶段记录 position / rotation / scale 快照

### `GeolocateControl` 的真实使用边界

推荐用法：

-   移动端或外勤业务做“定位到我”
-   需要持续跟踪时开 `trackUserLocation`

不推荐：

-   在无 HTTPS / 无权限策略的环境里把它当必定可用功能
-   把它当高精度测量入口

### `Attribution` / `Logo` 的职责分工

建议区分：

-   `Attribution`：数据来源、归因、版权片段
-   `Logo`：品牌跳转与固定 Logo 呈现

不要把两者混成一个自定义 DOM 去重写，除非你明确接管全部合规和样式责任。

不建议直接当最终用户常驻控件。

### `BatchAddInstanceControl` 作为批量布点工具

这个控件本质上是“实例模型批量生成面板 + 地图交互工具”。

已验证能力：

-   默认位置 `top-right`
-   支持点/线/面三类生成方式
-   可通过 `getData()` 读取工具生成的实例结果
-   可与 `ModelTransformationControl` 组合成“先批量生成，再局部校准”的流程

推荐工作流：

1. `map.addControl(new minemap.BatchAddInstanceControl())`
2. 用工具在地图上绘制生成区域/路径/点位
3. 用 `getData()` 导出 position / rotation / scale
4. 需要微调时再挂 `ModelTransformationControl`

### `FPSControl` 只用于性能观察

`FPSControl` 适合本地调试：

-   看帧率波动
-   粗看不同后处理/阴影/模型量级下的性能变化

不建议：

-   作为正式产品 UI 一部分上线
-   用它代替真实 profiling

## Strict Constraints

### 1) 所有标准控件都必须实现 `onAdd()` / `onRemove()`

这是 `map.addControl()` 的硬要求。

### 2) `onAdd()` 必须返回真实 DOM

否则控件无法插入地图容器。

### 3) `EarthRotationControl` 不要走 `addControl()`

它是行为控制器，不是 DOM 控件。

### 4) 编辑型控件依赖交互对象可被命中

如 `ModelTransformationControl` 要依赖模型可选中；若目标对象不允许 pick，控件就没有正常工作基础。

### 自定义 widget 的最小实现

```javascript
class CustomControl {
	onAdd(map) {
		this._map = map;
		this._container = document.createElement("div");
		this._container.className = "minemap-ctrl my-widget";
		this._container.textContent = "Reset";
		this._container.onclick = () => {
			map.easeTo({ center: [116.39, 39.9], zoom: 12, bearing: 0, pitch: 0 });
		};
		return this._container;
	}

	onRemove() {
		this._container?.remove();
		this._map = undefined;
	}

	getDefaultPosition() {
		return "top-left";
	}
}

map.addControl(new CustomControl());
```

约束：

-   `onAdd()` 必须返回真实 DOM
-   `onRemove()` 必须释放事件和引用
-   不要在控件内部偷偷改地图私有字段

## Failure Cases

### 1) 把非 `IControl` 对象传给 `map.addControl()`

错误示例：

```javascript
map.addControl({ start() {}, stop() {} });
```

这类对象没有 `onAdd` / `onRemove`，不符合控件协议。

### 2) 不做判重，反复添加同一个控件实例

错误模式：

```javascript
button.onclick = () => {
	map.addControl(scale);
};
```

正确做法：

```javascript
if (!map.hasControl(scale)) {
	map.addControl(scale);
}
```

### 3) 在 `onRemove()` 里不清监听器

控件是 DOM + map 事件的组合体。只删 DOM 不解绑地图事件，会留下隐藏监听器和状态泄漏。

### 4) 把调试控件当长期业务 UI

`FPSControl`、`ModelTransformationControl`、`BatchAddInstanceControl`、`Thumbnail` 都更接近工具型 widget。

若直接常驻线上：

-   会抬高界面复杂度
-   可能暴露调试能力
-   会让业务页面承担不必要的交互与渲染成本

## Demo-backed References

-   `demo/html/ScaleControl.html`
-   `demo/html/Thumbnail.html`
-   `demo/html/ModelTransformControl.html`
-   `demo/html/BatchAddInstanceControl.html`
-   `demo/html/EarthRotation.html`

## Official Docs And Example Lookup

### 官方入口

控件相关资料，建议固定查这几个入口：

-   官网首页：`https://minedata.cn/`
-   3D Ultra 开发指南：`https://minedata.cn/nce-support/guide-3D-Ultra`
-   3D Ultra API 参考：`https://minedata.cn/nce-support/api-3D-Ultra`
-   3D Ultra 示例中心：`https://minedata.cn/nce-support/demoCenter-3D-Ultra`
-   平台登录入口：`https://map.minedata.cn/main-user/login`

当前公开页面能确认的是：

-   有帮助中心
-   有示例中心
-   有平台登录入口
-   有产品试用入口

当前公开页面不能确认的是：

-   面向所有开发者的公开自助注册流程

所以在 skill 里应写成“登录 / 试用 / 开通”，不要直接写成“注册即可使用”。

### 控件文档怎么查

推荐顺序：

1. 先看示例中心里的“地图控件”分类，确认官方是否有现成案例
2. 再看 API 参考，确认控件类名、构造参数、公开方法
3. 最后回本地 `source/api/control/` 和 `source/api/map.js`，确认默认停靠位、生命周期和真实边界

特别注意：

-   `Navigation`、`Fullscreen`、`GeolocateControl` 的默认位置要以源码为准
-   `ModelTransformationControl`、`BatchAddInstanceControl`、`Thumbnail` 这类工具型控件，最好和本地 demo 一起看
-   官网 API 参考如果和当前仓库版本不一致，以本地源码为准

### 示例中心里该看哪些栏目

控件最相关的官方栏目通常是：

-   地图控件
-   地图基础
-   辅助功能
-   覆盖物（当控件和 `Marker` / `Popup` 联动时）

如果官网只证明“有这个能力”，但没有讲清实现边界：

-   回本 skill 看源码结论
-   再看 `minemap-official-resources-and-onboarding`
-   最终以本地 `source/` 与 `demo/` 为准

## See Also

-   `minemap-official-resources-and-onboarding`
-   `minemap-fundamentals`
-   `minemap-marker-and-popup`
-   `minemap-scene-components`
-   `minemap-events-and-picking`
-   `minemap-performance-and-backend`
