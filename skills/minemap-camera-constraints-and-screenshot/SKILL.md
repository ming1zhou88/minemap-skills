---
name: minemap-camera-constraints-and-screenshot
description: MineMap 相机限制与截图规范。用于近裁剪面阈值、区域相机限制、截图导出与相机交互约束。
---

# MineMap Camera Constraints and Screenshot

## Architecture Positioning

这里混合了两类能力：

1. 引擎正式公开 API
2. demo 层相机限制工具

正式公开的核心是：

-   `map.setNearClippingPlaneMinThreshold(...)`
-   `map.printScreen(range, download)`

而 `CameraRegionLimiter` 是 demo 提供的辅助工具，不是内核导出主类。

## Source-backed Entry Surface

-   `map.setNearClippingPlaneMinThreshold(num)`
-   `map.printScreen(range, download?)`
-   `map.getCameraPosition()`
-   `map.getCameraBearing()`
-   `map.getCameraPitch()`

## Demo-backed Patterns

### 模式 1：近裁剪面阈值用于减轻深度闪烁

对应 demo：

-   `demo/html/CameraWithNearThreshold.html`

源码说明很明确：

-   近裁剪面越大，远处深度分辨率越高
-   过大又会导致相机穿透模型

所以它是平衡项，不是越大越好。

### 模式 2：截图核心是传像素范围

对应 demo：

-   `demo/html/PrintScreen.html`

正式入口：

-   `map.printScreen(range, true)`

这个接口需要的是像素坐标列表，不要求你必须用 demo 的同一套交互采样方式。

### 模式 3：相机区域限制属于 demo 工具层

对应 demo：

-   `demo/html/CameraRegionLimiter.html`
-   `demo/js/CameraRegionLimiter.js`

它的真实定位是：

-   基于区域 polygon + 高度范围生成 OBB
-   在 `move` 事件里检查并回退相机

因此它更适合写成业务工具类，而不是描述为内核内建能力。

## Strict Constraints

### 1. `setNearClippingPlaneMinThreshold()` 只接受正值

源码对 `<= 0` 直接忽略。

### 2. 截图不要再依赖 `preserveDrawingBuffer`

源码注释已说明该路径不再推荐，正式做法应走 `printScreen()`。

### 3. `CameraRegionLimiter` 依赖内部相机状态

它是 demo 工具，可以复用思路，但不要把它误记成 `minemap.CameraRegionLimiter`。

## Failure Cases

-   为消闪烁把 near 阈值调得过大
-   误以为截图只能截整个 canvas
-   把 demo 工具类当成内核 API 直接宣传

## Demo References

-   `demo/html/CameraWithNearThreshold.html`
-   `demo/html/PrintScreen.html`
-   `demo/html/CameraRegionLimiter.html`
-   `demo/js/CameraRegionLimiter.js`

## See Also

-   `minemap-fundamentals`
-   `minemap-events-and-picking`
-   `minemap-performance-and-backend`
