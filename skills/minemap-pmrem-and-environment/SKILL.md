---
name: minemap-pmrem-and-environment
description: MineMap environment 环境贴图、HDR/cubemap 输入与内部 PMREM 边界。用于全局 PBR 环境配置和资源管理，不推荐内部 PMREM 构造器。
metadata:
  minemap-version: 4.28.2
---

# MineMap PMREM and Environment

## 概述

MineMap 的"环境贴图"分两层：

1. **`map.environment`**：业务层最常用接口，直接喂一张 `Texture` 或 `TextureCube` 给全局 PBR 材质
2. **`PMREMGenerator`**：把原始环境图做预过滤（面向不同粗糙度的卷积预过滤），得到更适合 IBL 用的输入

业务入口是 map.environment，内部 PMREM 不作为公共 API 推荐。

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

## 内部 PMREM 边界

PMREMGenerator 位于 source/third-party/threejs/pmrem/PMREMGenerator.js，未公开导出；不能构造 minemap.PMREMGenerator，也不能依赖 painter._pmremGenerator。设置 map.environment 后，引擎内部调用 PMREM，业务无需手动 fromScene/fromCubemap 或 dispose 引擎实例。PMREM 是卷积预过滤辐射环境贴图，不是“切比雪夫近似”。自行创建的输入纹理在确认无人引用后释放；引擎内部帧缓冲由引擎管理。

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

### 模式 3：纹理加载后交由引擎预过滤

使用 TextureLoader 的 loaded 回调设置 map.environment，不直接访问 painter 的内部实例或回调字段。

### 模式 4：替换自定义环境图与恢复自动环境

```javascript
// 开
map.environment = envTexture;
// 回到引擎自动环境路径（不是关闭 IBL）
map.environment = null;
```

## Strict Constraints

### 1. `map.environment` 不能当天空盒

源码注释明确：**目前阶段此图不能作为天空盒使用**。要天空盒走 `Sky` 入口（见 `minemap-sky-atmosphere`）。

### 2. HDR 纹理必须用 `HALF_FLOAT`

```javascript
const hdrTextureOptions = {
  pixelDatatype: minemap.WebGLConstants.HALF_FLOAT,
  resourceFormat: "HDR"
};
```

否则 LDR 数据被钳制到 0..1，金属反射会丢高光。

### 3. 等距柱状投影推荐 1k（1024 x 512）

`PMREMGenerator.fromEquirectangular` 注释：理想尺寸 1k，与 256x256 立方体输出最匹配。更大输入增加成本，最终质量还受输出分辨率与材质采样限制，应按资产实测。

### 4. 立方体贴图推荐 256x256 / 面

`fromCubemap` 注释：理想大小 256x256，与 256x256 立方体输出最匹配。

### 5. PMREMGenerator 是内部实例工具

`source/third-party/threejs/pmrem/PMREMGenerator.js` 有共享资源生命周期提醒；这不表示公开静态类：

> "你通常不需要多个 PMREMGenerator 对象。如果你确实创建了多个对象，对其中一个调用 dispose() 方法会导致其他对象也无法使用。"

### 6. 不设定 `map.environment` 不会黑屏

引擎会自动抓帧生成环境光，业务上"什么都不做"也跑得通；只有追求特定 IBL 效果才需要喂纹理。

## Failure Cases

-   金属反射黑 → 检查输入加载、HDR 格式、材质参数和自动环境内容，未设置自定义 environment 并不必然黑屏
-   HDR 加载后材质不变化 → `pixelDatatype` / `resourceFormat` 不是 HDR
-   环境纹理占用持续上涨 → 检查应用拥有的输入纹理生命周期，不手工销毁引擎 PMREM 实例
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
