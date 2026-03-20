---
name: minemap-terrain-and-analysis
description: MineMap 地形设置（setTerrain）、DEM 高程查询、分析对象容器（可视域/通视/挖填方/淹没等）。
---

# MineMap Terrain and Analysis

## Quick Start

```javascript
map.setTerrain({
	tiles: ["https://example.com/dem/{z}/{x}/{y}.terrain"],
	tileSize: 512,
	maxzoom: 14,
	extensions: {
		requestVertexNormals: false,
		requestWaterMask: false,
		requestHeightFieldNormals: false
	}
});
```

## Core Concepts

### 1) 地形入口

`map.setTerrain(options)` 是 MineMap 4.x 推荐 DEM 接口。

-   `tiles: null` 可回退平面模式
-   地形切换后，引擎会重置相关 quadtree 缓存并更新代理 source 顺序

### 2) 高程查询

常见查询：

-   `queryDEMHeightByPoint(...)`
-   `queryAccurateDEMHeightByPoint(...)`

使用建议：地图稳定后（`idle`）再做高频批量查询。

### 3) 分析容器

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

### 4) `index.js` 已导出的其他分析类

除了可视域、通视、淹没、阴影，公开导出里还包括：

-   `Highlight`
-   `CutFill`
-   `Excavation`
-   `ViewDome`
-   `Interference`
-   `InterferenceConjoined`
-   `Profile`
-   `Player`

### 5) 空间分析 skill 拆分结果

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

### 添加可视域对象

```javascript
const viewshed = new minemap.Viewshed3D({
	/* params */
});
map.analysis.viewshed3DCollection.add(viewshed);
```

### 批量清理

切换项目/场景时统一 `removeAll()`，避免分析对象残留。

### 分析对象归档管理

-   `Highlight` / `CutFill` / `Excavation` 更偏地形或对象表达修饰
-   `ViewDome` / `Interference` / `InterferenceConjoined` 更偏视场与干扰表达
-   `Profile` / `Player` 更偏剖切、播放或过程辅助

推荐做法：按业务模块维护自己的分析对象引用，并同步挂入 `map.analysis.*Collection`，不要创建后失去控制句柄。

## Performance Tips

-   大范围地形+分析叠加时，优先降级实时分析频率
-   对离线分析（如离线淹没）优先预计算输入
-   分析阶段暂时禁用不必要后处理（SSR/OIT/阴影）

## See Also

-   `minemap-fundamentals`
-   `minemap-2d-3d-overlay-and-classification`
-   `minemap-business-visibility-analysis`
-   `minemap-business-flood-and-shadow`
-   `minemap-business-measurement`
-   `minemap-business-spatial-analysis-suite`
-   `minemap-performance-and-backend`
