---
name: minemap-pmrem-and-environment
description: MineMap 环境贴图（environment）与 PMREM 预过滤辐射度专题。覆盖 map.environment、TextureLoader / TextureCubeLoader、PMREMGenerator.fromEquirectangular / fromCubemap、fromScene 与 compile*Shader 预编译。
---

# MineMap PMREM and Environment

## 概述

MineMap 的"环境贴图"分两层：

1. **`map.environment`**：业务层最常用接口，直接喂一张 `Texture` 或 `TextureCube` 给全局 PBR 材质
2. **`PMREMGenerator`**：把原始环境图做预过滤（多级 mipmap + 各 mipmap 切比雪夫近似），得到更适合 IBL 用的输入

业务上**绝大多数场景**用 `map.environment = texture` 就够了；只有需要"模糊半径 / 视角响应很精细"时才上 `PMREMGenerator`。

## Quick Start：用一张等距柱状投影（HDR）做环境光

对应 demo：`demo/html/PMREMGeneratorUserEquirectangularEnvImage.html`

```javascript
const texture = minemap.TextureLoader.load({
	map,
	texUrl: "../image/equirectangular/san_giuseppe_bridge_2k.hdr",
	anisotropy: 1,
	minFilter: minemap.WebGLConstants.LINEAR,
	magFilter: minemap.WebGLConstants.LINEAR,
	wrapS: minemap.WebGLConstants.CLAMP_TO_EDGE,
	wrapT: minemap.WebGLConstants.CLAMP_TO_EDGE,
	pixelFormat: minemap.WebGLConstants.RGBA,
	pixelDatatype: minemap.WebGLConstants.HALF_FLOAT,
	resourceFormat: "HDR",
	loaded: () => {
		map.environment = texture;
	}
});
```

## Quick Start：用六面体 cubemap 做环境光

对应 demo：`demo/html/PMREMGeneratorUserEnvCubeTextureImage.html`

```javascript
const texture = minemap.TextureCubeLoader.load({
	map,
	path: "../image/cube/Park2/",
	urls: ["posx.jpg", "negx.jpg", "posy.jpg", "negy.jpg", "posz.jpg", "negz.jpg"],
	anisotropy: 1,
	minFilter: minemap.WebGLConstants.LINEAR,
	magFilter: minemap.WebGLConstants.LINEAR,
	wrapS: minemap.WebGLConstants.CLAMP_TO_EDGE,
	wrapT: minemap.WebGLConstants.CLAMP_TO_EDGE,
	pixelFormat: minemap.WebGLConstants.RGBA,
	pixelDatatype: minemap.WebGLConstants.HALF_FLOAT,
	loaded: () => {
		map.environment = texture;
	}
});
```

## `map.environment` 公开行为

来源：`source/api/map.js` 注释

-   类型：`Texture` / `TextureCube`
-   作用：用于计算模型（Standard / Physical / PBR）的**全局光照**
-   不设定时：引擎会通过地图场景进行全景颜色取图（自动抓帧）
-   **重要**：当前阶段此图**不能作为天空盒使用**

设值与清除：

```javascript
map.environment = texture; // 设置
map.environment = null; // 清除（回退到自动抓帧）
```

## `PMREMGenerator` 类

来源：`source/third-party/threejs/pmrem/PMREMGenerator.js`

```javascript
const pmrem = new minemap.PMREMGenerator(painter); // 通常用引擎内置实例
```

> 源码注释明确说明：PMREMGenerator 是"静态类"，**通常不需要多个实例**。业务上一般拿引擎已经创建好的 `pmrem`，不要自己 new。

### 公开方法

| 方法                                                  | 说明                                              |
| ----------------------------------------------------- | ------------------------------------------------- |
| `fromEquirectangular(equirectangular, renderTarget?)` | 从等距柱状投影纹理生成 PMREM（推荐 1k：1024x512） |
| `fromCubemap(cubemap, renderTarget?)`                 | 从立方体贴图生成 PMREM（推荐 256x256 / 面）       |
| `fromScene(scene, sigma?, near?, far?, options?)`     | 直接把场景烘焙成 PMREM（适合 scene-level IBL）    |
| `compileEquirectangularShader()`                      | 预编译等距柱状投影 shader（用于网络并发预热）     |
| `compileCubemapShader()`                              | 预编译 cubemap shader（用于网络并发预热）         |
| `dispose()`                                           | 释放内部显存                                      |

### `fromScene` 选项

| 选项       | 说明                            |
| ---------- | ------------------------------- |
| `size`     | 输出 PMREM 立方体尺寸，默认 256 |
| `position` | 内部 cube camera 位置，默认原点 |

`sigma` 是预模糊半径（弧度），> 0 会在生成 PMREM 前对 scene 做预模糊。

### 性能建议

-   `fromEquirectangular` / `fromCubemap` 是**同步**操作（GPU 端），业务上放在纹理 `loaded` 回调里
-   `compileEquirectangularShader()` / `compileCubemapShader()` 是**预编译**接口；可以在网络下载纹理**期间**调用，节省首帧卡顿
-   `dispose()` 必须在替换或销毁 PMREM 时调用；漏掉会持续占显存

## Architecture Positioning

-   **`map.environment` = 业务最常用入口**：直接喂 `Texture` / `TextureCube`
-   **`PMREMGenerator` = 内部 IBL 工具**：把 environment 转成多 mipmap 的预过滤版本
-   业务上**不需要手动 PMREM** 也能跑：`map.environment = texture` 后引擎内部会自动 PMREM 化

`PBRMaterial` 的环境光依赖 `map.environment`：

```javascript
const material = new minemap.PhysicalMaterial({
	color: "#888",
	metalness: 1.0,
	roughness: 0.4,
	envMapIntensity: 1.0
});
// material.envMapIntensity 在材质层可调
```

`envMapIntensity` 是 PBR 材质对环境光的强度，参考 demo `PMREMSpherePrimitive.html`。

## Common Patterns

### 模式 1：一张 HDR 全景 → 全局 PBR 环境光

```javascript
const tex = minemap.TextureLoader.load({
	map,
	texUrl: "../image/equirectangular/*.hdr",
	resourceFormat: "HDR",
	pixelDatatype: minemap.WebGLConstants.HALF_FLOAT,
	loaded: () => {
		map.environment = tex;
	}
});
```

### 模式 2：六张面图 → 立方体环境光

```javascript
const tex = minemap.TextureCubeLoader.load({
	map,
	path: "...",
	urls: ["posx.jpg", "negx.jpg", "posy.jpg", "negy.jpg", "posz.jpg", "negz.jpg"],
	loaded: () => {
		map.environment = tex;
	}
});
```

### 模式 3：预编译 shader 减少首帧卡顿

```javascript
// 启动早期
const pmrem = painter._pmremGenerator;
pmrem.compileEquirectangularShader();
pmrem.compileCubemapShader();

// 等纹理下载完再喂
tex.onLoaded = () => {
	map.environment = tex;
};
```

### 模式 4：动态开关环境光（用于性能调优）

```javascript
// 开
map.environment = envTexture;
// 关
map.environment = null;
```

## Strict Constraints

### 1. `map.environment` 不能当天空盒

源码注释明确：**目前阶段此图不能作为天空盒使用**。要天空盒走 `Sky` 入口（见 `minemap-sky-atmosphere`）。

### 2. HDR 纹理必须用 `HALF_FLOAT`

```javascript
pixelDatatype: minemap.WebGLConstants.HALF_FLOAT,
resourceFormat: "HDR"
```

否则 LDR 数据被钳制到 0..1，金属反射会丢高光。

### 3. 等距柱状投影推荐 1k（1024 x 512）

`PMREMGenerator.fromEquirectangular` 注释：理想尺寸 1k，与 256x256 立方体输出最匹配。再大会更慢但不会更清晰。

### 4. 立方体贴图推荐 256x256 / 面

`fromCubemap` 注释：理想大小 256x256，与 256x256 立方体输出最匹配。

### 5. PMREMGenerator 是静态类

`source/third-party/threejs/pmrem/PMREMGenerator.js` 注释：

> "你通常不需要多个 PMREMGenerator 对象。如果你确实创建了多个对象，对其中一个调用 dispose() 方法会导致其他对象也无法使用。"

### 6. 不设定 `map.environment` 不会黑屏

引擎会自动抓帧生成环境光，业务上"什么都不做"也跑得通；只有追求特定 IBL 效果才需要喂纹理。

## Failure Cases

-   金属反射全是黑 → 没设 `map.environment`（或设了但 LDR 钳制）
-   HDR 加载后材质不变化 → `pixelDatatype` / `resourceFormat` 不是 HDR
-   PMREM 内存爆炸 → 没在替换 / 销毁时 `dispose()`
-   多 `PMREMGenerator` 互相破坏 → 没遵守"静态类"约束
-   `map.environment` 设了但天空没变 → 预期行为，environment 不控制天空盒

## Demo References

-   `demo/html/PMREMGeneratorUserEquirectangularEnvImage.html`：HDR 等距柱状投影环境光
-   `demo/html/PMREMGeneratorUserEnvCubeTextureImage.html`：六面体 cubemap 环境光
-   `demo/html/PMREMSpherePrimitive.html`：环境光 + `envMapIntensity` 调参
-   `demo/html/PMREM3DTiles.html`：环境光 + 3D Tiles 组合
-   `demo/html/PMREMSkyAtmosphere.html`：SkyAtmosphere + 环境光组合
-   `demo/html/PMREMGenerator.html`：3D Tiles 浏览器（仅名字撞车，并不直接调用 PMREMGenerator）

## See Also

-   `minemap-sky-atmosphere`：`Sky` / `SkyAtmosphere` 入口
-   `minemap-material-system-and-shading`：`PhysicalMaterial` / PBR 调参
-   `minemap-scene-components`：3D Tiles 加载与材质联动
