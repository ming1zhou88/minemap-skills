---
name: minemap-events-and-picking
description: MineMap 事件订阅模型、图层事件代理、once/off 管理、矢量与三维对象拾取。
---

# MineMap Events and Picking

## Quick Start

```javascript
map.on("click", "poi-layer", (e) => {
	const f = e.features?.[0];
	if (!f) return;
	new minemap.Popup()
		.setLngLat(e.lngLat)
		.setHTML(f.properties?.name || "-")
		.addTo(map);
});
```

## Core Concepts

### 1) 事件注册

-   通用：`map.on(type, listener)`
-   图层代理：`map.on(type, layerId|layerIds, listener)`
-   一次性：`map.once(...)`
-   解绑：`map.off(...)`

### 2) 事件类型

常见：`click`、`mousemove`、`mouseenter`、`mouseleave`、`load`、`style.load`、`data`、`idle`。

### 3) 结果差异

-   绑定图层时，回调通常带 `e.features`
-   非图层绑定时，不保证有 `features`

### 4) 三维拾取

-   `map.getFeatureByGPUPick(point)` 可用于三维对象/tileset 拾取
-   SceneModel / SceneTileset 默认可拾取（可通过 `allowPick` 关闭）
-   对图层、3D Tiles、3D Model、Primitive 可使用实例级事件封装（`onLayerOrModel`）

## Common Patterns

### 交互高亮

```javascript
map.on("mousemove", "building-layer", (e) => {
	const fid = e.features?.[0]?.id;
	if (fid == null) return;
	map.setFeatureState({ source: "building", id: fid }, { hover: true });
});
```

### 避免重复绑定

-   在组件销毁时统一 `off`
-   维护事件处理函数引用，避免匿名函数无法解绑

## Demo-backed Patterns

### 图层查询优先 `queryRenderedFeatures`

从多个 demo 看，二维图层交互的稳定做法是：

-   `map.queryRenderedFeatures(e.point, { layers: [...] })`

适合：

-   symbol / line / fill 图层
-   hover 高亮
-   框选结果过滤

业务上不要用 GPU 拾取替代所有二维图层查询。

### 三维对象查询优先 `getFeatureByGPUPick`

大量 demo 已采用：

-   `map.getFeatureByGPUPick(e.point)`
-   `map.getFeatureByGPUPickAsync(e.point)`

适合：

-   `SceneModel`
-   `SceneTileset`
-   `Primitive`
-   粒子、实例化对象、动画对象

如果目标是三维对象，不要先走 `queryRenderedFeatures`。

### 同步与异步拾取的选择

-   一般点击反馈：先用同步 `getFeatureByGPUPick`
-   若当前页面链路更适合 Promise 流程，可用 `getFeatureByGPUPickAsync`

两者不要在同一点击链路里无意义重复调用。

## Source Feature 查询

需要查看 GeoJSON 或 vector source 中已经加载到本地缓存的要素时，使用 `map.querySourceFeatures(sourceId, parameters)`。如果目标是屏幕命中测试，仍然使用 `queryRenderedFeatures`。

```javascript
const features = map.querySourceFeatures("campus-source", {
	sourceLayer: "campus-point-layer",
	filter: ["==", "kind", "school"]
});
```

当前 source 行为：

- vector source 优先传真实 MVT `source-layer`。
- 为了兼容旧写法，如果 `sourceLayer` 传入的是绑定同一 source 的 style layer id，`querySourceFeatureParams.js` 会映射到真实 `source-layer`。
- GeoJSON source 里如果误传 style layer id 作为 `sourceLayer`，源码会容忍并在查询前移除。
- 该方法只查询当前已加载的 source cache 数据，不是服务端全量查询。

批量过滤、结果面板、把查询结果写回高亮 GeoJSON source 时使用它。屏幕空间图层点选继续用 `queryRenderedFeatures`，三维场景对象继续用 `getFeatureByGPUPick`。

Demo 依据：

-   `demo/html/QuerySourceFeaturesSimpleDemo.html`
-   `demo/html/QuerySourceFeaturesMVTDemo.html`

## Performance Tips

-   高频事件（`mousemove`）里避免重计算
-   `queryRenderedFeatures` 必须传 `layers` 限定范围
-   `querySourceFeatures` 必须明确 source id，并尽量给 `filter` 收窄结果
-   `mouseenter/leave` 优于每帧全量命中检测

## Failure Cases

-   对二维图层不加 `layers` 就直接 `queryRenderedFeatures`，导致结果范围过大
-   对关闭了 `allowPick` 的三维对象仍然期待 GPU 拾取结果
-   `mousemove` 中同时做 `queryRenderedFeatures` 和 `getFeatureByGPUPick`，导致重复开销
-   把 `querySourceFeatures` 当成服务端全量查询；它只返回当前 source cache 已加载的 feature

## Demo References

-   `demo/html/GeoJSONWithHover.html`
-   `demo/html/QuerySourceFeaturesSimpleDemo.html`
-   `demo/html/QuerySourceFeaturesMVTDemo.html`
-   `demo/html/GridBoxSelect.html`
-   `demo/html/SymbolPick.html`
-   `demo/html/3DTilesNextPick.html`
-   `demo/html/ParticleSystemClick.html`
-   `demo/html/AnimationTrack.html`

## See Also

-   `minemap-style-and-data`
-   `minemap-scene-components`
