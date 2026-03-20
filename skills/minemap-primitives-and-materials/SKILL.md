---
name: minemap-primitives-and-materials
description: MineMap Primitive 体系、几何与材质组合、PointPrimitiveCollection 与渲染顺序控制。
---

# MineMap Primitives and Materials

## Quick Start

```javascript
const matrix = minemap.Transforms.cartographicToFixedFrame(new minemap.Math.Vector3(116.39, 39.9, 0));

const geometry = new minemap.Geometries.ConeGeometry({
	radiusBottom: 60,
	height: 100,
	radialSegments: 36
});

const primitive = new minemap.Primitive({
	id: "cone-1",
	modelMatrix: matrix,
	geometry,
	material: minemap.StandardMaterial.fromType("Color", { color: "green" })
});

map.addPrimitive(primitive);
```

## Core Concepts

### 1) 组织方式

`Primitive = geometry + material (+ modelMatrix)`，适合自定义渲染对象。

常见几何：

-   `BoxGeometry` / `SphereGeometry` / `ConeGeometry`
-   `PolylineGeometry` / `WallGeometry`
-   `RegionGeometry` / `FenceGeometry`

常见材质：

-   `StandardMaterial`
-   `PhongMaterial`
-   `LambertMaterial`
-   `PhysicalMaterial`
-   `WaterMaterial`

### 2) 点图元集合

使用 `PointPrimitiveCollection` 管理大量点对象，优于单独创建大量离散 primitive。

### 3) 生命周期 API

-   `addPrimitive(primitive)`
-   `removePrimitive(primitive)`
-   `removePrimitiveById(id)`
-   `getPrimitiveById(id)`

### 4) 低层资源与扩展图形族

`source/index.js` 公开了比“几何 + 材质”更低一层的资源对象：

-   纹理：`TextureLoader`、`TextureCubeLoader`、`Texture`、`TextureCube`
-   材质补充：`BasicMaterial`、`MirrorMaterial`、`BillboardMaterial`、`GradientColorBoxMaterial`
-   几何补充：`TubeGeometry`、`LocalPolylineGeometry`、`FrustumGeometry`、`FrustumOutlineGeometry`、`RadarHemisphereGeometry`
-   光照补充：`AmbientLight`、`SunLight`、`DirectionalLight`、`PointLight`、`SpotLight`

这些对象仍然属于同一低层渲染面：只有在图层和 scene component 都不合适时再使用。

## Common Patterns

### 批量点图元

```javascript
const p = new minemap.PointPrimitive({
	position: minemap.Math.Vector3.fromDegrees(116.39, 39.9, 20),
	pixelSize: 10,
	color: "red"
});

const pc = new minemap.PointPrimitiveCollection({
	id: "points",
	pointPrimitives: [p],
	allowPick: true
});

map.addPrimitive(pc);
```

### 渲染顺序

透明对象冲突时，可调整图元顺序（按引擎提供的 primitive 顺序控制接口），避免视觉覆盖异常。

### 纹理装载

```javascript
const texture = minemap.TextureLoader.load({
	map,
	texUrl: "../image/uv.jpg"
});

const material = new minemap.StandardMaterial({
	baseColorMap: texture
});
```

经验规则：

-   纹理装载尽量复用，不要在高频交互回调里重复 `load`
-   环境贴图、天空盒一类再考虑 `TextureCubeLoader`
-   贴图未就绪前，不要假设材质已经可以稳定参与渲染

### 灯光对象的定位

公开光照类是有效开发面，但它们通常是 primitive / 自定义对象渲染链的补充，不是普通底图业务的第一入口。

## Performance Tips

-   大量静态对象优先合批（集合/实例化）
-   控制半透明对象数量，减少过度混合
-   几何精度与分段数按距离动态分级
-   纹理、材质、几何对象优先复用，减少重复创建 GPU 资源

## See Also

-   `minemap-2d-3d-overlay-and-classification`
-   `minemap-transforms`
-   `minemap-math-foundations`
-   `minemap-lighting-and-shadows`
-   `minemap-material-system-and-shading`
-   `minemap-primitive-adapt-terrain`
-   `minemap-screen-space-reflection`
-   `minemap-water-surface-and-refraction`
-   `minemap-scene-components`
-   `minemap-events-and-picking`
