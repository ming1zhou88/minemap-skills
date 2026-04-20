---
name: minemap-scene-components
description: MineMap 三维场景组件体系：SceneModel、SceneTileset、SceneObject（Earth/Skybox/AirLine 等），重点补全 glTF/glb 与 3D Tiles 参数边界。
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

-   场景对象：`Sun`、`Earth`、`Skybox`、`Panorama`
-   业务对象：`AirLine`、`BeiDouGrid`、`GradientLine`、`GaussianSplatting`、`AdminDivision`
-   模型辅助：`TerrainLeveling`、`ModelInstance`、`ModelInstanceCollection`
-   裁剪辅助：`ClippingPlane`、`ClippingPlaneCollection`

## SceneModel（glTF / glb）详细参数

这一节以 `source/style/SceneModel.js` 为准。

### 使用定位

推荐场景：

-   单个 glTF / glb 模型
-   一份 glTF 做大量实例化
-   需要 pick、裁剪、阴影、水面交互、线框、模型自带动画

### 最核心的输入字段

-   `id`
-   `data`：glTF JSON、glb URL，或可被内部 loader 解析的数据
-   `map`

### 变换参数优先级

源码和注释都很明确，优先级必须按下面理解：

1. `matrix` 存在时，`position` / `rotation` / `scale` / `quaternion` 都被忽略
2. 没有 `matrix` 时，用 TRS 组合
3. `rotation` 和 `quaternion` 同时给时，优先 `rotation`

因此推荐：

-   简单业务用 `position + rotation + scale`
-   精确姿态控制用 `matrix`

### 位置与姿态

-   `position`：模型锚点位置，顺序 XYZ
-   `scale`：XYZ 缩放
-   `rotation`：欧拉角旋转，顺序 XYZ
-   `quaternion`：四元数旋转
-   `matrix`：完整变换矩阵
-   `modelFolder`：模型外部贴图、资源文件夹地址

### 渲染与显示控制

-   `minimumPixelSize`：最小屏幕像素尺寸，默认 `0`
-   `visibility`：默认 `visible`
-   `allowPick`：默认 `true`
-   `minzoom`：默认 `0`
-   `maxzoom`
-   `opacity`
-   `backCulling`

### 光照、分类、阴影、水面

-   `lightingModel`
-   `classificationType`
-   `castShadow`
-   `receiveShadow`
-   `allowWaterRefract`
-   `allowWaterReflect`

源码实现上的关键点：

-   构造默认 `lightingModel` 实际取 `LightingModelType.PBR`
-   `classificationType` 默认 `ClassificationType.NONE`
-   `castShadow` 默认 `true`
-   `receiveShadow` 默认 `true`

`classificationType` 的重要边界：

-   注释只允许 `IGNORECLASSIFICATION` 和 `NONE`
-   `IGNORECLASSIFICATION` 用在地形下挖、标绘场景较多
-   开了 `IGNORECLASSIFICATION` 后，不再接受 layer / primitive 的矢量贴地叠加

### glTF 自带动画与包围盒

-   `activeAnimation`：是否激活 glTF 自带动画，默认 `true`
-   `accurateBounding`：是否按动画实时刷新精确包围体，默认 `false`
-   `loaded`：加载完成回调

使用建议：

-   模型如果会动，但你只是看，不一定开 `accurateBounding`
-   只有出现视锥误剔除、动画边界错误时，再考虑打开

### 裁剪与距离控制

-   `clippingPlanes`
-   `distanceDisplayCondition`

### 实例化模式

源码里非常关键的一点：当传入 `positions` 时，会进入实例化模型路径。

实例化相关字段：

-   `positions`
-   `rotations`
-   `scales`
-   `shows`
-   `colors`
-   `instanceNames`

推荐场景：

-   一棵树、一个路灯、一个车模型的大量重复布点

不推荐：

-   把 1000 个同款模型写成 1000 个独立 `addSceneComponent({type:'3d-model'})`

### 线框相关参数

-   `showEmbeddedWireframe`：显示模型内置线框，默认 `true`
-   `generateWireframe`：是否生成折痕线框，默认 `false`
-   `showGeneratedWireframe`：默认跟随 `generateWireframe`
-   `wireframeCreaseThreshold`：默认 `0.05235988` 弧度

这套参数的意义：

-   有些 glTF 节点名本身就带 `_wireframe` / `_crease_edge`
-   没有内置线框时，可以让引擎在装配阶段按折痕阈值生成

最近 master 需要补充的一条行为是：

-   `_crease_edge` 已被明确纳入内置线框尾缀识别
-   不只是 `_wireframe`，带 `_crease_edge` 的内置数据也会进入这套兼容逻辑

对应 demo：

-   `demo/html/ModelCreaseEdgeWireframeRender.html`

因此当前版本更准确的表述是：

-   内置线框命名兼容至少覆盖 `_wireframe` / `_crease_edge`
-   如果资产本身已经带好这类内置线框数据，应优先用 `showEmbeddedWireframe`
-   只有没有内置线框数据时，再考虑 `generateWireframe`

### 自定义属性

-   `properties`

这不是渲染参数，而是给业务层挂附加信息的容器，常用于点击后弹面板、三维查询结果识别。

### 推荐写法

```javascript
map.addSceneComponent({
	id: "building-gltf",
	type: "3d-model",
	data: "https://example.com/building.glb",
	modelFolder: "https://example.com/assets/",
	position: [116.39, 39.9, 0],
	rotation: [90, 0, 0],
	scale: [1, 1, 1],
	allowPick: true,
	castShadow: true,
	receiveShadow: true
});
```

### 实例化推荐写法

```javascript
map.addSceneComponent({
	id: "tree-instance",
	type: "3d-model",
	data: "https://example.com/tree.glb",
	modelFolder: "https://example.com/assets/",
	positions,
	rotations,
	scales,
	shows,
	colors,
	instanceNames
});
```

## SceneTileset（3D Tiles）详细参数

这一节以 `source/style/SceneTileset.js` 为准。

### 使用定位

推荐场景：

-   倾斜摄影
-   BIM 切片
-   点云 `pnts`
-   多份 tileset 组合管理

### 数据入口

-   `id`
-   `map`
-   `url`
-   `urls`

推荐：

-   单份也尽量用 `urls` 思维理解，因为 MineMap 明确支持多份 tileset 组合

### 显示与变换

-   `show`：默认 `true`
-   `translation`：沿模型局部坐标平移，单位米
-   `minzoom`
-   `maxzoom`
-   `opacity`
-   `backCulling`

### 清晰度与缓存预算

-   `maximumScreenSpaceError`
-   `maximumMemoryUsage`

源码真实默认值：

-   `maximumScreenSpaceError = 16`
-   `maximumMemoryUsage = 256`

说明：

-   注释里还有旧文案写 64 MB，但运行时代码已经是 256 MB，文档应以实现为准
-   `maximumScreenSpaceError` 越小越清晰，网络和内存开销越大

### 跨级调度参数

-   `loadSiblings`
-   `skipLevelOfDetail`
-   `baseScreenSpaceError`
-   `skipLevels`
-   `skipScreenSpaceErrorFactor`
-   `immediatelyLoadDesiredLevelOfDetail`
-   `schedulingPolicy`

源码默认值：

-   `loadSiblings = false`
-   `skipLevelOfDetail = false`
-   `baseScreenSpaceError = 1024`
-   `skipLevels = 1`
-   `skipScreenSpaceErrorFactor = 16`
-   `immediatelyLoadDesiredLevelOfDetail = false`
-   `schedulingPolicy = BALANCED`

使用建议：

-   默认先不用跨级调度
-   只有大数据场景卡在调度阶段时，再逐项尝试
-   `immediatelyLoadDesiredLevelOfDetail` 会忽略 skip 相关因素，只拉目标精度 tile，移动相机时常更慢，不是普遍优化开关

### 点云专属字段

-   `pointSize`
-   `customizedColor`

说明：

-   `pointSize` 只对点云生效，默认 `1`
-   `customizedColor` 目前只对点云生效
-   点云不支持部分普通 mesh 参数，比如 pick / clipping / opacity 这类能力要具体看数据类型

### 拾取、分类、光照

-   `allowPick`：默认 `true`
-   `classificationType`
-   `lightingModel`：默认 `LightingModelType.NONE`
-   `sourceLoaded`

`sourceLoaded` 和 `SceneModel.loaded` 不同：

-   `sourceLoaded` 更偏 source / tileset 加载完成后的回调
-   很适合在 tileset 边界可用后立即 `flyTo({ target: tileset })`

### `geoBoundsOptions`

这是 MineMap 特有、而且很实用的参数：

-   默认 `{ index: 0, combine: false }`

作用：

-   用 `urls` 同时挂了多份 3D Tiles 时，指定飞行对准哪一份
-   或者把多份边界合并后再飞

规则：

-   一旦指定 `index`，`combine` 就不会再参与合并逻辑

### 裁剪、阴影、水面、线框

-   `clippingPlanes`
-   `castShadow`
-   `receiveShadow`
-   `allowWaterRefract`
-   `allowWaterReflect`
-   `showEmbeddedWireframe`
-   `generateWireframe`
-   `showGeneratedWireframe`
-   `wireframeCreaseThreshold`

源码默认值：

-   `castShadow = true`
-   `receiveShadow = false`
-   `allowWaterRefract = false`
-   `allowWaterReflect = true`
-   `showEmbeddedWireframe = true`
-   `generateWireframe = false`

### 其他运行时参数

-   `subIPs`：多服务地址并发方案，不建议常规使用
-   `debugFreezeFrame`：调试用，慎用
-   `flowOptions`：流动纹理参数
-   `useCompTexMipmapsGenerate`：压缩纹理解压后生成 mipmaps，容易吃内存，尤其倾斜摄影慎开

### 推荐写法

```javascript
map.addSceneComponent({
	id: "city-tiles",
	type: "3d-tiles",
	urls: [{ url: "https://example.com/tileset.json", name: "city" }],
	maximumScreenSpaceError: 8,
	maximumMemoryUsage: 512,
	allowPick: true,
	sourceLoaded: (tileset) => {
		map.flyTo({ target: tileset, duration: 1500, range: 0 });
	}
});
```

### 多份 tileset 推荐写法

```javascript
map.addSceneComponent({
	id: "multi-tiles",
	type: "3d-tiles",
	urls: [
		{ url: "https://example.com/a/tileset.json", name: "A" },
		{ url: "https://example.com/b/tileset.json", name: "B" }
	],
	geoBoundsOptions: { index: 0, combine: false }
});
```

## 获取/移除组件

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
