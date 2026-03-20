---
name: minemap-business-spatial-analysis-suite
description: MineMap 空间分析补充能力规范。覆盖地形专题分析、剖面、限高、挖填方、地形开挖、开敞度、干扰分析等非可视域/淹没/量算类分析。
---

# MineMap Business: Spatial Analysis Suite

## Quick Start

```javascript
const cutFill = new minemap.CutFill({
	positions: minemap.Math.Vector3.fromDegreesArrayHeights([
		116.39, 39.9, 10, 116.4, 39.9, 11, 116.4, 39.91, 12, 116.39, 39.91, 10
	]),
	enabled: true
});

map.analysis.cutFillCollection.add(cutFill);
```

## Skill Scope

本 skill 专门承接空间分析家族里此前未单独拆包、但已经有公开导出和 demo 证据的能力：

-   `map.setTerrainAnalysis(...)`
-   `CutFill`
-   `Excavation`
-   `LimitHeight`
-   `ViewDome`
-   `Interference`
-   `InterferenceConjoined`
-   `Profile`

不重复覆盖的对象：

-   `Viewshed3D` / `Sightline` → `minemap-business-visibility-analysis`
-   `FloodAnalysis` / `FloodAnalysisOffline` / `ShadowAnalysis` → `minemap-business-flood-and-shadow`
-   `Measurement3D` → `minemap-business-measurement`

## Analysis Family Map

### 1) 地形专题分析：`map.setTerrainAnalysis(...)`

这不是分析对象集合，而是地形渲染分析模式。

典型组合：

-   `minemap.TerrainAnalysisType.SLOPE`
-   `minemap.TerrainAnalysisType.ASPECT`
-   `minemap.TerrainAnalysisType.ELEVATION`
-   `minemap.InterpolationType.EXPONENTIAL`
-   `minemap.InterpolationType.INTERVAL`

适合：

-   坡度分析
-   坡向分析
-   等高面分带
-   限定区域地形专题图

对应 demo：`demo/html/DemAnalysis.html`、`demo/html/ContourAnalysis.html`

### 2) 体量与地形改造分析

#### `CutFill`

适合：

-   填挖方体积评估
-   基准面抬升/降低后的土方量对比

对应 demo：`demo/html/CutAndFillAnalysis.html`

#### `Excavation`

适合：

-   地形开挖
-   基坑、沟槽、地下空间挖除效果

对应 demo：`demo/html/DiggingTerrain.html`、`demo/html/DiggingTerrainWithIgnoreClassification.html`

#### `LimitHeight`

适合：

-   限高区校核
-   建筑超高识别

注意：`LimitHeight` 实际挂在 `map.analysis.highlightCollection` 下管理，而不是单独的 `limitHeightCollection`。

对应 demo：`demo/html/LimitHeightAnalysis.html`

### 3) 视场与干扰专题

#### `ViewDome`

适合：

-   开敞度分析
-   视场覆盖占比评估

对应 demo：`demo/html/ViewDome.html`

#### `Interference`

适合：

-   单视点干扰分析
-   球面与投影面的可见/遮挡区域表达

对应 demo：`demo/html/Interference.html`

#### `InterferenceConjoined`

适合：

-   多个干扰结果的联合表达
-   先有单个 `Interference`，再做联动汇总

对应 demo：`demo/html/Interference.html`

### 4) 剖面专题

#### `Profile`

适合：

-   地形与模型剖面线分析
-   起终点间高程曲线
-   引擎计算与服务端混合计算

对应 demo：`demo/html/Profile.html`

## Demo-backed Patterns

### 模式 1：地形分析不是 collection，而是 terrain shader 配置

`DemAnalysis.html` 说明：

-   先 `map.setTerrain(...)`
-   再调用 `map.setTerrainAnalysis(styleConfig)`
-   `styleConfig` 的核心是：
    -   `property`
    -   `stops`
    -   `region`
    -   `hasArrow`

因此它更像“地形专题渲染”，不是实体分析对象。

### 模式 2：`CutFill` / `LimitHeight` 都依赖交互圈区

`CutAndFillAnalysis.html` 与 `LimitHeightAnalysis.html` 都体现了相同工作流：

1. 鼠标绘制范围
2. 通过 GPU pick 获取三维点
3. 将范围写回分析对象
4. 再调颜色、透明度、基准面或限高值

这类分析的本质是“先定区，再调参数”，不是初始化一次就结束。

### 模式 3：`Excavation` 更接近地形改造对象

`DiggingTerrain.html` 证明：

-   `positions`
-   `depth`
-   `sampleDistance`
-   `enabled`

是最关键的运行时参数。

它和 `CutFill` 的差别是：

-   `CutFill` 更偏结果评估
-   `Excavation` 更偏直接修改开挖表达

### 模式 4：`ViewDome` / `Interference` 都支持参数化创建与点击创建

`ViewDome.html`、`Interference.html` 都不是只做静态展示，它们都验证了：

-   参数直接创建
-   鼠标点击创建
-   运行时改颜色、角度、半径、可见类型
-   `remove()` / `removeAll()` 等集合管理

这类分析非常适合封装成“交互工具模式”。

### 模式 5：`Profile` 是 Promise 型分析流程

`Profile.html` 说明它不是简单 collection 对象：

-   `new minemap.Profile(map, options)`
-   `profile.analysis().then((result) => { ... })`

结果里不只有高程点列，还包含：

-   `samplesNumber`
-   `minHeight`
-   `maxHeight`
-   `horizontalDistance`
-   `verticalDistance`
-   `surfaceDistance`

因此它更适合“分析结果回传 + 图表联动”的业务。

## Strict Constraints

### 1. 地形专题分析必须先有 terrain

`setTerrainAnalysis(...)` 的前提是已启用 DEM。不要在平面模式下把它当普通 raster 样式去用。

### 2. `LimitHeight` 的容器容易记错

它走的是 `map.analysis.highlightCollection`，不是独立集合。业务封装时必须把这个非直觉入口写清楚。

### 3. `Profile` 的服务端模式有明确边界

当 `profileType = ACCEPT_COMPUTING` 时：

-   `terrainServer` 或 `modelServer` 至少要有一个
-   当前语义是“服务端计算地形与 3D Tiles 高度”

不要把它写成任意场景对象都能服务端剖面。

### 4. `Excavation` 与叠加对象关系要单独验收

开挖会与地形、栅格、3D Tiles 的分类/深度关系耦合。涉及模型叠加时，要结合 `IGNORECLASSIFICATION` 等配置单独验收。

### 5. 地形分析色带不是纯视觉配置

`stops` 对应的是业务阈值，不应只按好看配色。尤其是坡度、坡向、等高面分带，必须先和业务指标对齐。

## Common Failure Cases

-   未启用 DEM 就尝试 `setTerrainAnalysis(...)`
-   `CutFill` / `LimitHeight` 区域点数不足或未形成有效面
-   `Excavation` 画完后丢失对象引用，后续无法改 `depth` 或 `enabled`
-   把 `LimitHeight` 当成普通高亮效果，忽略其超高判定语义
-   `Profile` 忘记等待 `analysis()` 的 Promise 完成就直接取结果
-   `InterferenceConjoined` 在没有基础 `Interference` 结果时直接使用

## Recommended Workflow

### 地形专题分析

1. 开启 DEM
2. 确定分析类型
3. 设定 `stops` 和 `region`
4. 用 GUI 或业务表单调参

### 区域型空间分析

1. 交互圈定范围
2. 创建分析对象并加入对应 collection
3. 运行时只改对象属性，不反复销毁重建
4. 切场景时统一 `removeAll()`

### 剖面分析

1. 获取起终点
2. 创建 `Profile`
3. `await` / `then` 获取分析结果
4. 再驱动剖面线和图表展示

## Demo References

-   `demo/html/DemAnalysis.html`
-   `demo/html/ContourAnalysis.html`
-   `demo/html/CutAndFillAnalysis.html`
-   `demo/html/DiggingTerrain.html`
-   `demo/html/DiggingTerrainWithIgnoreClassification.html`
-   `demo/html/LimitHeightAnalysis.html`
-   `demo/html/ViewDome.html`
-   `demo/html/Interference.html`
-   `demo/html/Profile.html`

## See Also

-   `minemap-terrain-and-analysis`
-   `minemap-business-visibility-analysis`
-   `minemap-business-flood-and-shadow`
-   `minemap-business-measurement`
-   `minemap-2d-3d-overlay-and-classification`
