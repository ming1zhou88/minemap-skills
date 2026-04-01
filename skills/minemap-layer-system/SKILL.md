---
name: minemap-layer-system
description: MineMap 图层系统规范。用于图层类型选型、各类 layer 参数展开、增删改移、过滤、缩放范围与渲染顺序控制。
---

# MineMap Layer System

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

## Layer-by-Layer Parameter Guide

### 1) `background`

用途：背景色、背景纹理、DEM 派生地形分析色带。

兼容数据：本质上不依赖普通矢量 source；做坡度/坡向/高程分带时通常配合 terrain。

关键 layout：

-   `visibility`

关键 paint：

-   `background-color`：背景颜色
-   `background-pattern`：背景纹理名，来自 sprite
-   `background-pattern-fixed`：纹理是否固定，不随 zoom 变化
-   `background-opacity`：背景透明度
-   `background-terrain-analysis`：DEM 分析颜色表达式，支持 `slope` / `aspect` / `elevation_band`

MineMap 特有项：

-   `show-depth`：调试项，不建议外部使用
-   `present-shadow`：注释明确写了“外部不支持创建，强烈不建议”

严格约束：

-   背景图层一套底图只能有一个，不应当成可复用模板层
-   地形分析必须先有 terrain / DEM

### 2) `circle`

用途：点要素圆点渲染，适合 POI、采样点、聚合点结果。

兼容数据：`GeoJSONSource`、`VectorTileSource`

关键 layout：

-   `circle-sort-key`
-   `visibility`
-   demo 中可见 `circle-height-key`：用属性字段给 circle 提供高程，不写则不带高程

关键 paint：

-   `circle-radius`
-   `circle-color`
-   `circle-blur`
-   `circle-opacity`
-   `circle-stroke-width`
-   `circle-stroke-color`
-   `circle-stroke-opacity`
-   `circle-pitch-scale`：`map` / `viewport`
-   `circle-pitch-alignment`：`map` / `viewport`
-   `circle-translate`
-   `circle-translate-anchor`

说明：

-   `circle-translate*` 在 4.22 源码注释里已标成“弃用”语义
-   大规模点、样式单一时，源码直接提示也可考虑 `PointPrimitiveCollection`

### 3) `line`

用途：普通线、道路、边界、渐变线、贴地图案线、带附属线名 symbol 的线图层。

兼容数据：`GeoJSONSource`、`VectorTileSource`

关键 layout：

-   `line-cap`：`butt` / `round-butt` / `round` / `square`
-   `line-join`：`miter` / `round` / `bevel`
-   `line-miter-limit`
-   `line-round-limit`
-   `line-sort-key`
-   `line-height-offset`
-   `border-visibility`
-   `visibility`

关键 paint：

-   `line-opacity`
-   `line-color`
-   `line-width`
-   `line-border-width`
-   `line-border-opacity`
-   `line-border-color`
-   `line-gap-width`
-   `line-offset`
-   `line-blur`
-   `line-dasharray`
-   `line-pattern`
-   `line-gradient`
-   `line-translate`
-   `line-translate-anchor`
-   `half-render`
-   `line-normal-direction`

MineMap 特有项：

-   `stencil-test`：解决同层重叠问题的内部向参数，不建议随意手写
-   `symbolLayerConfig`：给线附带一层线名 / 路名 symbol 配置

`symbolLayerConfig` 常用字段：

-   `layout.text-field`
-   `layout.text-size`
-   `layout.text-line-height`
-   `layout.text-allow-overlap`
-   `layout.text-pitch-alignment`
-   `layout.symbol-placement`，默认常见是 `line-center`
-   `layout.text-justify`
-   `layout.text-letter-spacing`
-   `layout.text-max-angle`
-   `paint.text-color`
-   `paint.text-opacity`
-   `depth-test`
-   `auto-height`
-   `auto-height-offset`
-   `hiddenType`

严格约束：

-   `line-gradient` 只能与 `lineMetrics: true` 的 GeoJSON source 一起用
-   少量贴地线优先 `PolylineGeometry`，大量贴地线优先 source 的 `adaptTerrain`
-   `half-render`、`line-normal-direction` 注释仍标记为“暂不生效”

### 4) `fill`

用途：矢量面、区域底色、面贴图、水面类 fill、面附带文字、面附带边线。

兼容数据：`GeoJSONSource`、`VectorTileSource`

关键 layout：

-   `fill-sort-key`
-   `dynamic`：`water` / `none`，注释标记为暂不生效
-   `visibility`

关键 paint：

-   `fill-antialias`
-   `fill-opacity`
-   `fill-color`
-   `fill-outline-color`
-   `fill-pattern`
-   `fill-translate-anchor`
-   `fill-water`
-   `fill-water-texture`
-   `fill-water-speed`
-   `fill-water-wave-size`
-   `sunlight-color`
-   `sunlight-direction`
-   `sky-color`
-   `fill-dynamic-speed`
-   `fill-dynamic-image-url`
-   `fill-offset`
-   `fill-rotation`
-   `fill-scale`

MineMap 特有层级参数：

-   `symbolLayerConfig`：给面附带标签
-   `borderLayerConfig`：给面附带边线
-   `depth`
-   `reflectivity`
-   `flowSpeed`
-   `flowScale`
-   `waterAbsorptionCoeff`
-   `heightOffset`

`symbolLayerConfig` 常见字段：

-   `layout.text-field`
-   `layout.text-size`
-   `layout.text-max-width`
-   `layout.text-line-height`
-   `layout.text-allow-overlap`
-   `layout.text-pitch-alignment`
-   `paint.text-color`
-   `paint.text-opacity`
-   `depth-test`
-   `auto-height`
-   `auto-height-offset`
-   `hiddenType`

`borderLayerConfig` 常见字段：

-   `paint.line-color`
-   `paint.line-opacity`
-   `paint.line-width`
-   `paint.line-dasharray`

严格约束：

-   源码注释明确写了：条件表达式版 `fill-pattern` 仍有问题，“暂时不要使用”
-   `fill-dynamic-speed`、`fill-dynamic-image-url` 注释写明“暂不生效”
-   少量贴地面，源码更推荐 `PolygonGeometry`

### 5) `symbol`

用途：点图标、点文字、点图文、沿线标注、线中心标注、带高程 symbol。

兼容数据：`GeoJSONSource`、`VectorTileSource`

关键 layout：

-   `symbol-placement`：`point` / `line` / `line-center`
-   `symbol-spacing`
-   `symbol-avoid-edges`
-   `symbol-sort-key`
-   `symbol-z-order`
-   `symbol-height-key`
-   `symbol-height-offset`
-   `symbol-pick-color`

图标相关 layout：

-   `icon-image`
-   `icon-size`
-   `icon-rotate`
-   `icon-padding`
-   `icon-keep-upright`
-   `icon-offset`
-   `icon-anchor`
-   `icon-pitch-alignment`
-   `icon-rotation-alignment`
-   `icon-allow-overlap`
-   `icon-ignore-placement`
-   `icon-optional`
-   `icon-text-fit`
-   `icon-text-fit-padding`

文字相关 layout：

-   `text-field`
-   `text-font`
-   `text-size`
-   `text-max-width`
-   `text-line-height`
-   `text-letter-spacing`
-   `text-anchor`
-   `text-max-angle`
-   `text-rotate`
-   `text-padding`
-   `text-keep-upright`
-   `text-transform`
-   `text-offset`
-   `text-allow-overlap`
-   `text-variable-anchor`
-   `text-justify`
-   `text-radial-offset`
-   `text-ignore-placement`
-   `text-optional`
-   `text-pitch-alignment`
-   `text-rotation-alignment`
-   `visibility`

关键 paint：

-   `icon-opacity`
-   `icon-color`
-   `icon-halo-color`
-   `icon-halo-width`
-   `icon-halo-blur`
-   `icon-translate`
-   `icon-translate-anchor`
-   `text-opacity`
-   `text-color`
-   `text-halo-color`
-   `text-halo-width`
-   `text-halo-blur`
-   `text-translate`
-   `text-translate-anchor`

MineMap 特有层级参数：

-   `auto-height`
-   `auto-height-offset`
-   `hiddenType`
-   `classificationType`

严格约束：

-   `classificationType` 只支持 `minemap.ClassificationType.FORCE_VECTOR_OVER_MODEL`
-   `symbol-pick-color` 仅在带高程 symbol 做精确点击时才值得打开，否则可能影响性能
-   `hiddenType` 的场景遮挡不支持沿线文字

### 6) `raster`

用途：影像、地图瓦片、WMTS、TMS、百度瓦片、叠地形、区域裁剪、模板裁剪。

兼容数据：`RasterTileSource`

关键 layout：

-   `visibility`

关键 paint：

-   `raster-opacity`
-   `raster-hue-rotate`
-   `raster-brightness-min`
-   `raster-brightness-max`
-   `raster-saturation`
-   `raster-contrast`
-   `raster-resampling`
-   `raster-fade-duration`

MineMap 特有层级参数：

-   `depth-test`
-   `independent`
-   `wireframe`
-   `maskingEnabled`
-   `maskingPolygonHierarchy`

严格约束：

-   注释里明确写了：`raster-fade-duration` 暂不生效
-   地形下挖基于模板方式，山区可能出现大面积白屏
-   `wireframe` 只是渲染样式开关，不是分析能力入口

### 7) `heatmap`

用途：二维热力图和三维热力图。

兼容数据：`GeoJSONSource`、`VectorTileSource`

关键 layout：

-   `visibility`
-   `display-mode`：`2d` / `3d`

关键 paint：

-   `heatmap-height`
-   `heatmap-radius`
-   `heatmap-weight`
-   `heatmap-intensity`
-   `heatmap-color`
-   `heatmap-opacity`
-   `heatmap-height-factor`

严格约束：

-   `heatmap-height-factor` 仅在 `display-mode = '3d'` 时生效
-   `heatmap-color` 是颜色阶梯表达式；源码会在修改后重建 color ramp

### 8) `extrusion`

用途：白膜建筑、挤出面、顶侧分纹理、扫描和发光特效。

兼容数据：`GeoJSONSource`、`VectorTileSource`

关键 layout：

-   `visibility`

关键 paint：

-   `extrusion-opacity`
-   `extrusion-color`
-   `extrusion-pattern`
-   `extrusion-top-pattern`
-   `extrusion-height`
-   `extrusion-base`
-   `extrusion-pattern-factor`
-   `extrusion-close-stencil-clip`
-   `extrusion-self-bloom`
-   `extrusion-bloom-height`
-   `extrusion-glow-pure-color-height`
-   `extrusion-glow-width`
-   `extrusion-glow-speed`
-   `extrusion-offset`
-   `extrusion-rotation`
-   `extrusion-scale`
-   `extrusion-translate`
-   `extrusion-translate-anchor`

MineMap 特有层级参数：

-   `radiation-regions`
-   `useEnvMap`
-   `metallic`
-   `roughness`
-   `shininess`
-   `topPatternCompatible`

严格约束：

-   `extrusion-translate*` 注释已标明废弃
-   这里是 2.5D 挤出，不等价于 glTF 模型
-   如需真实模型、实例化、骨骼动画、裁剪面，改用 `SceneModel`

### 9) `sprite`

用途：粒子路况线、动态图案线、基于贴图或纯数据的运动线效果。

兼容数据：`GeoJSONSource`、`VectorTileSource`

关键 layout：

-   `sprite-cap`
-   `sprite-join`
-   `sprite-miter-limit`
-   `sprite-round-limit`
-   `sprite-sort-key`
-   `visibility`

关键 paint：

-   `sprite-opacity`
-   `sprite-color`
-   `sprite-width`
-   `sprite-gap-width`
-   `sprite-offset`
-   `sprite-blur`
-   `sprite-speed-factor`
-   `sprite-speed`
-   `sprite-pattern`
-   `sprite-move-direction`
-   `sprite-interpolate`
-   `sprite-status`
-   `sprite-instatus`
-   `sprite-height-offset`
-   `sprite-translate`
-   `sprite-translate-anchor`

严格约束：

-   注释和构造逻辑都说明：此类不能使用 `classificationType` 的贴地形式
-   `sprite-move-direction` 注释写为暂不生效
-   这是专用特效图层，不是普通业务线图层首选

### 10) `tracking`

用途：OD 线、粒子流动轨迹线。

兼容数据：`GeoJSONSource`、`VectorTileSource`

关键 layout：

-   `tracking-cap`
-   `tracking-join`
-   `tracking-round-limit`
-   `tracking-sort-key`
-   `tracking-display-mode`：`loop` / `reverse` / `loop-reserve` / `reverse-reserve`
-   `visibility`

关键 paint：

-   `tracking-opacity`
-   `tracking-color`
-   `tracking-width`
-   `tracking-gap-width`
-   `tracking-offset`
-   `tracking-blur`
-   `tracking-type`
-   `tracking-run-time`
-   `tracking-seg-count`
-   `tracking-seg-group`
-   `tracking-speed`
-   `tracking-delay`
-   `tracking-speed-factor`
-   `tracking-translate`
-   `tracking-translate-anchor`

严格结论：

-   源码注释明确写了“不建议使用”
-   推荐替代方案是 `PolylineGeometry` + `PolylineMaterial` 的 OD 线能力

### 11) `symtracking`

用途：沿轨迹移动的图标跟踪，如车辆、小车、设备移动图标。

兼容数据：`GeoJSONSource`、`VectorTileSource`

关键 layout：

-   `symtracking-display-mode`：`default` / `time-sequence` / `timestamp`
-   `symtracking-time-segment`
-   `symtracking-fps`
-   `symtracking-sort-key`
-   `symtracking-z-order`
-   `compatible-mode`
-   `symbol-spacing`
-   `icon-size`
-   `icon-image`
-   `icon-offset`
-   `text-optional`
-   `visibility`

关键 paint：

-   `icon-opacity`
-   `icon-color`
-   `icon-halo-color`
-   `icon-halo-width`
-   `icon-halo-blur`
-   `symtracking-delay`

层级参数：

-   `classificationType` 只支持 `FORCE_VECTOR_OVER_MODEL`

严格结论：

-   源码注释明确写了“使用较少，暂不推荐”
-   场景动画更推荐 `AnimationManager`
-   使用新数据组织时，demo 注释明确建议 `compatible-mode: false`

### 12) `histogram`

用途：柱状图层、分段颜色柱体、上下波动柱体。

兼容数据：`GeoJSONSource`、`VectorTileSource`

关键 layout：

-   `histogram-max-height-render`
-   `histogram-color-render`
-   `visibility`

关键 paint：

-   `histogram-opacity`
-   `histogram-color`
-   `histogram-colors`
-   `histogram-height`
-   `histogram-base`
-   `histogram-vertical-gradient`
-   `histogram-speed`
-   `histogram-max-height`
-   `histogram-count`
-   `depth-change`
-   `histogram-translate`
-   `histogram-translate-anchor`

严格结论：

-   源码注释明确写了“正在逐渐减少，建议开发者逐渐停止使用”
-   `histogram-colors` 需要 5 个颜色，注释写得很死，不要少给

### 13) `dynamicLine` / `dynamic-line`

用途：历史动态线效果。

当前状态：

-   `dynamic_line_style_layer.js` 类注释明确标记 `@deprecated`
-   源码说明“目前停止维护，由于使用较少且业务支撑能力不足”

结论：

-   只做兼容认知，不用于新业务设计

### 14) 旧三维 layer：`3d-model` / `3d-tiles`

`3d-model`：

-   需要 `TDModelSource`
-   layer 级几乎只有 `3d-model-opacity`、`visibility`、`classificationType`
-   源码注释明确写了：已有 `SceneModel`，功能更丰富，不推荐继续走此路径

`3d-tiles`：

-   需要 `TDTilesSource`
-   layer 级几乎只剩 `point-cloud-color`、`visibility`
-   源码注释明确写了：已有 `SceneTileset`，功能更丰富，不推荐继续走此路径

结论：

-   新代码统一改用 `map.addSceneComponent()`

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
