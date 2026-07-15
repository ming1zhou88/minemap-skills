---
name: minemap-volume-rendering
description: MineMap 3D 体渲染（VolumeShell）场景组件。用于医学、地质、海洋等场景的多时次 / 多数据类型体渲染，叠加 2D 切片堆叠并支持 urlBuilder 动态切换数据源。
---

# MineMap Volume Rendering

## 概述

`VolumeShellSceneComponent` 是 MineMap 4.25 公开的体渲染组件，适合"叠片式 3D 体数据"场景：把一系列 2D 切片（通常是 webp / png / 灰度图）按高度堆叠到 WGS84 椭球内壳，运行时按视线做体积积分。

典型应用：

-   全球海洋温度 / 海流
-   地质含水层 / 矿层
-   大气层 / 等高气压层
-   医学影像（多切片堆叠）

入口：

```javascript
const shell = new minemap.VolumeShellSceneComponent({ ... });
map.addSceneComponent(shell);
```

## Quick Start

```javascript
const timeFolders = ["20251201-00h", "20251201-06h", "20251201-12h", "20251201-18h", "20251202-00h"];

function urlBuilder(timeIndex, dataType, depthIndex) {
	const folder = timeFolders[timeIndex];
	const id = String(depthIndex).padStart(3, "0");
	return `../data/GLOBAL/${folder}/images/${dataType}/depth_${id}_${dataType}.webp`;
}

const initialUrls = [];
for (let i = 0; i < 50; i++) initialUrls.push(urlBuilder(0, "temperature", i));

const shell = new minemap.VolumeShellSceneComponent({
	id: "temperature-shell",
	layerUrls: initialUrls,
	urlBuilder,
	timeFolders,
	timeIndex: 0,
	dataType: "temperature",
	minHeight: 1000,
	maxHeight: 100000,
	heightScale: 2,
	alpha: 0.45,
	mode: 0,
	dataMin: -10,
	dataMax: 40,
	interpolationMode: 1,
	maxTextureBytes: 96 * 1024 * 1024
});

map.addSceneComponent(shell);
```

## Architecture Positioning

-   `VolumeShellSceneComponent` 是一个 `scene-object`，**不是** source/layer
-   内部使用 `Texture3D`，所以对 WebGL 2 / WebGPU 后端有要求
-   通过 `urlBuilder(timeIndex, dataType, depthIndex)` 动态拼 URL，因此天然支持多时间 / 多类型切换
-   数值映射模式（`mode: 1`）会把灰度图按 `dataMin` / `dataMax` 映射到 colormap
-   取色模式（`mode: 0`）直接使用图像本身颜色

## 关键参数

| 参数                | 说明                                                             |
| ------------------- | ---------------------------------------------------------------- |
| `id`                | 唯一标识                                                         |
| `layerUrls`         | 首次加载时的 URL 列表（深度方向）                                |
| `urlBuilder`        | 动态 URL 生成函数：`(timeIndex, dataType, depthIndex) => string` |
| `timeFolders`       | 时间目录列表                                                     |
| `timeIndex`         | 初始时间索引                                                     |
| `dataType`          | 数据子集标识，如 `"temperature"` / `"temperature_rgba"`          |
| `minHeight`         | 最低高度（米）                                                   |
| `maxHeight`         | 最高高度（米）                                                   |
| `heightScale`       | 高度缩放                                                         |
| `alpha`             | 全局不透明度                                                     |
| `mode`              | `0` 取色模式 / `1` 数值映射模式                                  |
| `dataMin`           | 数值模式下，浮点数据最小值                                       |
| `dataMax`           | 数值模式下，浮点数据最大值                                       |
| `interpolationMode` | `1` linear / `0` nearest                                         |
| `maxTextureBytes`   | 纹理最大内存占用，超过会降级                                     |

## 公共方法

```javascript
shell.setDataType("temperature_rgba");
shell.setTimeIndex(2);
shell.setHeightScale(5);
shell.setOpacity(0.6);
shell.setMode(1);
shell.setInterpolation(0);
shell.setDebugMode(true);
shell.setFlipVertical(false);
shell.flipY = true; // 修改后需调用 _initTextures 重新加载（私有，按需使用）
```

## Common Patterns

### 模式 1：全球海洋温度（多时间切换）

对应 demo：`demo/html/VolumeShellTemperature.html`

关键信号：

-   用空 style 起图，加一张卫星底图
-   `timeFolders` 控制可选时间点
-   `urlBuilder` 把每个时间 / 类型 / 深度拼成最终 URL
-   UI 控件通过 `setTimeIndex` / `setDataType` / `setHeightScale` 等方法驱动

### 模式 2：取色 vs 数值映射

```javascript
// 取色模式：用图本身的颜色
shell.setMode(0);

// 数值映射：把灰度按 dataMin/dataMax 映射到 colormap
shell.setMode(1);
shell.dataMin = 0;
shell.dataMax = 100;
```

### 模式 3：性能 / 显存控制

```javascript
// 限制纹理内存占用，超过会自动降级
const shell = new minemap.VolumeShellSceneComponent({
	...
	maxTextureBytes: 96 * 1024 * 1024 // 96MB
});
```

## Strict Constraints

### 1. 必须使用 `addSceneComponent`

`VolumeShellSceneComponent` 不走 source/layer。`addSource({ type: 'volume' })` 之类的写法无效。

### 2. 首次 `layerUrls` 与 `urlBuilder` 要保持一致

`layerUrls` 只是首帧用的初始 URL，**真正能动态切换的**是 `urlBuilder`。如果两者不匹配，切换时间 / 类型会拿到错图。

### 3. WebGL 2 / WebGPU 后端

`Texture3D` 依赖 WebGL 2 或 WebGPU。如果用户开的是 WebGL 1，会自动降级或失败，需要提示用户切到 WebGL 2。

### 4. 不要把 `flipY` 当通用公开 API

demo 中直接 `shell.flipY = true` 后调 `_initTextures` 重新加载纹理，是私有路径。普通业务上请避免改这个字段。

### 5. 大体积数据要算好 `maxTextureBytes`

一张 1024×1024 的 RGBA 浮点图约 16MB，50 层就是 800MB。引擎会按 `maxTextureBytes` 自动降级切片数或模式，但显式传小一点更稳。

## Failure Cases

-   加了 scene component 但完全看不到 → 检查 `minHeight` / `maxHeight` 与实际切片数值范围；或相机 `pitch` 太低，视线没穿过 shell
-   切时间 / 类型图没变 → 检查 `urlBuilder` 是否正确拼路径
-   显存爆掉 → 调小 `maxTextureBytes` 或减少 `layerUrls` 长度
-   数值模式全黑 / 全亮 → `dataMin` / `dataMax` 与实际数据范围不匹配

## See Also

-   `minemap-scene-components`
-   `minemap-material-system-and-shading`
-   `minemap-performance-and-backend`
