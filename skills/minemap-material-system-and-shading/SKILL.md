---
name: minemap-material-system-and-shading
description: MineMap 材质与着色规范。覆盖 `StandardMaterial`、`PhongMaterial`、`PhysicalMaterial`、`LambertMaterial`、`PolylineMaterial` 等主流材质的选型与调参。
---

# MineMap Material System and Shading

## Quick Start

```javascript
const material = minemap.StandardMaterial.fromType("Color", {
	color: "green",
	opacity: 1,
	doubleSided: false
});

const primitive = new minemap.Primitive({
	geometry,
	material
});

map.addPrimitive(primitive);
```

## Core Concepts

### 1) 材质先按用途分，不要先按类名分

当前最常用的几类是：

-   `StandardMaterial`
    -   通用入口
    -   颜色、贴图、扫描、扩散环等都可走它
-   `PhongMaterial`
    -   经典高光模型
    -   参数直观，适合普通演示材质
-   `PhysicalMaterial`
    -   PBR 路线
    -   金属度、粗糙度、环境贴图更完整
-   `LambertMaterial`
    -   更简单的漫反射思路
-   `PolylineMaterial`
    -   线材专用
-   `WaterMaterial`
    -   水面专用，已单独拆 skill

### 2) `StandardMaterial` 是最常用起点

源码和 demo 都说明，`StandardMaterial` 不只是“普通材质”，而是一个多模式入口。

常见类型包括：

-   `Color`
-   `Texture`
-   `RadarScan`
-   `DiffusingRing`

因此当你还不确定选哪种材质时，先看 `StandardMaterial.fromType(...)`。

### 3) 材质是否受光照影响，要和场景光照一起看

材质表达并不是孤立的。至少要联动考虑：

-   `map.enableLight()`
-   具体光源
-   材质自己的 `useLight`
-   是否有法线 / 法线贴图
-   是否参与水面反射折射

### 4) 贴图与环境贴图要当资源系统处理

高频贴图入口：

-   `TextureLoader.load(...)`
-   HDR / envMap 资源
-   normal / alpha / roughness / metallic 等贴图

不要把它们写成每次交互都重复创建的临时对象。

## Demo-backed Patterns

### 模式 1：`StandardMaterial` 做通用表达

对应 demo：`demo/html/StandardMaterial.html`

它证明了：

-   普通颜色/贴图材质可走 `StandardMaterial`
-   扫描、扩散环这类业务特效也可走 `StandardMaterial.fromType(...)`
-   且可与 `adaptTerrain` 结合

### 模式 2：`PhongMaterial` 做经典高光材质

对应 demo：`demo/html/PhongMaterial.html`

高频参数：

-   `baseColor`
-   `specularColor`
-   `emissive`
-   `shininess`
-   `baseColorMap`
-   `emissiveIntensity`

### 模式 3：`PhysicalMaterial` 做 PBR

对应 demo：`demo/html/PBRMaterial.html`

高频参数：

-   `baseColor`
-   `metallic`
-   `roughness`
-   `envMap`
-   `normalMap`
-   `alphaMap`
-   `emissive`
-   `clearcoat`
-   `ior`

这条路径适合更强调材质真实感的对象。

### 模式 4：线材不要硬套面材

对应 demo：`demo/html/PolylineMaterial.html`

线流动、箭头、重复纹理，应优先 `PolylineMaterial`，不要勉强拿面材来模拟。

## Material Selection Guide

### 用 `StandardMaterial`，如果你要：

-   普通 primitive 颜色/贴图
-   快速业务特效原型
-   扫描、扩散环等内置风格

### 用 `PhongMaterial`，如果你要：

-   经典镜面高光
-   参数相对简单直观

### 用 `PhysicalMaterial`，如果你要：

-   PBR 表现
-   环境贴图
-   metallic / roughness 工作流

### 用 `PolylineMaterial`，如果你要：

-   流动线
-   箭头线
-   线纹理重复

## Strict Constraints

### 1. 不要先上最复杂材质

默认先 `StandardMaterial`，只有确实需要 PBR 或经典高光时再切换。

### 2. 材质调不对，先查光照

很多“材质不生效”的根因其实是：

-   没开光照
-   光源方向不对
-   没有法线/法线贴图

### 3. 贴图要复用

尤其是：

-   `baseColorMap`
-   `normalMap`
-   `envMap`

不要在鼠标事件里重复装载。

### 4. `WaterMaterial` 不再塞进通用材质 skill 里展开

它已经有独立 skill，应通过互链使用。

## Common Failure Cases

-   用 `PhysicalMaterial` 做简单单色物体，徒增复杂度
-   忽略 `envMap` 却追求高质量 PBR 观感
-   把线材需求写成面材需求
-   材质参数改了很多，但场景根本没开光照
-   高频交互里反复 `TextureLoader.load(...)`

## Performance Discipline

-   材质从简到繁：`Standard` → `Phong` → `Physical`
-   贴图和环境图尽量复用
-   高质量材质、阴影、SSR 联动时要一起测预算

## Demo References

-   `demo/html/StandardMaterial.html`
-   `demo/html/PhongMaterial.html`
-   `demo/html/PBRMaterial.html`
-   `demo/html/PolylineMaterial.html`

## See Also

-   `minemap-primitives-and-materials`
-   `minemap-lighting-and-shadows`
-   `minemap-water-surface-and-refraction`
-   `minemap-primitive-adapt-terrain`
