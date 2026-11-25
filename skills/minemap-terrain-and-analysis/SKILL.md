---
name: minemap-terrain-and-analysis
description: MineMap 地形设置（setTerrain）、DEM 高程查询、地形分析、TerrainLeveling、相机碰撞高度与 terrain 相关约束。
metadata:
  minemap-version: 4.28.2
---

# MineMap Terrain and Analysis

## Quick Start

```javascript
const map = new minemap.Map({
	container: "map",
	style,
	position: [89.11, 27.4, 15000],
	pitch: 55,
	projection: minemap.ProjectionType.MERCATOR,
	minHeightToDemSurface: 5,
	terrainSkirtHeight: -1
});

map.on("load", () => {
	map.setTerrain({
		tiles: ["https://example.com/dem/{z}/{r}/{c}/{z}_{x}_{y}.terrain"],
		availabilities: ["https://example.com/dem/layer.json"],
		tileSize: 512,
		maxzoom: 14,
		extensions: {
			requestVertexNormals: false,
			requestWaterMask: false,
			requestHeightFieldNormals: false
		}
	});

	map.addSource("imagery", {
		type: "raster",
		tiles: ["https://example.com/satellite/{z}/{r}/{c}/{z}_{x}_{y}.jpg"],
		tileSize: 512
	});

	map.addLayer({
		id: "imagery",
		type: "raster",
		source: "imagery"
	});

	const height = map.queryDEMHeightByLngLat([89.11316, 27.40665]);
	console.log(height.z);
});
```

## Core Concepts

### 1) 地形不是一个普通图层，而是“基础球面网格替换”

`map.setTerrain(...)` 的本质不是“再叠一层 DEM 图层”。

它做的是：

-   把地图底层的基础球面 mesh 从平面/椭球模式切到 terrain mesh
-   用 DEM 瓦片驱动底层地表几何
-   让后续贴地能力、地形拾取、相机碰撞、坡度/坡向分析都建立在这套 terrain 数据之上

所以要记住：

-   影像图层还是普通 `raster` source/layer
-   地形本身由 `map.setTerrain()` 管
-   terrain 开关会影响全局 quadtree、代理 source 缓存、地表拾取和碰撞逻辑

这就是为什么 MineMap 4.x 已不推荐继续用旧式 `terrainTiles` + `renderWithTerrain` 那套 source/layer 写法。

### 2) `setTerrain()` 背后实际做了什么

`map.setTerrain(options)` 是 MineMap 4.x 推荐 DEM 接口。

源码里能确认的实际行为有：

-   2 秒后清掉 `symbolHeightExtraction` 缓存
-   调 `style.rasterSourceCache.setTerrain(options)` 切换 terrain source
-   重置所有 sourceCache 上的 quadtree，避免切地形后包围盒/框选错误
-   更新 `proxySourceCache` 的 layer order，保证矢量贴地缓存状态刷新

所以正确理解是：

-   `setTerrain()` 是一个全局场景切换动作
-   不要把它当作每帧频繁开关的轻量操作

### 3) Terrain 数据在内部如何工作

`TerrainTileSource` 是内部 terrain 数据源。

从源码可确认：

-   terrain 默认格式是 `quantized`
-   也兼容 `heightmap`
-   terrain tile 会单独缓存到 `_terrainTiles`
-   卸载后的 terrain tile 会暂存到 `_tempTerrainTiles`，用来减轻切换时闪烁
-   请求失败时，会生成替代球面 mesh 兜底，而不是直接让地表缺块
-   超过 `maxzoom` 时，不再继续请求更细 DEM，而是走上采样 `upsample`

这决定了两个开发结论：

-   `maxzoom` 不是越大越好，取决于你实际 DEM 精度
-   terrain 请求失败时未必马上出现黑洞，但精度会退化

### 4) 地形与影像的关系

只开 `setTerrain()` 还不够，它只负责地表几何。

通常你还需要再加一个普通影像或栅格底图：

-   terrain 决定“地表起伏”
-   raster / vector layer 决定“地表画什么”

典型组合：

-   `map.setTerrain(...)`
-   `map.addSource("imagery", { type: "raster", ... })`
-   `map.addLayer({ type: "raster", source: "imagery" })`

### 5) 地形和贴地/自动赋高不是一回事

地形开启后，只是让地图有真实起伏。

要不要贴地，还要看对象本身：

-   style layer 要看各自的贴地能力和分类配置
-   primitive 要看 `adaptTerrain`
-   `Marker` / `Popup` 要看 `setAltitude()` 与 terrain pick
-   某些分析对象要直接依赖 DEM 查询

不要把“开启 terrain”误解成“所有对象自动贴地完成”。

## 按需参考

设置地形 provider、查询 DEM、地下模式、区域瓦片或地形分析时按需阅读参考。

[参数与模式](references/terrain-parameters-and-analysis.md)

## Analysis Containers

### 4) 分析容器

`map.analysis` 下默认挂载：

-   `cutFillCollection`
-   `excavationCollection`
-   `highlightCollection`
-   `sightlineCollection`
-   `viewDomeCollection`
-   `viewshed3DCollection`
-   `interferenceCollection`
-   `shadowAnalysisCollection`
-   `floodAnalysisCollection`
-   `floodAnalysisOfflineCollection`

### 5) `index.js` 已导出的其他分析类

除了可视域、通视、淹没、阴影，公开导出里还包括：

-   `Highlight`
-   `CutFill`
-   `Excavation`
-   `ViewDome`
-   `Interference`
-   `InterferenceConjoined`
-   `Profile`
-   `Player`

### 6) 空间分析 skill 拆分结果

当前空间分析相关能力已按职责拆成 4 个 skill，不再只放在一个总包里：

-   `minemap-terrain-and-analysis`
    -   地形开关
    -   DEM 高程查询
    -   `map.analysis` 容器入口
-   `minemap-business-visibility-analysis`
    -   `Viewshed3D`
    -   `Sightline`
-   `minemap-business-flood-and-shadow`
    -   `FloodAnalysis`
    -   `FloodAnalysisOffline`
    -   `ShadowAnalysis`
-   `minemap-business-measurement`
    -   `Measurement3D`
-   `minemap-business-spatial-analysis-suite`
    -   `setTerrainAnalysis(...)`
    -   `CutFill`
    -   `Excavation`
    -   `LimitHeight`
    -   `ViewDome`
    -   `Interference`
    -   `InterferenceConjoined`
    -   `Profile`

文档策略：

-   高频业务对象单独拆 skill（如 `Viewshed3D`、`Sightline`、`FloodAnalysis`）
-   其余分析类先挂在本 skill 统一说明
-   真正落业务前，再根据 demo 和源码决定是否继续独立拆包

## Common Patterns

### 批量清理

切换项目/场景时统一 `removeAll()`，避免分析对象残留。

### 分析对象归档管理

-   `Highlight` / `CutFill` / `Excavation` 更偏地形或对象表达修饰
-   `ViewDome` / `Interference` / `InterferenceConjoined` 更偏视场与干扰表达
-   `Profile` / `Player` 更偏剖切、播放或过程辅助

推荐做法：按业务模块维护自己的分析对象引用，并同步挂入 `map.analysis.*Collection`，不要创建后失去控制句柄。

### 判断当前是否已启用 terrain

```javascript
if (map.isTerrain()) {
	console.log("terrain enabled");
}
```

### 切回平面模式

```javascript
map.setTerrain({ tiles: null });
```

虽然实现层面对 falsy 也有兼容，但文档化写法推荐保持显式。

## Performance Tips

-   大范围地形+分析叠加时，优先降级实时分析频率
-   对离线分析（如离线淹没）优先预计算输入
-   分析阶段暂时禁用不必要后处理（SSR/OIT/阴影）
-   没有高精度查询需求时，优先用 `queryDEMHeightByPoint()` 而不是总是走 `queryAccurateDEMHeightByPoint()`
-   `queryVisibleRegionTerrainTiles()` 依赖额外区域索引服务；按需调用并配置缓存 / 网络策略，不要对每帧相机变化都重复请求
-   `availabilities` 能配就配，尤其是高精度查询和大范围分析
-   `terrainSkirtHeight` 优先保留动态值，除非你明确在调接缝问题
-   `TerrainLeveling` 依赖 WebGL2，不要把它当成所有终端都能稳定开启的通用效果
-   terrain 不是轻量开关，不要在高频 UI 操作中反复切换

## Official Docs And Example Lookup

terrain 相关资料建议交叉看：

-   官网与帮助入口：`https://minedata.cn/`
-   3D Ultra 开发指南：`https://minedata.cn/nce-support/guide-3D-Ultra`
-   3D Ultra API 参考：`https://minedata.cn/nce-support/api-3D-Ultra`
-   3D Ultra 示例中心：`https://minedata.cn/nce-support/demoCenter-3D-Ultra`

本地 demo 里和 terrain 强相关的证据源包括：

-   `demo/html/QueryDEMHeight.html`
-   `demo/html/DemAnalysis.html`
-   `demo/html/TerrainLeveling.html`
-   `demo/html/MarkerDraggableOnTerrain.html`
-   `demo/html/RasterTerrainChange.html`
-   `demo/html/TerrainProvidersSwitch.html`
-   `demo/html/RegionTerrainVisibleTiles.html`

查 terrain 时推荐顺序：

1. 先看示例中心和本地 demo，确认能力范围
2. 再看 API 参考，确认方法名
3. 最后回 `source/api/map.js`、`source/data-source/source-cache/TerrainTileSource.js` 看真实行为

## See Also

-   `minemap-official-resources-and-onboarding`
-   `minemap-fundamentals`
-   `minemap-2d-3d-overlay-and-classification`
-   `minemap-marker-and-popup`
-   `minemap-primitives-and-materials`
-   `minemap-business-visibility-analysis`
-   `minemap-business-flood-and-shadow`
-   `minemap-business-measurement`
-   `minemap-business-spatial-analysis-suite`
-   `minemap-performance-and-backend`
