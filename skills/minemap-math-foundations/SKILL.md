---
name: minemap-math-foundations
description: MineMap 数学基础规范。覆盖 `Vector2/3/4`、`Matrix3/4`、`Quaternion`、`HeadingPitchRoll`、`Cartographic`、以及角度弧度转换的实际用法。
---

# MineMap Math Foundations

## Quick Start

```javascript
const position = minemap.Math.Vector3.fromDegrees(116.39, 39.9, 20);
const hpr = minemap.Math.HeadingPitchRoll.fromDegrees(0, 65, 0);

console.log(position, hpr);
```

## Core Concepts

### 1) `minemap.Math` 是业务参数类型系统

高频公开对象包括：

-   `Vector2`
-   `Vector3`
-   `Vector4`
-   `Cartographic`
-   `Matrix2`
-   `Matrix3`
-   `Matrix4`
-   `Quaternion`
-   `HeadingPitchRoll`
-   `Euler`
-   `toRadians(...)`
-   `toDegrees(...)`

这套对象在 MineMap 里不是底层细节，而是大量 API 的直接输入类型。

### 2) `Vector3` 是最常见核心类型

它高频用于：

-   位置
-   顶点
-   观察点
-   目标点
-   光源点

如果在写 MineMap 代码，`Vector3` 基本是最常见对象之一。

### 3) `fromDegrees*` 系列是经纬高构造主路径

高频入口：

-   `Vector3.fromDegrees(...)`
-   `Vector3.fromDegreesArrayHeights(...)`

适合：

-   点位构造
-   线面顶点批量构造
-   分析范围构造

### 4) 姿态不要总靠裸数组表达

当业务开始涉及：

-   航向 / 俯仰 / 横滚
-   四元数旋转
-   姿态矩阵

应优先使用：

-   `HeadingPitchRoll`
-   `Quaternion`
-   `Matrix4`

### 5) `Math` 与 `Transforms` 分工不同

-   `Math`：向量、矩阵、姿态对象本身
-   `Transforms`：不同坐标系之间的转换

## Demo-backed Patterns

### 模式 1：业务点位通常从 `Vector3` 开始

大量 demo 与现有 skill 都直接使用：

-   `new minemap.Math.Vector3(...)`
-   `Vector3.fromDegrees(...)`
-   `Vector3.fromDegreesArrayHeights(...)`

这说明 `Vector3` 是业务层主类型，而不是数学细枝末节。

### 模式 2：实例化姿态优先 `HeadingPitchRoll.fromDegrees(...)`

当前业务里只要姿态来自“度数”输入，就优先这样构造，能明显减少单位混淆。

### 模式 3：矩阵只在确实需要时接管

普通点位不必过早矩阵化。

但当你开始：

-   组合旋转
-   维护 `modelMatrix`
-   做局部坐标姿态

就应转向 `Matrix4`。

## Practical Selection Guide

### 用 `Vector2`

-   二维参数
-   平面辅助量

### 用 `Vector3`

-   位置
-   顶点
-   方向
-   光源点

### 用 `Vector4`

-   RGBA / 四维参数
-   齐次坐标

### 用 `HeadingPitchRoll`

-   相机或对象姿态
-   业务按度描述航向俯仰横滚

### 用 `Quaternion`

-   旋转组合
-   避免欧拉角问题

### 用 `Matrix4`

-   对象总变换
-   和 `Transforms` 联动
-   固定坐标矩阵表达

## Strict Constraints

### 1. 不要混淆“经纬高”和“世界笛卡尔”

它们都可能看起来是三个数，但语义完全不同。

### 2. 批量几何优先 `fromDegreesArrayHeights(...)`

不要手工一组组 new 大量 `Vector3`。

### 3. 角度与弧度要显式区分

有的 API 吃度数，有的对象内部用弧度。不要裸写魔法数后忘记单位。

### 4. 不要在明显需要对象语义时继续只写数组

当接口已经明显是向量、姿态、矩阵语义时，优先显式使用 `minemap.Math.*` 类型。

## Common Failure Cases

-   用经纬高数组冒充世界坐标
-   批量线面点位仍然手工 new 大量 `Vector3`
-   `HeadingPitchRoll` 与普通 `[x, y, z]` 旋转数组混着用
-   进入矩阵工作流后仍用零散数值硬拼姿态

## Demo References

-   `demo/html/3DModelTransform.html`
-   `demo/html/ModelTransformControl.html`
-   `demo/html/PolylineAdaptTerrain.html`

## See Also

-   `minemap-transforms`
-   `minemap-scene-components`
-   `minemap-primitives-and-materials`
-   `minemap-business-measurement`
-   `minemap-business-roaming-and-tracking`
