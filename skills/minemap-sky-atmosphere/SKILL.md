---
name: minemap-sky-atmosphere
description: MineMap 天空大气（Sky / SkyAtmosphere）专题。覆盖 map.getSky / enableSkyAtmosphere / disableSkyAtmosphere、瑞利散射、米散射、臭氧分布、太阳盘、大气透视 LUT 与天空盒纹理。
---

# MineMap Sky and Atmosphere

## 概述

MineMap 公开的"天空 / 大气"有两条独立路径，**业务上不要混用**：

1. **云与体积大气**（云层、CloudLayer）
    - 见 `minemap-clouds-and-atmosphere`
    - 入口：`map.enableClouds()` / `map.getClouds()`
2. **天空大气（SkyAtmosphere）**（本技能）
    - 入口：`map.getSky()` / `map.enableSkyAtmosphere()` / `map.disableSkyAtmosphere()`
    - 形态：物理大气 + 太阳盘 + 天空盒

> 这是两条独立路径。云用 "Cloud / atmosphere effect" 入口；天空大气用 `getSky()` + `enableSkyAtmosphere()` 入口。

## Quick Start

对应 demo：`demo/html/SkyAtmosphere.html`

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
	position: [108.94, 34.31, 442],
	pitch: 85.5,
	bearing: 53,
	maxZoom: 22,
	minZoom: 1
});
map.repaint = true;

map.enableLight();
map.enableSkyAtmosphere();

map.on("load", function () {
	/* 业务图层加载 */
});

// 业务上做时间联动
map.setSunLightTime({ month, day, hours, minutes });
```

`getSky()` 返回 `Sky` 对象，可以继续访问大气参数（见下表）。

## `Sky` 公开参数

| 参数                            | 说明                                                       |
| ------------------------------- | ---------------------------------------------------------- |
| `useAerialPerspectiveLUT`       | 是否用 LUT 渲染大气透视（默认 `true`）                     |
| `useSkyViewLUT`                 | 是否用 LUT 渲染天空（默认 `true`）                         |
| `useAerialPerspective`          | 是否启用大气透视效果（默认 `false`）                       |
| `useSunDisk`                    | 是否渲染日面（默认 `true`）                                |
| `groundHeight`                  | 地表参考高度，单位米；`setProps({ groundHeight })` 写入    |
| `atmosphereHeight`              | 大气层顶高；`setProps({ atmosphereHeight })`               |
| `rayleighDistribution`          | 瑞利分布尺度（`meters` 数量级）                            |
| `mieDistribution`               | 米散射分布尺度                                             |
| `ozoneDistribution`             | 臭氧分布 `Vector2`，**两段高度**，分别对应 `setProps` 分量 |
| `rayleighScatteringCoefficient` | 瑞利散射系数 `Vector3(R, G, B)`                            |
| `mieScatteringCoefficient`      | 米散射系数                                                 |
| `mieAbsorptionCoefficient`      | 米散射吸收系数                                             |
| `ozoneAbsorptionCoefficient`    | 臭氧吸收系数 `Vector3(R, G, B)`                            |

> 只读字段直接赋值无效，**必须** `skyAtmosphere.setProps({ ... })`。

## 启用 / 关闭

```javascript
map.enableSkyAtmosphere(); // 开启
map.disableSkyAtmosphere(); // 关闭
const sky = map.getSky(); // 拿 Sky 对象调参
```

源码入口在 `source/api/map.js`：

-   `enableSkyAtmosphere()` → `painter.useSkyAtmosphere = true`
-   `disableSkyAtmosphere()` → `painter.useSkyAtmosphere = false`
-   `getSky()` → `this.style.sky`

## 时间联动（太阳盘 / 太阳光）

`SkyAtmosphere` 的太阳位置由 `SunLight` 提供，**不**自己持有。

业务推荐路径：

```javascript
map.enableLight();                                    // 1. 开启光源
map.setSunLightTime({ year, month, day, hours, ... }); // 2. 设时间
const { sunLight } = map.getLights();
sunLight.castShadow = true;                            // 3. 投影
sunLight.intensity = 1.5;                              // 4. 强度
```

详见 `minemap-lighting-and-shadows`。

## 大气透视（aerial perspective）

`useAerialPerspectiveLUT` 是 LUT 路径，`useAerialPerspective` 是"开启真实大气透视"开关，**两者不是一回事**。

-   `useAerialPerspectiveLUT = true` + `useAerialPerspective = false`：天空 / 远景用 LUT 着色，但不叠加透视衰减
-   `useAerialPerspectiveLUT = true` + `useAerialPerspective = true`：在 LUT 基础上叠加真实大气透视
-   `useAerialPerspectiveLUT = false`：关闭 LUT，纯用基础大气

业务上"远山泛蓝"是 `useAerialPerspective = true`；"天空一片均匀"是 `useAerialPerspective = false`。

## Common Patterns

### 模式 1：日出 / 黄昏配色

```javascript
const sky = map.getSky();
sky.setProps({
	rayleighScatteringCoefficient: new minemap.Math.Vector3(2e-5, 4e-5, 8e-5),
	mieScatteringCoefficient: 1e-5,
	mieAbsorptionCoefficient: 8e-6
});
```

低高度角（早晨 / 黄昏）时 R/G/B 比例偏暖；高高度角（正午）时偏冷。

### 模式 2：火星 / 异星大气

```javascript
sky.setProps({
	groundHeight: 5e6,
	atmosphereHeight: 1e7,
	rayleighDistribution: 1e4,
	mieDistribution: 5e3,
	rayleighScatteringCoefficient: new minemap.Math.Vector3(1e-4, 5e-5, 2e-5)
});
```

`rayleighDistribution` / `mieDistribution` 越小，大气层越"薄"。

### 模式 3：天空盒贴图

```javascript
map.getSky().loadSkyBoxTextures(painter, skyboxUrls);
map.getSky().removeSkyBoxTextures(options);
```

业务上一般不需要碰这俩，引擎默认会从 `style.sprite` / `style.glyphs` 推。

## Strict Constraints

### 1. 大气参数必须用 `setProps`

直接 `skyAtmosphere.rayleighScatteringCoefficient = ...` 大概率**不生效**。源码用 `setProps({...})` 统一更新材质 Uniform。

### 2. `Sky` 与 `Clouds` 互相独立

-   `enableSkyAtmosphere()` 不开云，云层用 `enableClouds()`
-   两者可以同时开；视觉上叠加（蓝天 + 云海）
-   业务上想"不要云、只要大气" → 只 `enableSkyAtmosphere()`

### 3. 太阳盘 / 太阳光联动

`useSunDisk` 控制**日面**是否渲染；太阳**光**方向来自 `SunLight`，由 `setSunLightTime` 控制。两者独立。

### 4. `enableSkyAtmosphere` 后必须 `enableLight`

只开大气不开光源，远景不会有阳光感；推荐顺序：

```javascript
map.enableLight();
map.enableSkyAtmosphere();
```

### 5. `useAerialPerspectiveLUT` 默认是 `true`

不要无脑设成 `false`，那样天空颜色会变平。

### 6. `ozoneDistribution` 是 `Vector2`

`setProps({ ozoneDistribution: new Vector2(height, width) })`，**两段独立**。`SkyAtmosphere.html` 的 GUI 写法是先取 `ozoneDistribution[0]` / `[1]` 再重组。

## Failure Cases

-   天空全黑 → 没 `enableLight` / `setSunLightTime`
-   改 `rayleighScatteringCoefficient` 不生效 → 没用 `setProps`
-   日面（白圆盘）不见了 → `useSunDisk = false`
-   远景一片均匀没泛蓝 → `useAerialPerspective = false`
-   大气效果和地形不协调 → `groundHeight` 数值不匹配实际地形
-   进入地下后天空消失 → `enableUnderGround` 时大气已自动隐藏（引擎内置），不是 bug

## Demo References

-   `demo/html/SkyAtmosphere.html`：完整调试面板（瑞利 / 米 / 臭氧 / 透视 / 日面 / 大气高度）
-   `demo/html/PMREMSkyAtmosphere.html`：SkyAtmosphere + PMREM 环境贴图组合

## See Also

-   `minemap-clouds-and-atmosphere`：云与体积大气（独立路径）
-   `minemap-lighting-and-shadows`：`enableLight` / `setSunLightTime` / `SunLight`
-   `minemap-pmrem-and-environment`：`map.environment` 全局光照环境贴图
-   `minemap-terrain-and-analysis`：`enableUnderGround` 与 `setTerrain` 联动
