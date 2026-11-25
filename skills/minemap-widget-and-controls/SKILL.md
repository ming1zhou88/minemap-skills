---
name: minemap-widget-and-controls
description: MineMap 控件与 widget 体系。涵盖 IControl 协议、map.addControl/removeControl/hasControl、自定义控件，以及 Navigation/Scale/Fullscreen/Geolocate/Attribution、Thumbnail、FPSControl、ModelTransformationControl、BatchAddInstanceControl 的使用边界。
metadata:
  minemap-version: 4.28.2
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

## 按需参考

选择具体内建或编辑控件时按需阅读参数和 demo 模式参考。

[参数与模式](references/control-parameters.md)

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
