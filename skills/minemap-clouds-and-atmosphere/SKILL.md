---
name: minemap-clouds-and-atmosphere
description: MineMap 云与大气渲染链。用于启用体积云（多层 CloudLayer）、单片实例化云（map.cloudCollection）、大气散射与太阳 / 天空辐照度，重点说明主分支公开参数与禁用字段。
---

# MineMap Clouds and Atmosphere

## 概述

MineMap 主分支公开的云与大气能力有两条独立路径，**业务上不要混用概念**：

1. **体积云（Volumetric Clouds）**
    - 入口：`map.enableClouds()` / `map.disableClouds()` / `map.getClouds()`
    - 形态：覆盖全场景的大气层，包含多层 `CloudLayer`
    - 适合：天空远景、宏观云海、随时间飘动的大气效果
2. **单片实例化云（CloudCollection）**
    - 入口：`map.cloudCollection.add({ ... })`
    - 形态：局部放置的单体云（积云）
    - 适合：景点 / 园区上方放置少量写实云朵

> 这两条路径是引擎里**早就存在**的成熟能力，并非 4.25 专属新增，但 4.25 仍是主分支当前推荐使用方式。

## Quick Start：体积云

```javascript
var style = {
	glyphs: "minemap://fonts/{fontstack}/{range}",
	sprite: "minemap://sprite/sprite",
	sources: {},
	layers: [],
	version: 8
};

var map = new minemap.Map({
	container: "map",
	style,
	position: [116.038, 40.105, 2944],
	pitch: 80.96,
	bearing: -27.36,
	projection: "MERCATOR"
});

map.enableLight();
map.enableClouds();
map.repaint = true;

map.setTerrain({
	tiles: ["https://.../dem/{z}/{r}/{c}/{z}_{x}_{y}.terrain"],
	availabilities: ["https://.../dem/layer.json"],
	tileSize: 512,
	maxzoom: 14
});

map.on("load", function () {
	map.addSource("IMAGERY", {
		type: "raster",
		tiles: ["https://.../satellite/{z}/{x}/{y}"],
		tileSize: 512
	});
	map.addLayer({
		id: "IMAGERY",
		type: "raster",
		source: "IMAGERY",
		minzoom: 0,
		maxzoom: 23
	});

	const clouds = map.getClouds();
	clouds.coverage = 0.3;
	clouds.localWeatherVelocity = [0.005, 0]; // 飘动；设成 [0, 0] 关闭动画
});
```

## Quick Start：单片实例化云

对应 demo：`demo/html/CloudCollection.html`

```javascript
const clouds = [];

for (let i = 0; i < 20; i++) {
	clouds[i] = map.cloudCollection.add({
		position: minemap.Math.Vector3.fromDegrees(
			115.9733 + 0.1 * (Math.random() * 2 - 1),
			40.20299 + 0.1 * (Math.random() * 2 - 1),
			1300
		),
		scale: new minemap.Math.Vector2(3000 + 1000 * Math.random(), 500 + 100 * Math.random()),
		maximumSize: new minemap.Math.Vector3(50, 15, 13),
		slice: Math.max(Math.min(Math.random(), 0.7), 0.2) // 0.2 ~ 0.7
	});
}

// 整体显隐
map.cloudCollection.show = false;

// 删除全部
map.cloudCollection.removeAll();
```

## Architecture Positioning

### 体积云

-   入口是 `Map` 实例的 `enableClouds()` / `disableClouds()` / `getClouds()`
-   `getClouds()` 返回的是 `CloudsEffect` 对象，包含全部可调参数
-   与 `enableLight` 互相独立但通常一起开；不开启光照时云会失去明暗对比
-   体积云依赖 `setTerrain` 提供可投阴影的地形，否则云层与地面无视觉交互

### 单片实例化云

-   入口是 `map.cloudCollection`（`CloudCollection` 实例）
-   `add(options)` 返回一个 `CumulusCloud` 实例，支持逐个控制位置 / 缩放 / 切片 / 颜色 / 亮度
-   适合"放置若干朵云在场景固定位置"的局部表达
-   自身不做体积积分，性能上比体积云便宜

## 体积云 `CloudsEffect` 公开参数

| 参数                    | 说明                                                                |
| ----------------------- | ------------------------------------------------------------------- |
| `coverage`              | 云层覆盖度（0..1），值越大覆盖场景越大                              |
| `localWeatherVelocity`  | 云的飘动速度 `[u, v]`；设成 `[0, 0]` 关闭动画                       |
| `temporalUpscale`       | 时间上采样；开启可提升性能，但当前阶段噪点和透明度过滤较粗          |
| `shapeDetail`           | 启用形状细节纹理（更细腻 / 更贵）                                   |
| `turbulence`            | 启用湍流纹理（更真实的云底起伏 / 更贵）                             |
| `scatteringCoefficient` | 散射系数；越大明暗对比越强                                          |
| `absorptionCoefficient` | 吸收系数；越大越暗                                                  |
| `scatterAnisotropy1`    | 双叶 HG 相位函数第一项                                              |
| `scatterAnisotropy2`    | 双叶 HG 相位函数第二项                                              |
| `scatterAnisotropyMix`  | 两个相位函数的混合比例                                              |
| `skyIrradianceScale`    | 天空辐照度缩放                                                      |
| `powderScale`           | Beer-Powder 效果强度                                                |
| `powderExponent`        | Beer-Powder 曲线斜率                                                |
| `cloudLayers`           | 云层集合（默认 4 层），可通过索引 `clouds.cloudLayers[i]` 访问      |
| `lightShafts`           | 阴影长度 / 光投射相关；**主分支默认 `false`（源码注释："先关掉"）** |

### CloudLayer（云层）

每层独立可调：

| 参数                  | 说明                                                                       |
| --------------------- | -------------------------------------------------------------------------- |
| `altitude`            | 海拔高度（米）                                                             |
| `height`              | 厚度（米）；设为 0 即可关闭该层                                            |
| `densityScale`        | 密度缩放；设为 0 等价于关闭                                                |
| `shapeAmount`         | 形状纹理影响系数                                                           |
| `shapeDetailAmount`   | 形状细节纹理影响系数                                                       |
| `weatherExponent`     | 天气图渐变锐度                                                             |
| `shapeAlteringBias`   | 云形状垂直偏移（1 对称，0 底部完全偏移）                                   |
| `coverageFilterWidth` | 天气信号影响形状改变密度的过渡宽度                                         |
| `densityProfile`      | 密度随高度的分布（`expTerm` / `exponent` / `linearTerm` / `constantTerm`） |

### 高级参数（位于 `clouds.cloudsPass.currentMaterial`）

`multiScatteringOctaves`、`accurateSunSkyIrradiance`、`accuratePhaseFunction`、`maxIterationCount`、`minStepSize` / `maxStepSize`、`maxRayDistance`、`perspectiveStepScale`、`minDensity` / `minExtinction` / `minTransmittance`、`maxIterationCountToSun` 等。

这些都属于材质层高级参数，主分支 demo 的 `Clouds.html` 已经把它们放进 GUI 调试面板，对照参考即可。

### 关于 `lightShafts`

主分支源码里 `lightShafts` 字段确实存在，但：

-   默认值是 `false`
-   源码注释："用于控制阴影长度，进而控制光照投射效果。先关掉"
-   主分支 `Clouds.html` 没有把它放进调试 GUI
-   与之配套的阴影长度 / 雾霭 / 地面辐照度等参数在 demo 中也是**注释掉**的状态

所以在主分支上，**不要把 `lightShafts` 当成稳定的可调效果**。它和"阴影长度""雾霭""丁达尔 / 太阳光柱"等扩展效果目前在主分支上都没有公开推荐路径。

## 单片实例化云 `CloudCollection` 公开 API

```javascript
const cloud = map.cloudCollection.add({
	position, // Vector3，单体云的位置
	scale, // Vector2，X 宽度 / Y 高度（米）
	maximumSize, // Vector3，云朵最大尺寸（决定形状）
	slice, // number，0..1，控制沿 z 方向的横截面位置
	color, // Color
	brightness, // number，默认 1
	show // boolean，默认 true
});
```

`CumulusCloud` 实例属性（可读可写）：

-   `cloud.position` / `cloud.scale` / `cloud.maximumSize` / `cloud.slice` / `cloud.color` / `cloud.brightness` / `cloud.show`

`CloudCollection` 实例方法：

-   `add(options)`：新增单体云
-   `remove(cloud)`：移除指定云
-   `removeAll()`：移除全部
-   `contains(cloud)`：判断是否属于该集合
-   `get(index)`：按索引获取
-   `map.cloudCollection.show`：整集合显隐

## 时间与光照联动

-   `map.setSunLightTime({ year, month, day, hours, minutes, seconds })`
-   `map.setSunLightTimestamp(date.getTime())`
-   推荐把 `enableLight` 和 `setSunLightTime` 放在 `load` 之后立即执行，避免首帧黑天

## Common Patterns

### 模式 1：体积云 + 飘动

对应 demo：`demo/html/Clouds.html`

```javascript
map.enableLight();
map.enableClouds();
map.setTerrain({...});

map.on("load", () => {
	const clouds = map.getClouds();
	clouds.coverage = 0.3;
	clouds.localWeatherVelocity = [0.005, 0];
});
```

### 模式 2：多云层密度配置

```javascript
const clouds = map.getClouds();
// 把第 1 层关掉
clouds.cloudLayers[1].height = 0;
clouds.cloudLayers[1].densityScale = 0;
// 把第 2 层做成"卷云"
clouds.cloudLayers[2].altitude = 7500;
clouds.cloudLayers[2].height = 500;
clouds.cloudLayers[2].densityScale = 0.003;
clouds.cloudLayers[2].shapeAmount = 0.4;
clouds.cloudLayers[2].shapeDetailAmount = 0;
```

### 模式 3：单片云按需摆放

对应 demo：`demo/html/CloudCollection.html`

```javascript
const cloud = map.cloudCollection.add({
	position: minemap.Math.Vector3.fromDegrees(116.0, 40.1, 1300),
	scale: new minemap.Math.Vector2(3000, 500),
	maximumSize: new minemap.Math.Vector3(50, 15, 13),
	slice: 0.4,
	brightness: 1.0
});
```

### 模式 4：单片云做东西向飘动

demo 通过 `requestAnimationFrame` + `eastNorthUpToFixedFrame` 计算当前位置的 East 单位向量，然后按时间步长平移 `cloud.position`，从而实现"东西向飘动"效果。

```javascript
function stepCloudAnimation(timestamp) {
	const dtSec = (timestamp - lastTime) / 1000;
	const d = SPEED * dtSec;
	for (const cloud of clouds) {
		const east = getEastUnitVectorAtCartesian(cloud.position);
		cloud.position = new minemap.Math.Vector3(
			cloud.position.x + east.x * d,
			cloud.position.y + east.y * d,
			cloud.position.z + east.z * d
		);
	}
	requestAnimationFrame(stepCloudAnimation);
}
```

业务上可以参考这个写法做自定义动画。

## Strict Constraints

### 1. 体积云 + 单片云是两条独立路径

不要混用：

-   体积云通过 `getClouds()` 拿参数
-   单片云通过 `map.cloudCollection.add()` 增加实例
-   两者都"开"的话会同时渲染；视觉上不会冲突，但管理时要分清哪部分是全局大气、哪部分是局部积云

### 2. 体积云必须配合 `enableLight`

只 `enableClouds()` 不 `enableLight()`，云会失去明暗对比，呈现"灰团"。

### 3. 关闭体积云的推荐方式

业务上想隐藏云，推荐用 `clouds.coverage = 0` 或把所有层 `densityScale = 0` / `height = 0`，而**不是** `disableClouds()`，后者会拆掉整个 pass，重新打开有重建成本。

### 4. 停止体积云动画

```javascript
clouds.localWeatherVelocity = [0, 0];
```

不要用 demo 里没公开过的 `enableAnimate` 之类私有变量。

### 5. `lightShafts` 默认关闭

主分支默认 `false`，**不**对应"丁达尔 / 太阳光柱 / crepuscular rays"等用户可见效果。把 `lightShafts` 视作内部调试字段，而不是业务上的可调功能。

### 6. 单片云 `slice` 范围

demo 推荐 `0.2 ~ 0.7`。低于 0.2 横截面过小会露出椭球体边缘；高于 0.7 云团会显得过小。

### 7. 单片云 `position` 高度

`position.z`（或 Vector3.fromDegrees 的第三参数）是海拔高度（米）。要云停留在地表附近就用几十到几百米；要做"高空云"就放到几千米。

## Failure Cases

-   体积云变全黑 / 灰团 → 没 `enableLight` / `setSunLightTime`
-   体积云开销过大，帧率崩 → 调小 `multiScatteringOctaves` / `maxIterationCount`，关闭 `turbulence` / `shapeDetail`
-   单片云不显示 → 检查 `slice` 是否在 0.2~0.7 区间、`position.z` 高度是否合理、`map.cloudCollection.show` 是否为 `true`
-   单片云形变异常 → `maximumSize` 设得过小，或 `slice` 越界
-   体积云看不到与地面关系 → 没 `setTerrain` 加载地形

## Demo References

-   `demo/html/Clouds.html`：体积云 + 大气散射 + 调试 GUI（覆盖度 / 时间 / 形状细节 / 散射 / 多重散射 / 步长 / 早退阈值 / 多云层参数等）
-   `demo/html/CloudCollection.html`：局部单体实例化云（随机生成 20 朵 + 显隐 + 东西向飘动 + 全部删除）

## See Also

-   `minemap-scene-components`
-   `minemap-lighting-and-shadows`
-   `minemap-terrain-and-analysis`
