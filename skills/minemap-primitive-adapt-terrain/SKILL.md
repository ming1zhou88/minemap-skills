---
name: minemap-primitive-adapt-terrain
description: MineMap Primitive 贴地规范。覆盖 `primitive.adaptTerrain`、几何前提、`classificationType` 联动、以及 raster `depth-test` 对贴地结果的影响。
---

# MineMap Primitive Adapt Terrain

## Quick Start

```javascript
const primitive = new minemap.Primitive({
	geometry: new minemap.Geometries.PolylineGeometry({
		positions: minemap.Math.Vector3.fromDegreesArrayHeights([116.39, 39.9, 50, 116.4, 39.9, 50, 116.41, 39.91, 50]),
		width: 12,
		arcType: minemap.ARCTYPE.GEODESIC
	}),
	material: minemap.PolylineMaterial.fromType("TextureFlow", {
		baseColorMap: arrowTexture,
		repeat: [6, 1],
		speed: 2
	}),
	adaptTerrain: true
});

map.addPrimitive(primitive);
```

## Core Concepts

### 1) `primitive.adaptTerrain` 是低层贴地主路径

当你不走 style layer，而是走 `Primitive` 时，正式入口就是：

-   `adaptTerrain: true`

这是低层几何贴 DEM 的主路径。

### 2) 不是所有 geometry 都适合贴地

源码注释已明确把可用核心对象收得很窄：

-   `PolylineGeometry`
-   `PolygonGeometry`

因此 Primitive 贴地更适合：

-   少量高精度线
-   少量高精度面
-   需要自定义材质或交互的低层对象

而不是海量专题数据。

### 3) 开启贴地后会联动 `classificationType`

`Primitive` 源码里，若 `adaptTerrain` 打开，会自动把：

-   `classificationType` 处理到 `ClassificationType.ALL`

因此 Primitive 贴地不是单纯几何插值，而是进入了分类/贴合通道。

### 4) 栅格深度测试会直接影响贴地结果

这一点在 `StandardMaterial` 注释和 demo 都有证据：

-   若底层 raster 没有 `"depth-test": true`
-   Primitive 贴地可能看起来失效或不稳定

所以 Primitive 贴地问题，不能只查 primitive 本身。

## Demo-backed Patterns

### 模式 1：贴地流动线

对应 demo：`demo/html/PolylineAdaptTerrain.html`

这个 demo 很关键，因为它同时验证了：

-   `PolylineGeometry`
-   `PolylineMaterial.fromType("TextureFlow")`
-   `adaptTerrain: true`
-   运行时可在悬浮 / 贴地之间切换

### 模式 2：贴地 polygon + `StandardMaterial`

对应 demo：`demo/html/StandardMaterial.html`

该 demo 证明：

-   `PolygonGeometry`
-   `StandardMaterial.fromType("Texture")`
-   `StandardMaterial.fromType("RadarScan")`
-   `StandardMaterial.fromType("DiffusingRing")`

都可以和 `adaptTerrain: true` 结合。

### 模式 3：Primitive 贴地与二三维叠加不是一回事

Primitive 贴地属于低层几何路径。

如果是大批量矢量专题数据，还是优先：

-   `source.adaptTerrain`

Primitive 贴地不是 style layer 贴地的替代品，而是低层补充。

## Recommended Workflow

1. 先确认你真的需要 `Primitive`
2. 若只是大批量线面专题，改回 source/layer 路径
3. 若是少量精细几何，启 `adaptTerrain: true`
4. 检查底图 raster 是否有 `"depth-test": true`
5. 再验 `classificationType` 与其他三维对象的关系

## Strict Constraints

### 1. Primitive 贴地不等于贴任意三维对象

它的主语义仍是贴 DEM 地形，不应写成“贴附 3D Tiles / glTF 表面”的通用方案。

### 2. 海量业务线面不要全改成 Primitive

源码与 demo 的推荐方向都更偏向：

-   大量数据 → source/layer
-   少量精细对象 → primitive

### 3. `depth-test` 是刚性排查项

尤其当地表有 raster 时，先查这个，再查别的。

### 4. `adaptTerrain` 会影响分类链路

不要在不了解分类副作用时，随意与复杂 `classificationType` 叠加。

## Common Failure Cases

-   明明是海量专题线面，却全部走 Primitive
-   raster 没开 `depth-test`
-   把 Primitive 贴地误写成“贴模型表面”
-   用不合适的 geometry 强行追求贴地效果
-   只盯材质参数，不查 `adaptTerrain` 和分类状态

## Performance Discipline

-   Primitive 贴地只留给少量关键对象
-   大量线面贴地优先 source 路径
-   贴地线若还叠加动态纹理、阴影、反射，要逐项验收

## Demo References

-   `demo/html/PolylineAdaptTerrain.html`
-   `demo/html/StandardMaterial.html`
-   `demo/html/VectorAdaptGroupExample.html`

## See Also

-   `minemap-2d-3d-overlay-and-classification`
-   `minemap-primitives-and-materials`
-   `minemap-material-system-and-shading`
-   `minemap-terrain-and-analysis`
