---
name: minemap-layer-system
description: MineMap 图层系统规范。用于图层类型选型、各类 layer 参数展开、增删改移、过滤、缩放范围与渲染顺序控制。
metadata:
  minemap-version: 4.28.2
---

# MineMap Layer System

## 拉伸轮廓（4.28 系列）

extrusion paint 新增 extrusion-outline（默认 false）、extrusion-outline-width（默认 1 像素，非负）、extrusion-outline-color（默认黑色）、extrusion-outline-opacity（默认 1，范围 0..1）。使用 setPaintProperty 热更新，不必复制 Polygon 边界到另一 line layer。这些是 data-constant 属性，支持 schema 指定的 zoom 表达式，不支持 get/feature-state 逐要素控制。layer type 仍是 extrusion，不能使用 Mapbox fill-extrusion。源码为 `source/style/style-reference/style-spec/reference/v8.json`，示例 `demo/html/ExtrusionOutline.html`。

## Architecture Positioning

MineMap 的 layer 体系仍然是 2D/2.5D 地图渲染主干，但不是全部三维能力的统一入口。

从 `source/api/map.js` 与 `source/style/style_layer/*.js` 可确认，正式 style/layer 类型主要包括：

-   `background`
-   `circle`
-   `fill`
-   `line`
-   `symbol`
-   `raster`
-   `heatmap`
-   `extrusion`
-   `sprite`
-   `tracking`
-   `symtracking`
-   `histogram`
-   `dynamicLine` / `dynamic-line` 兼容体系
-   历史兼容的 `3d-model` / `3d-tiles`

结论要先分两层：

1. 仍推荐使用的 style/layer 图层：`background`、`circle`、`fill`、`line`、`symbol`、`raster`、`heatmap`、`extrusion`
2. 偏历史功能或专用特效图层：`sprite`、`tracking`、`symtracking`、`histogram`、`dynamicLine`、`3d-model`、`3d-tiles`

统一原则：

-   图层类问题优先 `addSource()` + `addLayer()`
-   glTF / glb / 3D Tiles 新代码优先 `map.addSceneComponent()`
-   少量三维几何贴地时，很多场景直接用 `Primitive` / `Geometry` 更清晰

## Core Entry Surface

图层系统的主要入口：

-   `map.addLayer(layer, before?)`
-   `map.removeLayer(id)`
-   `map.getLayer(id)`
-   `map.moveLayer(id, beforeId?)`
-   `map.upwardMoveLayer(id, afterId?)`
-   `map.setFilter(layerId, filter)`
-   `map.setSourceFilter(layerId, filter)`
-   `map.setLayerZoomRange(layerId, minzoom, maxzoom)`
-   `map.setPaintProperty(layerId, name, value)`
-   `map.getPaintProperty(layerId, name)`
-   `map.setLayoutProperty(layerId, name, value)`
-   `map.getLayoutProperty(layerId, name)`

## Common Layer Fields

几乎所有 layer 注释都重复强调了同一组通用字段：

-   `id`：图层唯一标识
-   `type`：图层类型
-   `source`：source id；部分 demo 也支持内联 source，但长期维护不如分开写清楚
-   `source-layer` / `sourceLayer`：仅对 MVT / vector tile 有意义
-   `filter`：按要素属性过滤
-   `minzoom` / `maxzoom`：图层显示区间
-   `layout.visibility`：`visible` / `none`

### `source.maxzoom` 和 `layer.maxzoom` 的真实关系

源码注释在多个图层类里都重复写了同一条规则：

-   如果 `source.maxzoom < layer.maxzoom`，则在 `(source.maxzoom, layer.maxzoom]` 之间仍会显示，但本质是在放大 source 最高级别的数据
-   如果 `source.maxzoom > layer.maxzoom`，则在 `(layer.maxzoom, source.maxzoom]` 之间图层不会显示，即使 source 还有更高层级数据

所以：

-   source 决定“数据供给上限”
-   layer 决定“渲染展示区间”

## Recommended Workflow

1. 先判断是不是应该用 layer，而不是 `SceneModel` / `SceneTileset` / primitive
2. 再选 source 类型
3. 再选 layer type
4. 先把数据和过滤逻辑写稳
5. 再逐步加 `layout` / `paint`
6. 最后再处理顺序、拾取、缩放区间、贴地或自动赋高

## 按需参考

编写具体图层时按需阅读类型参数参考；热更新和排序规则仍在入口。

[参数与模式](references/layer-parameters.md)

## Hot Update Rules

视觉参数变化时，优先：

-   `setPaintProperty()`
-   `setLayoutProperty()`
-   `setFilter()`

不要反复：

-   `removeLayer()`
-   `addLayer()`

适合热更新的典型字段：

-   `heatmap-color`
-   `line-color`
-   `line-width`
-   `fill-color`
-   `fill-pattern`
-   `extrusion-pattern`
-   `extrusion-top-pattern`
-   `raster-opacity`

## Failure Cases

### 失败场景 1：把 `symbol` 当普通点层

`symbol` 不是“点 + 文本开关”，而是一整套碰撞、沿线放置、旋转对齐、锚点策略系统。

### 失败场景 2：拿 `extrusion` 代替 glTF 模型

`extrusion` 只适合面拉伸；模型动画、实例化、裁剪、精确拾取等都不该强塞给它。

### 失败场景 3：继续扩散 `3d-model` / `3d-tiles` layer

这两条路径在 4.22 已经明显处于兼容保留状态。

### 失败场景 4：忽略 source 限制去写 layer

例如：

-   `line-gradient` 忘了 `lineMetrics: true`
-   `source-layer` 写在 GeoJSON source 上
-   贴地能力错误地写到不支持的图层上

## Performance Discipline

-   高频调参优先改属性，不重建 layer
-   收紧 `minzoom` / `maxzoom`
-   有过滤条件就提前 `filter`
-   高开销 `symbol`、高程 `symbol-pick-color`、复杂碰撞要谨慎开
-   少量几何别为了“统一”硬塞进 vector tile layer

## Demo References

-   `demo/html/Circle.html`
-   `demo/html/HeatMap.html`
-   `demo/html/LayerWithSymbol.html`
-   `demo/html/LineWithSymbols.html`
-   `demo/html/ExtrusionExpressions.html`
-   `demo/html/ExtrusionPattern.html`
-   `demo/html/gradientLineLayer.html`

## See Also

-   `minemap-style-and-data`
-   `minemap-style-system`
-   `minemap-scene-components`
-   `minemap-2d-3d-overlay-and-classification`
