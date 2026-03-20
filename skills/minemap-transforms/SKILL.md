---
name: minemap-transforms
description: MineMap 坐标变换规范。覆盖 `cartographicToCartesian3()`、`cartographicToFixedFrame()`、`headingPitchRollToFixedFrame()`、`fixedFrameToHeadingPitchRoll()` 等高频变换入口。
---

# MineMap Transforms

## Quick Start

```javascript
const anchor = new minemap.Math.Vector3(116.39, 39.9, 10);
const matrix = minemap.Transforms.cartographicToFixedFrame(anchor);

const primitive = new minemap.Primitive({
	modelMatrix: matrix,
	geometry,
	material
});

map.addPrimitive(primitive);
```

## Core Concepts

### 1) `Transforms` 负责坐标系之间的转换

`minemap.Transforms` 不是普通数学库，而是 MineMap 的场景坐标变换入口。

它主要处理：

-   经纬高 ↔ 笛卡尔坐标
-   局部 ENU 坐标 → 固定坐标矩阵
-   姿态角 → 固定坐标矩阵

### 2) 最高频的 3 个入口

最常用的是：

-   `cartographicToCartesian3(...)`
-   `cartographicToFixedFrame(...)`
-   `headingPitchRollToFixedFrame(...)`

推荐理解：

-   只想拿空间点 → `cartographicToCartesian3`
-   想给 primitive 生成锚点矩阵 → `cartographicToFixedFrame`
-   想把锚点 + 姿态一起生成矩阵 → `headingPitchRollToFixedFrame`

### 3) `cartographicToFixedFrame(...)` 是业务主入口

在实际 demo 和现有 skill 中，最常见的 `Transforms` 用法就是：

-   先把经纬高转成固定坐标锚点矩阵
-   再作为 `modelMatrix` 挂给 primitive 或其他低层对象

### 4) 姿态矩阵与姿态角是两回事

如果当前有：

-   `HeadingPitchRoll`

想得到矩阵，用：

-   `headingPitchRollToFixedFrame(...)`

如果已经有固定坐标矩阵，想回推出姿态角，用：

-   `fixedFrameToHeadingPitchRoll(...)`

## Source-backed API Map

高频公开方法：

-   `cartesian3ToCartographic(...)`
-   `cartographicToCartesian3(...)`
-   `cartographicToFixedFrame(...)`
-   `cartesianToFixedFrame(...)`
-   `localFrameToFixedFrame(...)`
-   `headingPitchRollToFixedFrame(...)`
-   `fixedFrameToHeadingPitchRoll(...)`

## Demo-backed Patterns

### 模式 1：模型 GUI 输入先转笛卡尔

对应 demo：`demo/html/3DModelTransform.html`、`demo/html/3DModelTransformBounding.html`

这两个 demo 体现了一个很稳定的实践：

-   UI 里持有经纬高
-   写回对象前先 `cartographicToCartesian3(...)`

这样可以避免把表单输入和内部空间坐标混写。

### 模式 2：Primitive 通过固定坐标矩阵定位

对应 demo：`demo/html/ModelTransformControl.html`

标准做法是：

1. 用 `minemap.Math.Vector3([lng, lat, height])` 组织锚点
2. 用 `minemap.Transforms.cartographicToFixedFrame(...)` 生成 `modelMatrix`
3. 把矩阵交给 primitive

### 模式 3：复杂姿态优先矩阵化

当业务开始涉及：

-   本地轴旋转
-   多段姿态叠加
-   保存 / 恢复对象姿态快照

就应优先围绕 `Transforms + Matrix4` 组织，而不是只改多个独立字段。

## Recommended Workflow

### Scene component

1. 数据层先持有经纬高
2. 普通定位可直接用 `position`
3. 复杂姿态再切到 `Transforms` 计算矩阵

### Primitive

1. 先确定锚点经纬高
2. `cartographicToFixedFrame(...)`
3. 将 geometry 放到该 `modelMatrix` 下

### 姿态回显

1. 若已有矩阵，先 `fixedFrameToHeadingPitchRoll(...)`
2. 再回填到业务表单或快照界面

## Strict Constraints

### 1. 输入语义必须匹配

-   `cartographicToCartesian3` 吃经纬高
-   `cartesianToFixedFrame` 吃笛卡尔坐标

不要混用。

### 2. 不要把 `Transforms` 当普通数学库

向量、矩阵、四元数本身的运算应放回 `minemap.Math`。

### 3. 有稳定 `modelMatrix` 后不要再多入口改姿态

一旦确定围绕矩阵维护，就尽量不要同时继续零散改 `position` / `rotation` / `scale`。

## Common Failure Cases

-   把经纬高直接当笛卡尔坐标传入
-   已经改成 `modelMatrix` 工作流，还继续零散改姿态字段
-   用数组与 `Vector3` 混写，导致语义不清
-   本该用 `headingPitchRollToFixedFrame(...)`，却自己硬拼旋转流程

## Demo References

-   `demo/html/3DModelTransform.html`
-   `demo/html/3DModelTransformBounding.html`
-   `demo/html/ModelTransformControl.html`

## See Also

-   `minemap-math-foundations`
-   `minemap-scene-components`
-   `minemap-primitives-and-materials`
-   `minemap-business-roaming-and-tracking`
