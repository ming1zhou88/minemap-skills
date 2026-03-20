---
name: minemap-layer-system
description: MineMap 图层系统规范。用于图层类型选型、增删改移、过滤、缩放范围、渲染顺序与图层交互。
---

# MineMap Layer System

## Architecture Positioning

MineMap 的 layer 体系仍是地图渲染主干之一，但它不是“所有三维能力的统一入口”。

要先分清两类：

1. 仍属于 style/layer 体系的图层
2. 已不推荐继续走 source/layer 的旧三维入口

从 `source/api/map.js` 的注释与实现可确认，正式 layer 类型覆盖：

-   `background`
-   `fill`
-   `line`
-   `symbol`
-   `circle`
-   `heatmap`
-   `raster`
-   `extrusion`
-   `sprite`
-   `histogram`
-   `tracking`
-   `symtracking`
-   历史兼容的 `3d-model` / `3d-tiles`

但文档和现有技能库都应坚持：

-   新三维对象优先 `map.addSceneComponent()`
-   不继续扩散 `3d-model` / `3d-tiles` source/layer 路径

## Source-backed Entry Surface

图层系统的正式入口主要是：

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

## Demo-backed Patterns

### 模式 1：标准 source + layer 二段式组织

几乎所有 demo 都仍遵循：

1. `map.addSource(...)`
2. `map.addLayer(...)`

这是 MineMap layer 体系的标准组织方式。典型例子：

-   `demo/html/HeatMap.html`
-   `demo/html/Circle.html`
-   `demo/html/LayerWithSymbol.html`

### 模式 2：图层顺序显式控制

对应源码入口：`moveLayer()`、`upwardMoveLayer()`。

业务中只要有：

-   底图 raster
-   行政区 fill
-   路网 line
-   标签 symbol

就不要依赖“添加先后碰巧正确”，而应显式定义顺序。

### 模式 3：属性热更新优先 `setPaintProperty()`

对应 demo：

-   `demo/html/HeatMap.html`
-   `demo/html/ExtrusionExpressions.html`
-   `demo/html/ExtrusionPattern.html`

这些 demo 都说明一个原则：

-   视觉参数变化时，优先改图层属性
-   不要反复 remove/add layer

典型可热更新属性包括：

-   `heatmap-color`
-   `extrusion-pattern`
-   `extrusion-top-pattern`
-   `extrusion-offset`
-   `extrusion-scale`
-   `raster-opacity`

### 模式 4：symbol 图层要单独看布局语义

对应 demo：

-   `demo/html/LayerWithSymbol.html`
-   `demo/html/LineWithSymbols.html`
-   `demo/html/LayerListener.html`

MineMap 的 symbol 图层不是“带文字的普通点图层”，它有专门的布局语义，如：

-   `icon-image`
-   `text-field`
-   `symbol-placement`

尤其 `symbol-placement: 'line' | 'line-center'`，说明 symbol 经常承担“沿线标注”而不是单点标注。

### 模式 5：heatmap / extrusion / fill 等图层各有自己的 paint 体系

对应 demo：

-   `demo/html/HeatMap.html`
-   `demo/html/ExtrusionExpressions.html`
-   `demo/html/FillExpressions.html`
-   `demo/html/gradientLineLayer.html`

不要把所有 layer 都理解成只会调：

-   颜色
-   透明度
-   宽度

MineMap 各 layer 类型有各自的专属 paint 语义：

-   `heatmap-density` / `heatmap-color`
-   `line-gradient`
-   `extrusion-pattern` / `extrusion-top-pattern`
-   `fill-antialias`

## Recommended Workflow

1. 先确定是不是应该用 layer，而不是 scene component / primitive
2. 再选 layer type
3. 先保证 source 稳定
4. 再决定 `layout` / `paint`
5. 最后显式控制顺序、过滤器和缩放范围

## Strict Constraints

### 1. 新三维入口不要继续走旧 layer 路径

虽然 `map.addLayer()` 仍兼容 `3d-model` / `3d-tiles` 旧写法，但源码注释和现有规范都应统一到：

-   `SceneModel`
-   `SceneTileset`
-   `map.addSceneComponent()`

### 2. 图层顺序必须显式管理

复杂地图里，`addLayer()` 的先后不是长期可靠的架构手段。业务要明确谁压谁。

### 3. `setPaintProperty()` 与 `setLayoutProperty()` 职责不同

-   `paint` 负责视觉表现
-   `layout` 更偏放置、可见性、标注布局

不要把两类更新混着记。

### 4. symbol 查询和渲染有额外成本

`source/api/map.js` 已有注释说明，4.x 的 symbol 查询部分走 GPU 读取像素，性能不能按旧版经验想当然。

## Failure Cases

### 失败场景 1：把所有数据都丢给同一种 layer

例如把热力、路径、标签、行政区全塞到单一 layer 思路里，会直接失去样式能力。

### 失败场景 2：每次视觉变化都 remove/add layer

会造成不必要的重建、闪烁和状态丢失。可热更新属性应直接改 property。

### 失败场景 3：忽略 symbol / raster / fill / extrusion 的边界差异

MineMap 的 layer 不是“统一材质接口”，而是强类型图层系统。

## Performance Discipline

-   高频调参优先 `setPaintProperty()`
-   显式收紧 `minzoom` / `maxzoom`
-   有过滤条件就提前过滤，不要全量常显
-   复杂 symbol 图层少做无节制 hover / pick

## Demo References

-   `demo/html/HeatMap.html`
-   `demo/html/Circle.html`
-   `demo/html/LayerWithSymbol.html`
-   `demo/html/LineWithSymbols.html`
-   `demo/html/LayerListener.html`
-   `demo/html/ExtrusionExpressions.html`
-   `demo/html/ExtrusionPattern.html`
-   `demo/html/gradientLineLayer.html`

## See Also

-   `minemap-style-and-data`
-   `minemap-style-system`
-   `minemap-2d-3d-overlay-and-classification`
-   `minemap-anti-aliasing`
