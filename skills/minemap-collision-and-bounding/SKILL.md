---
name: minemap-collision-and-bounding
description: MineMap 包围体与碰撞检测规范。用于 AABB/OBB/包围球相交判断、BVH 碰撞、角色碰撞与调试边界框。
---

# MineMap Collision and Bounding

## Architecture Positioning

这类能力分两层：

1. 数学/几何层相交判断
2. 地图运行时 BVH 碰撞与角色碰撞

源码与导出表明，MineMap 已公开：

-   `AxisAlignedBoundingBox`
-   `OrientedBoundingBox`
-   `BoundingSphere`
-   `BoundingBoxIntersection`
-   `map.setCollisionDetection(...)`
-   `map.collisionDetection`

## Source-backed Entry Surface

-   `new minemap.AxisAlignedBoundingBox(...)`
-   `new minemap.OrientedBoundingBox(...)`
-   `new minemap.BoundingSphere(...)`
-   `minemap.BoundingBoxIntersection.*`
-   `map.setCollisionDetection(options)`
-   `map.collisionDetection.showBvhBoundingBox`

## Demo-backed Patterns

### 模式 1：静态相交判断优先用公开数学对象

对应 demo：

-   `demo/html/BoundingBoxIntersectionDemo.html`
-   `demo/html/AABBIntersectionDemo.html`
-   `demo/html/OBBIntersectionDemo.html`
-   `demo/html/BoundingSphereIntersectionDemo.html`

稳定调用：

-   `BoundingBoxIntersection.intersects(...)`
-   `BoundingBoxIntersection.aabbIntersectsAabb(...)`
-   `BoundingBoxIntersection.obbIntersectsObb(...)`
-   `BoundingBoxIntersection.sphereIntersectsSphere(...)`

### 模式 2：世界空间包围体先构造再判断

这些 demo 都说明一个点：

-   包围体判断之前，先把位置、姿态和尺度折算到当前空间

不要把局部模型尺寸直接拿去做世界碰撞。

### 模式 3：运行时角色碰撞走 `setCollisionDetection(...)`

对应 demo：

-   `demo/html/BVHCollisionDetection.html`

关键参数：

-   `source`
-   `targets`
-   `bvhBoundingBoxDepth`
-   `showBvhBoundingBox`

demo 还验证了：

-   `collisionDistance`
-   重力
-   第一人称
-   角色包围框显示

## Strict Constraints

### 1. 几何相交判断和地图碰撞系统不是一回事

-   前者是数学判断
-   后者是运行时角色/场景碰撞

### 2. BVH 调试框只能用于调试

`showBvhBoundingBox` 不是正式业务显示层。

### 3. `targets: 'keep'` 只在你明确理解复用碰撞体时使用

否则容易让碰撞目标状态失真。

## Failure Cases

-   用 AABB 心智处理已经旋转过的对象
-   忽略姿态变换，直接比较原始尺寸
-   长期开启 BVH 调试显示

## Demo References

-   `demo/html/BoundingBoxIntersectionDemo.html`
-   `demo/html/AABBIntersectionDemo.html`
-   `demo/html/OBBIntersectionDemo.html`
-   `demo/html/BoundingSphereIntersectionDemo.html`
-   `demo/html/BVHCollisionDetection.html`

## See Also

-   `minemap-math-foundations`
-   `minemap-transforms`
-   `minemap-business-roaming-and-tracking`
