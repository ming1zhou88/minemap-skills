# minemap-terrain-and-analysis 参数与模式参考

按需查关键词：provider/format/Cesium/ArcGIS 对应外部地形；terrainSkirtHeight、minHeightToDemSurface、enableUnderGround 对应地下和相机；queryDEM/queryAccurateDEM 对应高程；queryVisibleRegionTerrainTiles 对应区域元数据查询；TerrainLeveling 对应平整。主入口包含选型与分析容器，需配合阅读。

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
tiles: ["https://example.com/dem/{z}/{r}/{c}/{z}_{x}_{y}.terrain"];
```

语义：

-   这是 DEM 几何数据，不是影像地址
-   `tiles: null` 表示回退到平面/椭球地表

源码实现上，传 falsy 也会退回 ellipsoid；但稳定文档写法仍建议显式用：

```javascript
map.setTerrain({ tiles: null });
```

### 外部 Cesium / ArcGIS 地形服务

4.27.1 的 `setTerrain()` 支持保留默认 MineMap 格式，也可显式选择外部 provider：

```javascript
// Cesium quantized-mesh：tiles 使用服务返回的瓦片 URL 模板
map.setTerrain({
	provider: "cesium",
	format: "quantized",
	tiles: [cesiumTerrainTileTemplate],
	scheme: "tms",
	zoomOffset: -1,
	tileSize: 512,
	maxzoom: 15
});

// ArcGIS ImageServer：SDK 根据 url 请求对应地形瓦片
map.setTerrain({
	provider: "arcgis",
	format: "lerc",
	url: arcGisImageServerUrl,
	tileSize: 256,
	gridSize: 65,
	maxzoom: 16
});
```

Cesium 服务需由应用按其账号与授权方式取得可用的瓦片 URL；ArcGIS 需要可访问的 ImageServer 地址。不要复制 demo 中的临时 token。`provider` 缺省时仍使用 MineMap 既有地形格式；`format` 应与来源匹配（ArcGIS 为 `lerc`，Cesium / MineMap 为 `quantized`）。外部服务的 `maxParallelRequests` 可用于降低首次加载压力，但也会降低同时取瓦片的速度。

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

地下模式。**注意**：`enableUnderGround` 在主分支上是一个**属性赋值**，不是方法，调用方式与 `enableLight` / `enableClouds` 不同。

来源：`source/api/map.js` line 4999+。

**两种开启方式**：

```javascript
// 方式 A：构造选项（推荐一次性开启）
var map = new minemap.Map({
	container: "map",
	style,
	enableUnderGround: true, // 地下模式
	position: [108.96, 34.21, 338],
	pitch: 90,
	maxZoom: 100, // 地下模式要尽量大于 24
	logDepth: true // 必须配 logDepth
});

// 方式 B：运行时动态切换
map.enableUnderGround = true; // 开启
map.enableUnderGround = false; // 关闭
```

**开启后行为**：

-   忽略地形碰撞检测
-   相机可以进入地下（position 高度为负）
-   引擎自动隐藏天空大气（大气透视与太阳盘等自动关闭）
-   地形层（`raster` / 影像）继续渲染，需要显式关闭 `depth-test` 避免被地形遮挡

**配套设置**：

| 配套项              | 推荐值  | 说明                                             |
| ------------------- | ------- | ------------------------------------------------ |
| `logDepth`          | `true`  | 源码注释：否则进入地下后地形会闪烁（z-fighting） |
| `maxZoom`           | `>= 24` | 地下模式要求 maxZoom 至少大于 24，否则可能报错   |
| `pitch`             | `>= 80` | 高俯仰角更利于观察地下场景                       |
| raster `depth-test` | `false` | 影像层在地下时需关闭深度测试，否则会被地形遮住   |

**示例**（参考 `demo/html/Underground.html`）：

```javascript
var map = new minemap.Map({
	container: "map",
	style,
	position: [108.96, 34.21, 338],
	bearing: 90,
	pitch: 90,
	roll: 0,
	maxZoom: 100,
	minZoom: 1,
	projection: "MERCATOR",
	logDepth: true,
	enableUnderGround: true
});

map.enableLight();
map.setTerrain({
	tiles: ["https://.../dem/{z}/{r}/{c}/{z}_{x}_{y}.terrain"],
	tileSize: 512,
	maxzoom: 14
});

map.on("load", function () {
	map.addSource("IMAGERY_SOURCE", {
		type: "raster",
		tiles: ["https://.../satellite/{z}/{r}/{c}/{z}_{x}_{y}.jpg"],
		tileSize: 512
	});
	map.addLayer({
		id: "IMAGERY",
		type: "raster",
		source: "IMAGERY_SOURCE",
		minzoom: 0,
		maxzoom: 23,
		"depth-test": false, // 地下时影像不能被地形挡住
		paint: { "raster-opacity": 1 }
	});

	// 地下管廊 / 隧道 / 设施
	map.addSceneComponent({
		id: "pipeline",
		type: "3d-model",
		data: "../data/UndergroundTube/pipeline.gltf",
		modelFolder: "../data/UndergroundTube/",
		rotation: [90, 0, 0],
		position: [108.965, 34.218, 360] // z 大于 0 仍然显示为"在地下"
	});
});
```

**注意事项**：

-   `enableUnderGround` 是**属性赋值**，不是 `enableUnderGround()` 方法调用；`map.enableUnderGround()` 会报错
-   `position` 第三项（z/alt）填负值也可以工作，但源码注释提醒：未开启 `enableUnderGround` 时传负高度，会被定位到 `[0, 0, 6378137*6]`
-   地下模式下 `Sky` 自动隐藏，相机飞到地下不会渲染天空盒

**适用场景**：

-   管廊 / 隧道 / 地下空间可视化
-   地下停车场 / 地铁站 / 综合管沟
-   数字孪生中的"透视"或"剖切"效果

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

## 按相机视野查询行政区划地形瓦片

`map.queryVisibleRegionTerrainTiles()` 读取省级区划索引和区域高程元数据，再结合当前相机视野筛选瓦片。它需要与该接口配套的数据服务，不会从任意 `setTerrain()` provider 自动推导完整区域高程索引。

```javascript
const result = await map.queryVisibleRegionTerrainTiles({
	baseUrl: "https://your-domain.example/region/",
	outputZ: 15,
	maxDistance: 10000,
	bufferDistance: 100
});

console.log(result.visibleTiles);
```

`baseUrl` 应指向由应用部署或获准访问的区域数据服务根目录。当前源码有仅供开发使用的默认服务地址；面向外部开发者的代码必须显式配置自己的 `baseUrl`。接口返回 `inputTiles`、`visibleTiles` 与 `meta`，可选 `withDetails` 返回筛选明细；查询失败时应显示服务和网络错误，不要把空数组解释成区域没有地形。

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
