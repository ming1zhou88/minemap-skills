---
name: minemap-terrain-and-analysis
description: MineMap 地形设置（setTerrain）、DEM 高程查询、地形分析、TerrainLeveling、相机碰撞高度与 terrain 相关约束。
---

# MineMap Terrain and Analysis

## Quick Start

```javascript
const map = new minemap.Map({
    container: "map",
    style,
    position: [89.11, 27.40, 15000],
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

## `setTerrain()` Parameter Surface

### 最核心参数

-   `tiles: string[] | null`
-   `tileSize: number`
-   `maxzoom: number`
-   `availabilities: string[] | null`
-   `extensions`

### `tiles`

terrain 瓦片 URL 模板。

常见形式是 `.terrain` 文件：

```javascript
tiles: ["https://example.com/dem/{z}/{r}/{c}/{z}_{x}_{y}.terrain"]
```

语义：

-   这是 DEM 几何数据，不是影像地址
-   `tiles: null` 表示回退到平面/椭球地表

源码实现上，传 falsy 也会退回 ellipsoid；但稳定文档写法仍建议显式用：

```javascript
map.setTerrain({ tiles: null });
```

### `tileSize`

DEM 瓦片大小。

MineMap 4.x terrain 相关示例和源码默认使用：

-   `512`

建议：

-   和实际服务一致
-   未确认前优先按 `512` 组织

### `maxzoom`

DEM 数据的最高请求层级。

源码注释明确给出了经验值：

-   30 米 DEM 常见 `14`
-   12.5 米 DEM 常见 `16`

注意：

-   超过这个 zoom，内部会走上采样，不再继续请求更细 DEM
-   如果你同时配置了 `availabilities`，源码注释说明 `maxzoom` 不再是最高精度遍历的主要依据

### `availabilities`

地形索引文件列表，常见是 `layer.json`。

它的价值很大：

-   用于更高精度的地形可用性判断
-   `queryAccurateDEMHeightByPoint()` 想拿到“最高精度”结果时，推荐配置它
-   配置后，terrain 最细级别的判断不再单纯依赖 `maxzoom`

推荐：

-   只要服务提供 `layer.json`，就加上
-   尤其是需要精确高程查询、沿地形动画、分析工具时

### `extensions`

terrain 扩展对象，源码里可确认的 3 个字段：

-   `requestVertexNormals`
-   `requestWaterMask`
-   `requestHeightFieldNormals`

默认都为 `false`。

#### `requestVertexNormals`

请求顶点法线扩展。

适用：

-   对地表法线、光照细节有额外依赖的场景

#### `requestWaterMask`

请求水面掩膜扩展。

适用：

-   DEM 服务确实提供水面掩膜，并且你有相关渲染需求

#### `requestHeightFieldNormals`

请求区域场法线扩展。

这个在地形分析里尤其有意义。源码和 demo 都表明：

-   坡向/坡度分析想显示更稳定的方向箭头时，可打开它
-   如果 DEM 数据里带区域面法线，箭头方向会更稳定，不容易出现分叉感

### `zeroTileRequestUrl`

这是一个非常值得写进 skill 的“半隐藏参数”。

虽然 `map.setTerrain()` 的当前类型注释里没有完整展开，但：

-   `queryAccurateDEMHeightByPoint()` 的示例里出现了它
-   `TerrainTileSource` 构造函数里也明确支持它

作用：

-   当 0 级瓦片缺失时，单独指定 0 级瓦片目录
-   避免根级 DEM 缺失导致初始化或高精度遍历出错

适合：

-   非标准 terrain 服务目录
-   根层瓦片缺省的数据集

## Terrain-Related Map Parameters

### `terrainSkirtHeight`

地图构造参数和运行时属性都支持它。

语义：

-   地形裙边高度
-   用于减轻 terrain tile 接缝和边缘裂缝问题

规则：

-   `< 0`：按级别动态计算
-   `>= 0`：固定值

源码里动态值的实际逻辑是：

-   `getLevelMaximumGeometricError(z) * 5.0`

建议：

-   默认保留 `-1`
-   只有在明确看到接缝问题、且知道 DEM 服务特性时再手动改

运行时也可直接改：

```javascript
map.terrainSkirtHeight = 80;
```

改完会重新按当前 terrain 配置刷新 terrain source。

### `minHeightToDemSurface`

这个参数非常关键，但经常被忽略。

语义：

-   相机距离 DEM 表面的最小高度
-   用于相机与地表碰撞检测

默认值：

-   `5`

适用：

-   控制相机不要贴地穿模
-   第一人称或低空飞行时限制相机压到地表以下

运行时可直接调：

```javascript
map.minHeightToDemSurface = 5;
```

### `enableUnderGround`

这不是 terrain 参数本身，但和 terrain 强相关。

开启后：

-   会忽略地形碰撞检测
-   相机可以进入地下

源码注释同时提醒：

-   地下模式通常要配合 `logDepth`
-   否则进入地下后可能出现地形闪烁

### `requestInterval`

地图构造参数中的请求重试/等待间隔，源码注释明确写了：

-   当前主要只在 terrain 请求上生效
-   单位秒
-   默认 `60`

适用：

-   需要控制 terrain 重复请求节奏的网络环境

## DEM Query APIs

### 2) 高程查询

常见查询：

-   `queryDEMHeightByPoint(...)`
-   `queryAccurateDEMHeightByPoint(...)`
-   `queryDEMHeightByLngLat(...)`
-   `queryDEMSourceHeightByPoint(...)`

使用建议：地图稳定后（`idle`）再做高频批量查询。

### `queryDEMHeightByPoint(queryGeometry)`

输入：

-   屏幕像素坐标

过程：

-   先做 GPU pick，拿到地表经纬位置
-   如果当前已开启 terrain，再用 `sampleTerrain(...)` 采样高程

返回：

-   经纬高 `Vector3`

特点：

-   同步
-   快
-   精度受当前已加载 terrain 数据影响

适合：

-   点击取高
-   hover 辅助显示
-   一般业务交互

### `queryAccurateDEMHeightByPoint(queryGeometry)`

输入仍然是屏幕像素坐标，但内部会调用：

-   `sampleTerrainMostDetailed(...)`

特点：

-   异步
-   精度更高
-   想拿到最佳结果时，最好配置 `availabilities`

适合：

-   量算前取点
-   精确高程回写
-   编辑结果持久化

### `queryDEMHeightByLngLat(lnglat)`

输入：

-   经纬度

输出：

-   经纬高 `Vector3`

适合：

-   已知经纬点批量查高
-   初始化模型、标注、分析点位时补高度

### `queryDEMSourceHeightByPoint(e, source)`

这是更低层的“指定 DEM source”查询入口。

当前项目里更常用的还是前三个方法；只有你明确在做 DEM source 级控制时再使用它。

## Common Terrain Usage Patterns

### 模式 1：基础地形 + 影像底图

```javascript
map.on("load", () => {
    map.setTerrain({
        tiles: [terrainUrl],
        tileSize: 512,
        maxzoom: 14
    });

    map.addSource("imagery", {
        type: "raster",
        tiles: [imageryUrl],
        tileSize: 512
    });

    map.addLayer({
        id: "imagery",
        type: "raster",
        source: "imagery"
    });
});
```

这是最标准的 terrain 起图模式。

### 模式 2：点击获取精确高程

```javascript
map.on("click", async (e) => {
    const point = await map.queryAccurateDEMHeightByPoint(e.point);
    console.log(point.z);
});
```

前提：

-   已开启 terrain
-   最好配置 `availabilities`

### 模式 3：已知经纬度补地表高程

```javascript
const point = map.queryDEMHeightByLngLat([116.39, 39.9]);
const z = point.z;
```

适合在放模型、marker、分析点之前先拿地表高度。

### 模式 4：terrain 下的拖拽标注

terrain 打开后，`Marker` 拖拽会走 terrain pick。

这意味着：

-   不只是平面位置变了
-   高度也可能自动刷新

适合：

-   山地区域点位编辑
-   设备点落点修正

### 模式 5：terrain 下贴地 primitive

terrain 开启后，如果 primitive 本身支持：

-   `adaptTerrain: true`

就可以让线面对象沿地表起伏。

但这依赖对象本身的 geometry/material 支持，不是 terrain 自动替你完成的。

## Terrain Analysis

### 3) 地形分析入口

`map.setTerrainAnalysis(options)` 是 terrain 栅格分析入口。

当前源码可确认：

-   `property` 支持 `SLOPE`、`ASPECT`、`ELEVATION`
-   `type` 支持 `EXPONENTIAL`、`INTERVAL`、`CATEGORICAL`
-   支持 `base`
-   支持 `stops`
-   支持 `default`
-   支持 `region`
-   支持 `hasArrow`

### `property`

决定分析的地形属性：

-   `SLOPE`：坡度
-   `ASPECT`：坡向
-   `ELEVATION`：高程带

### `type`

颜色插值方式：

-   `EXPONENTIAL`
-   `INTERVAL`
-   `CATEGORICAL`

如果用 `EXPONENTIAL`，可再配：

-   `base`

默认 `base = 1.0`，也就是线性插值感。

### `stops`

分析值到颜色的映射表。

要求：

-   按值递增排列
-   颜色可用 hex / rgb / rgba / 基础 `Color` 名称

### `default`

不命中 `stops` 时的默认颜色。

### `region`

区域分析时的掩膜区域。

不传时：

-   一般按全局 terrain 分析

传入时：

-   只分析指定区域

### `hasArrow`

是否显示方向箭头。

源码注释写的是“目前仅有坡度分析有效”，而 demo 用它表达坡向指示，所以更稳妥的理解是：

-   主要在坡度/坡向类场景下才有意义
-   最终表现还依赖法线扩展数据质量

### 推荐示例

```javascript
map.setTerrainAnalysis({
    type: minemap.InterpolationType.EXPONENTIAL,
    property: minemap.TerrainAnalysisType.SLOPE,
    stops: [
        [0, "#00FF00"],
        [30, "#FFFF66"],
        [60, "#FF8000"],
        [90, "#FF0000"]
    ],
    default: "rgba(0,0,255,1)",
    region: region,
    hasArrow: true
});
```

## `TerrainLeveling`：地形平整

### 原理

`TerrainLeveling` 不是简单的“盖一个面”。

从源码看，它会：

-   接收区域 `region` 和平整深度 `depth`
-   在 `onAdd(map)` 时找出与区域相交的 terrain tile
-   扫描 tile 内 `cartographicList`
-   取区域内最小高程作为 `referenceHeight`
-   给相交 terrain tile 准备 transform feedback buffer
-   再通过 leveling mask material 做地形平整效果

也就是说，它本质是：

-   先求区域参考高程
-   再在 GPU 管线里对相交 terrain 顶点做平整处理

### 参数

-   `region: number[]`：`[lng, lat, lng, lat, ...]`
-   `depth: number`

### 使用方式

它是 scene object，不是 layer，也不是普通 analysis API。

推荐写法：

```javascript
const leveling = new minemap.TerrainLeveling({
    region: [89.1, 27.4, 89.2, 27.4, 89.2, 27.3, 89.1, 27.3],
    depth: 50
});

map.addSceneComponent(leveling);
```

运行时可调：

```javascript
leveling.setDepth(80);
```

### 约束

-   依赖 WebGL2
-   WebGL1 下源码会直接 `console.warn` 并禁用
-   依赖 terrain tile 已经可见/已加载到一定程度
-   区域应尽量规范，避免自交

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
