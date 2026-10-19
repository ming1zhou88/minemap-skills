---
name: minemap-water-bodies
description: MineMap 水体类选型与使用规范。覆盖 DynamicWater、Ocean、WaterPro、RiverFlowRenderer、FluidRenderer、FloodRenderer 与 ThreeLayer GPGPU 水文分析；按场景选择合适方案并正确接入、调参。当用户需要渲染水面、海面、河流、洪水或做水文分析时调用。
---

# MineMap Water Bodies

MineMap 提供了多套「水体 / 水面」渲染与仿真能力。它们不是同一能力的别名，而是定位、技术栈、性能开销、接入方式都不同的独立实现。本 skill 负责「选型 + 正确接入 + 调参与排查」。

> 注意：本 skill 覆盖的是**场景组件 / 独立渲染器级**的水体类。`WaterMaterial` + `Primitive`、`fill` 图层 + `fill-water`、`waterRenderingMode`、`allowWaterRefract/Reflect` 等内容见 `minemap-water-surface-and-refraction`。

## 速览与选型

| 类 / 入口                   | 示例                         | 定位                           | 接入方式                                   |
| --------------------------- | ---------------------------- | ------------------------------ | ------------------------------------------ |
| `minemap.DynamicWater`      | `DynamicWater.html`          | 小封闭水体（泳池/水塘/人工湖） | `map.addSceneComponent`                    |
| `minemap.Ocean`             | `Ocean.html`                 | 大范围真实海面（FFT）          | `map.addSceneComponent`                    |
| `minemap.WaterPro`          | `WaterPro.html`              | 电影级海洋 + 船体互动          | `map.addSceneComponent`                    |
| `minemap.RiverFlowRenderer` | `RiverFlowRenderer.html`     | 河道顺流演进                   | `new` + `await add()`                      |
| `minemap.FluidRenderer`     | `FluidRender.html`           | 洪水扩散（地图 DEM）           | `new FluidRenderer(map, options)`          |
| `minemap.FloodRenderer`     | `Flood.html`                 | 浅水方程洪水（3D 模型）        | `new FloodRenderer(map, { terrainModel })` |
| `ThreeLayer` + GPGPU        | `Threejs-HydroAnalysis.html` | 交互式水文分析                 | `map.addLayer(new ThreeLayer(...))`        |

**按场景一句话选型**：

-   泳池 / 水塘 / 人工湖，要看水底和边界 → `DynamicWater`
-   入海口 / 海湾 / 海域，追求物理真实 → `Ocean`
-   舰船 / 码头 / 海港，要浮船 / 尾迹 / 水下 → `WaterPro`
-   河流 / 河道，顺流演进或流向可视化 → `RiverFlowRenderer`
-   洪水淹没，只有地图 DEM → `FluidRenderer`
-   洪水淹没，已有 3D 地形模型、要统计指标 → `FloodRenderer`
-   降雨 / 侵蚀 / 蒸发 / 雕刻等交互式水文研究 → `ThreeLayer` + GPGPU
-   大量矢量面水域（湖泊 / 水库），保留图层体系 → `fill` + `fill-water`（见 `minemap-water-surface-and-refraction`）

## 决策流程

1. **是「仿真」还是「渲染」？**
    - 洪水/淹没仿真：已有 3D 模型 → `FloodRenderer`；只有地图 DEM → `FluidRenderer`
    - 河道顺流演进 → `RiverFlowRenderer`
    - 交互式水文研究（雨/侵蚀/蒸发/雕刻）→ `ThreeLayer` + GPGPU
    - 只是「把水面画出来」→ 下一步
2. **水域已在图层体系里？**
    - 是（矢量面，效果要求不高）→ `fill` + `fill-water`
    - 要场景组件级精细效果 → 下一步
3. **水域大小与真实感？**
    - 小封闭水体 → `DynamicWater`
    - 大范围海面、物理真实 → `Ocean`
    - 中尺度、电影化 + 船体互动 → `WaterPro`

## 各方案用法

### `minemap.DynamicWater` —— 小封闭水体

一次性生成水面 + 水底焦散 + 侧墙。掩膜纹理（`maskSize`）裁剪出多边形水域，分形噪声（fBm）驱动波浪。

```javascript
const water = new minemap.DynamicWater({
	positions: [[120.301135, 28.575922] /* ... 多边形，无需手动闭合 */],
	maskSize: 512,
	widthSegments: 256,
	heightSegments: 256,
	waterHeight: 10,
	bottomHeight: 0,
	withGround: true,
	withWalls: true, // 水底焦散 / 侧墙
	groundTexture: "../image/pool_tile.jpg",
	wallColor: "#7ec8e3",
	wallOpacity: 0.28,
	water: { color: "#4a90a4", opacity: 0.9, waveAmplitude: 0.39, flowDirection: [1, 0] /* ... */ },
	caustics: { intensity: 1, scale: 10, speed: 0.5 }
});
map.addSceneComponent(water);
water.setOptions({ water: { opacity: 0.7 } });
map.removeSceneComponent(water.id);
```

约束：不依赖 WebGL2；波浪是程序化噪声，大范围会失真；边缘精度受 `maskSize` 限制。

### `minemap.Ocean` —— 大范围真实海面

Tessendorf FFT 频谱 + Gerstner 涌浪 + GGX 着色 + 程序化泡沫。依赖 WebGL2 + `EXT_color_buffer_float`。

```javascript
const ocean = new minemap.Ocean({
  positions: [...],               // 单多边形外环，不支持孔洞/MultiPolygon
  ocean: { spectrum: 'JONSWAP', windSpeed: 9, windDirection: 43, fetch: 150000,
           fftSize: 128, meshResolution: 256, patchSize: 1200 },
  water: { roughness: 0.09, specular: 0.8, ior: 1.333 },
  foam: { threshold: 0.62 },
  lighting: { month: 6, day: 6, hours: 10 }
});
map.addSceneComponent(ocean);
ocean.updateOptions({ ocean: { windSpeed: 12 } });
ocean.pause(); ocean.resume(); ocean.setSunLightTime({ hours: 14 });
```

约束：`positions` / `fftSize` / `meshResolution` 变化会重建资源，不要在高频回调里连续调用；单外环，无孔洞。

### `minemap.WaterPro` —— 电影级海洋 + 船体互动

Gerstner 波叠加 + 程序化泡沫（白帽/岸线/接触/尾迹）+ 程序化天空 + 轻量后处理。支持多孔洞 `excludePositions`、浮船采样 `sampleSurface()`、水下视角。

```javascript
const waterPro = new minemap.WaterPro({
  positions: [...], excludePositions: [[...], [...]], // 支持多孔洞/排除区
  waves: { windDirection: 28.8, amplitude: 0.72 },
  foam: { coverage: 0.9, shoreWidth: 3.4 },
  underwater: { fogStrength: 1, causticStrength: 1 },
  post: { exposure: 1.05, bloom: 0.5 }
});
map.addSceneComponent(waterPro);
const surface = waterPro.sampleSurface(boatPosition, boatHeading); // 返回 height/pitch/roll
waterPro.setContactFoamSources([{ position, heading, length: 30, width: 9, strength: 1.35 }]);
```

约束：波浪偏风格化（非 FFT）；多 pass + 多层波采样开销中高；船体互动依赖 CPU 侧 `sampleSurface` 与 shader 同相位。
`positions` 表示单个水域外环；4.27.1 扩展的是 `excludePositions`，可传多个排除多边形 / 孔洞，不代表 `positions` 接受 GeoJSON `MultiPolygon`。多个分离水域应分别创建 `WaterPro` 对象。

### `minemap.RiverFlowRenderer` —— 河道顺流演进

以遮罩图（`maskImageUrl`）+ 流向图（`flowImageUrl`）+ 法线图（`normalImageUrl`）+ 矩形范围（`riverBounds`）驱动，CPU 生成传播纹理。

```javascript
const river = new minemap.RiverFlowRenderer({
	map,
	maskImageUrl: "../image/river-range.png",
	flowImageUrl: "../image/flowmap.png",
	normalImageUrl: "../image/water/waternormals.jpg",
	mode: "water", // arrow | water | range
	riverBounds: [118.568, 29.815, 118.621, 29.84], // [west, south, east, north]
	riverHeight: 145.65,
	water: { revealDuration: 7, animationSpeed: 0.012, opacity: 0.88 }
});
await river.add(); // 异步：加载纹理、拟合水源、生成传播纹理
river.fit({ duration: 0 });
river.setMode("arrow");
river.setWaterOptions({ opacity: 0.6 });
river.restart();
```

约束：依赖「遮罩图 + 流向图」美术资产；矩形范围适合单一河段；波浪是法线扰动，非几何位移。

### `minemap.FluidRenderer` —— 洪水扩散（地图 DEM）

围绕水源点捕捉地图地形高度，多 pass 纹理计算推进水量与出流。依赖 `map.setTerrain`。

```javascript
const fluid = new minemap.FluidRenderer(map, {
  lonLat: [118.5718, 29.8221],
  width: 1024, height: 1024,
  dimensions: new minemap.Math.Vector3(10000, 10000, 1000),
  sourceUv: new minemap.Math.Vector2(0.5, 0.5),
  flowInertia: 0.995, flowForce: 0.25, timeStep: 0.3
});
fluid.pause(); fluid.resume(); fluid.reset({ ... }); fluid.destroy();
```

约束：参数较敏感，尽量初始化时调好；视觉偏仿真示意；重建有成本。

### `minemap.FloodRenderer` —— 浅水方程洪水（3D 模型）

以 `3d-model` 地形模型为底（`terrainModel`，需含 `boundingVolume`），GPU 三步算子分裂（Advect → Height → Velocity）+ 统计 pass。参考 WebFlood / Iowa City 浅水方程。

```javascript
floodRenderer = new minemap.FloodRenderer(map, {
	terrainModel: model, // 3d-model 场景对象（必需）
	resolution: 512, // 创建后不可改
	dt: 0.25,
	manningCoefficient: 0.07,
	sourceWaterHeight: 23,
	sourceRadius: 0.035,
	stepsPerFrame: 2,
	opacity: 0.85
});
floodRenderer.setDisplayMode("water"); // water | duration | velocity | maxDepth
floodRenderer.updateOptions({ manningCoefficient: 0.1 });
floodRenderer.pause();
floodRenderer.resume();
floodRenderer.reset(params);
floodRenderer.destroy();
floodRenderer.getWaterStatistics(); // 湿网格数 / 最大水深
floodRenderer.getDebugInfo(); // 调试信息表
```

约束：地形必须用 `3d-model`（非 `setTerrain`）；`resolution` 创建后不可改；与 `FluidRenderer` 地形来源不同，勿混用。

### `ThreeLayer` + GPGPU —— 交互式水文分析

以 `ThreeLayer` 为桥、`minemap-three-plugin` 提供能力（`TerrainSim` / `WaterSim` / `SculptController` / `RainParticles` / `loadTIFFToDataTexture`），与 MineMap 共享 WebGL2 上下文。覆盖降雨 / 蒸发 / 侵蚀 / 雕刻 / GeoTIFF 高程。

```javascript
const map = new minemap.Map({ container: "map", style, renderBackend: "webgl2" }); // 必须 webgl2
const threeLayer = new ThreeLayer({ id: "hydro", refCenter, useDefaultAmbientLight: true });
map.addLayer(threeLayer);
const renderer = threeLayer.getWebGLRenderer();
const camera = threeLayer.getCamera();
const terrainSim = new TerrainSim(renderer, 512, 512);
const sculptController = new SculptController(renderer.domElement);
```

约束：必须 `renderBackend: 'webgl2'`；依赖 `minemap-three-plugin` 与 `three` / `geotiff` CDN；是「图层 + 插件」组合，非单类 API。通用 ThreeLayer 接入见 `minemap-threejs-integration`。

## Strict Constraints

1. **接入方式三选一，不要混用**：`DynamicWater` / `Ocean` / `WaterPro` 用 `addSceneComponent`；`fill-water` / `ThreeLayer` 用 `addLayer`；`RiverFlowRenderer` / `FluidRenderer` / `FloodRenderer` 是独立渲染器，`new` 后持有引用（`RiverFlowRenderer` 还需 `await add()`）。
2. **持续动画必须 `map.repaint = true`**：上述所有动态水体/仿真都依赖逐帧重绘推进时间。
3. **两个洪水类别混用**：`FluidRenderer` 底是地图 DEM（`setTerrain`），`FloodRenderer` 底是 `3d-model`。有 3D 模型用 `FloodRenderer`，只有 DEM 用 `FluidRenderer`。
4. **重建类参数不要高频调用**：`Ocean.positions/fftSize/meshResolution`、`FluidRenderer` 区域/分辨率、`FloodRenderer.resolution`（创建后不可改）、`ThreeLayer` 模拟分辨率，都不应放逐帧或滑块 `input` 连续触发。
5. **`fill-water` / `WaterMaterial` 不要混成本 skill 的类**：矢量水域 / 低层水面材质见 `minemap-water-surface-and-refraction`。

## Failure Cases

-   用 `DynamicWater` 做大范围海面（会失真、缺物理海浪）
-   在无 WebGL2/float 缓冲设备上跑 `Ocean` 或 `ThreeLayer` GPGPU
-   给 `Ocean` 传多环/孔洞 `positions`（只支持单外环）
-   用 `FloodRenderer` 却只设了 `map.setTerrain` 而没传 `terrainModel`（反之亦然）
-   忘记 `map.repaint = true`，导致水面/仿真静止
-   `RiverFlowRenderer` 未 `await add()` 就调用 `fit()` / `setMode()`
-   高频滑块直接改 `FloodRenderer.resolution` 或 `Ocean.fftSize`

## Performance Discipline

-   小水体优先 `DynamicWater`（最省）；大范围海面用 `Ocean` 时从 `fftSize: 128` + 低网格细分起步
-   弱设备先 `waterRenderingMode = "simplified"` 验证基线（见 `minemap-water-surface-and-refraction`）
-   `ThreeLayer` GPGPU 分辨率（`simWidth/simHeight`）越高计算量越大，按需调低
-   连续动画场景确认已开启 `map.repaint`，避免无意义空转

## Demo References

-   `demo/html/DynamicWater.html`
-   `demo/html/Ocean.html`
-   `demo/html/WaterPro.html`
-   `demo/html/RiverFlowRenderer.html`
-   `demo/html/FluidRender.html`
-   `demo/html/Flood.html`
-   `demo/html/Threejs-HydroAnalysis.html`
-   `demo/html/WaterFillLayer.html`（fill 水面，见 `minemap-water-surface-and-refraction`）

## See Also

-   `minemap-water-surface-and-refraction`（WaterMaterial / fill-water / waterRenderingMode）
-   `minemap-threejs-integration`（ThreeLayer 通用接入）
-   `minemap-terrain-and-analysis`（`setTerrain` / DEM，FluidRenderer 依赖）
-   `minemap-scene-components`（`3d-model`，FloodRenderer 依赖）
-   `minemap-performance-and-backend`（`renderBackend: 'webgl2'` 选择）
