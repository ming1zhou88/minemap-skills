---
name: minemap-scene-components
description: MineMap 三维场景组件体系：SceneModel、SceneTileset、SceneObject（Earth/Skybox/AirLine 等），用于 3D 模型与 3D Tiles 场景搭建。
---

# MineMap Scene Components

## Quick Start

```javascript
map.on("load", () => {
	map.addSceneComponent({
		id: "building-gltf",
		type: "3d-model",
		data: "https://example.com/building.glb",
		position: new minemap.Math.Vector3(116.39, 39.9, 0),
		rotation: [90, 0, 0],
		scale: [1, 1, 1],
		allowPick: true
	});

	map.addSceneComponent({
		id: "city-tiles",
		type: "3d-tiles",
		urls: [{ url: "https://example.com/tileset.json", name: "city" }],
		maximumScreenSpaceError: 8,
		maximumMemoryUsage: 512
	});
});
```

## Core Concepts

### 1) 统一入口

`map.addSceneComponent()` 是三维对象推荐入口，内部按 `type` 分发：

-   `3d-model` -> `SceneModel`
-   `3d-tiles` -> `SceneTileset`
-   `scene-object` -> 场景对象（如 `Earth`、`Skybox`、`Panorama`、`AirLine`）

### 2) 迁移规则

-   `addSource(type="3d-tiles")` 与 `addLayer(type="3d-tiles")` 已标注废弃提示
-   新代码不要继续新增该路径

### 2.1) 统一生命周期管理

推荐统一用：

-   `map.addSceneComponent()`
-   `map.getSceneComponent()`
-   `map.removeSceneComponent()`

不要一部分对象走 scene component，一部分对象继续走历史 source/layer 3D Tiles 路径。

### 3) SceneModel 关键项

-   位置/旋转/缩放：`position`、`rotation`、`scale`
-   或直接矩阵：`matrix`（存在时覆盖 TRS）
-   可选拾取：`allowPick`
-   可选最小屏幕像素：`minimumPixelSize`
-   支持实例化：`positions`、`rotations`、`scales`、`colors`、`shows`

### 4) SceneTileset 关键项

-   清晰度：`maximumScreenSpaceError`（越小越清晰，开销越大）
-   缓存：`maximumMemoryUsage`（MB）
-   拾取：`allowPick`
-   视距控制：`minzoom/maxzoom`
-   LOD 策略：`skipLevelOfDetail` 等参数（谨慎启用）

### 5) 其他公开场景对象与辅助类

`source/index.js` 对外不只导出 `SceneModel` / `SceneTileset`，还导出一组 scene-object / scene-helper：

-   场景对象：`Sun`、`Earth`、`Skybox`、`Panorama`
-   业务对象：`AirLine`、`BeiDouGrid`、`GradientLine`、`GaussianSplatting`、`AdminDivision`
-   模型辅助：`TerrainLeveling`、`ModelInstance`、`ModelInstanceCollection`
-   裁剪辅助：`ClippingPlane`、`ClippingPlaneCollection`

它们的推荐挂载方式仍然是：

-   能走 `addSceneComponent()` 的优先走 scene component
-   需要直接构造对象时，也要统一纳入 map 生命周期管理

## Common Patterns

### 获取/移除组件

```javascript
const tileset = map.getSceneComponent("city-tiles");
if (tileset) {
	map.flyTo({ target: tileset, duration: 1500 });
}

map.removeSceneComponent("city-tiles");
```

### 通过矩阵精确控制模型

复杂姿态建议先算 `modelMatrix`（`Transforms.cartographicToFixedFrame` + 自定义旋转缩放），再作为 `matrix` 传入。

### 模型/瓦片裁剪面

```javascript
const clippingPlanes = new minemap.ClippingPlaneCollection({
	enabled: true,
	planes: [
		new minemap.ClippingPlane({
			normal: new minemap.Math.Vector3(1, 0, 0),
			distance: 0
		})
	]
});

sceneModel.clippingPlanes = clippingPlanes;
```

适用对象：

-   `SceneModel`
-   `SceneTileset`
-   部分地形/拉伸对象

注意：demo 中有直接改内部 `_planes` 的调试写法，技能库不把它当正式推荐 API。

## Do Not Do This

-   不要在新业务里继续扩散 `addSource/addLayer` 的旧 3D Tiles 路径
-   不要把大量重复对象拆成大量独立 `3d-model`，优先实例化能力
-   不要默认对所有 scene component 开 `allowPick`

## Demo-backed References

常见业务 demo 已经普遍采用 scene component 作为主入口，尤其是：

-   3D Tiles 装载
-   BIM / glTF 模型
-   航线、北斗网格、高斯泼溅等场景对象

## Demo References

-   `demo/html/3DTiles.html`
-   `demo/html/3DTilesLoaded.html`
-   `demo/html/3DGS-3DTiles.html`
-   `demo/html/BeiDouGridSceneObject.html`
-   `demo/html/AirLine.html`

## Performance Tips

-   3D Tiles 优先设置合理 `maximumScreenSpaceError`（常见 8~16）
-   多份 tileset 优先用 `urls` 聚合管理
-   非必要关闭 `allowPick`
-   对低端机限制并发三维组件数量

## See Also

-   `minemap-2d-3d-overlay-and-classification`
-   `minemap-transforms`
-   `minemap-math-foundations`
-   `minemap-lighting-and-shadows`
-   `minemap-water-surface-and-refraction`
-   `minemap-primitives-and-materials`
-   `minemap-terrain-and-analysis`
-   `minemap-performance-and-backend`
