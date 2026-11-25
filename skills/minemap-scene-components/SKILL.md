---
name: minemap-scene-components
description: MineMap 三维场景组件体系：SceneModel、SceneTileset、SceneObject（Earth/Skybox/AirLine 等），重点补全 glTF/glb 与 3D Tiles 参数边界。
metadata:
  minemap-version: 4.28.2
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
-   `scene-object` -> 场景对象（如 `Earth`、`Panorama`、`AirLine`）；旧 `Skybox` 对象的添加入口已弃用

### 2) 迁移规则

-   `addSource(type="3d-model" | "3d-tiles")` + `addLayer(...)` 是历史路径
-   4.22 继续兼容，但源码注释已经明确不推荐继续新增

最近 master 补充的一个重要现实情况是：

-   历史 `addSource + addLayer(type='3d-model')` 路径在当前版本仍被修复维护
-   `demo/html/GLBAddSourceOfficialStyle.html` 就是专门补给这条历史路径的 shipped demo

因此文档应写成：

-   新业务仍优先 `addSceneComponent()`
-   但如果你在维护旧项目，`addSource + addLayer(type='3d-model')` 不是“理论兼容而已”，当前 master 仍在修复其实际加载问题

### 3) 统一生命周期管理

推荐统一用：

-   `map.addSceneComponent()`
-   `map.getSceneComponent()`
-   `map.removeSceneComponent()`

### 天空盒入口迁移

不要再通过 `map.addSceneComponent(new minemap.Skybox())` 添加天空盒。4.27.1 会发出弃用警告并跳过该对象；改用 `map.setSkyBoxTextures()`，移除时使用 `map.removeSkyBoxTextures()`：

```javascript
map.setSkyBoxTextures({
	starry: { nx, px, ny, py, nz, pz },
	day: { nx, px, ny, py, nz, pz }
});

map.removeSkyBoxTextures({ starry: true, day: true });
```

每组贴图都需要按六个立方体方向提供面资源。天空盒不是 `addSceneComponent()` 的推荐新增路径；场景对象列表中出现 `Skybox` 仅用于识别旧代码。

### 4) SceneModel 和 SceneTileset 的角色边界

`SceneModel`：

-   适合 glTF / glb
-   适合单模型和实例化模型
-   适合做模型级动画、线框、局部裁剪、最小像素显示

`SceneTileset`：

-   适合 3D Tiles
-   适合倾斜摄影、BIM 切片、点云
-   适合做 LOD、内存预算、多数据集组合飞行、TileFeature 级运行时样式

### 5) 其他公开场景对象与辅助类

`source/index.js` 还导出了：

-   场景对象：`Sun`、`Earth`、`Panorama`；`Skybox` 通过 Map 天空盒方法管理
-   业务对象：`AirLine`、`BeiDouGrid`、`GradientLine`、`GaussianSplatting`、`AdminDivision`
-   模型辅助：`TerrainLeveling`、`ModelInstance`、`ModelInstanceCollection`
-   裁剪辅助：`ClippingPlane`、`ClippingPlaneCollection`

## 按需参考

配置 glTF 实例或 3D Tiles 调度时按需阅读模型参数参考。

[参数与模式](references/model-tileset-parameters.md)

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

复杂姿态建议先算 `modelMatrix`（`Transforms.cartographicToFixedFrame` + 自定义旋转缩放），再作为 `matrix` 传入；这时就不要再同时依赖 `position` / `rotation` / `scale`。

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
-   不要把大量重复对象拆成大量独立 `3d-model`，优先实例化
-   不要把 `extrusion` 当 glTF 替代品
-   不要默认对所有 scene component 都开 `allowPick`
-   不要把 `adaptTerrain` 和 “贴 3D Tiles / 贴模型”混为一谈
-   不要把 `ThreeLayer` 融合层和原生 `SceneModel` / `SceneTileset` 当成同一套对象体系

## Demo-backed References

-   `demo/html/GLBAddSourceOfficialStyle.html`
-   `demo/html/ModelCreaseEdgeWireframeRender.html`

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
-   `maximumMemoryUsage` 不要只看注释旧值，要按数据体量实测
-   多份 tileset 优先用 `urls` 聚合管理
-   非必要关闭 `allowPick`
-   实例化模型优先代替海量重复 glTF
-   线框、实时精确包围盒、压缩纹理 mipmaps 都是高成本开关

## See Also

-   `minemap-threejs-integration`

-   `minemap-2d-3d-overlay-and-classification`
-   `minemap-transforms`
-   `minemap-math-foundations`
-   `minemap-lighting-and-shadows`
-   `minemap-water-surface-and-refraction`
-   `minemap-primitives-and-materials`
-   `minemap-terrain-and-analysis`
-   `minemap-performance-and-backend`
