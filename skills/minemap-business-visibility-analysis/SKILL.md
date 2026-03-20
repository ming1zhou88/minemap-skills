---
name: minemap-business-visibility-analysis
description: MineMap 可视域与通视分析业务规范。用于安防、选址、观测点视域评估、遮挡判定。
---

# MineMap Business: Visibility Analysis

## Quick Start

```javascript
const viewshed = new minemap.Viewshed3D({
	viewerPosition: new minemap.Math.Vector3(116.39, 39.9, 20),
	direction: 0,
	pitch: 90,
	distance: 200,
	horizontalFov: 60,
	verticalFov: 60,
	visibleAreaColor: "rgba(0,255,0,0.3)",
	hiddenAreaColor: "rgba(255,0,0,0.3)"
});
map.analysis.viewshed3DCollection.add(viewshed);
```

## Choosing the Right Tool

### `Viewshed3D`

适合：

-   单观察点扇形视域
-   安防岗亭 / 摄像头 / 观景点可视覆盖
-   需要“可见区域 / 不可见区域”面结果表达

### `Sightline`

适合：

-   单点到多目标点的通视判断
-   目标清单式遮挡评估
-   线路可见 / 不可见分段表达

## `Viewshed3D` Strict Constraints

-   `viewerPosition` 应为 `Vector3`
-   `direction` 必须在 $[0, 360]$
-   `pitch` 必须在 $[0, 180]$
-   `horizontalFov` / `verticalFov` 必须在 $[0, 180]$
-   `distance > 0`

任何超界值都应在业务入参层先兜底，不要把异常留给渲染时。

## `Sightline` Strict Constraints

-   `viewerPosition` 是唯一观察点
-   `targets` 是目标点集合
-   每个目标点都要带稳定 `name`
-   如果业务需要屏幕前景遮挡判定，可启用 `mainCameraInvisibleDetection`

## Demo-backed Patterns

### 模式 1：参数化可视域

对应 demo：`demo/html/Viewshed3D.html`

它清楚展示了 `Viewshed3D` 的两种工作流：

1. 参数直接创建
2. 鼠标交互创建

并且 demo 验证了这些集合操作是正式可用的：

-   `getByIndex()`
-   `getById()`
-   `remove()`
-   `removeById()`
-   `removeByIndex()`
-   `removeAll()`
-   `locateToViewer()`

### 模式 2：鼠标驱动通视分析

对应 demo：`demo/html/Sightline.html`

这个 demo 的实战价值很高：

1. 第一次点击确定视点
2. 后续点击增加目标点
3. `mousemove` 做预览目标点
4. `contextmenu` 结束当前绘制

这正是业务里最常见的“交互式通视”流程。

## Standard Pattern

```javascript
const sightline = new minemap.Sightline({
	viewerPosition: new minemap.Math.Vector3(116.39, 39.9, 20),
	targets: [{ name: "tower-a", position: [116.391, 39.901, 15] }],
	visibleColor: "green",
	hiddenColor: "red",
	indicatorSize: 10,
	mainCameraInvisibleDetection: true
});
map.analysis.sightlineCollection.add(sightline);
```

## Interactive Construction Workflow

推荐交互方式：

1. 点击获取观察点
2. 移动鼠标实时预览目标点
3. 再确认分析参数

配合方式：

-   `map.painter.getPositionByGPUPick(...)`
-   或业务统一拾取封装

更细一点的建议：

### `Viewshed3D`

1. 先创建对象，初始只给一个很小的 `distance`
2. 第一次点击设置 `viewerPosition`
3. 鼠标移动时用 `setTargetPoint()` 驱动方向与距离
4. 右键结束绘制

### `Sightline`

1. 第一次点击设观察点
2. 后续点击逐个加入目标点
3. 鼠标移动阶段用临时目标点预览
4. 在确认目标后再固化结果

## Runtime Tuning

demo 也证明这些参数适合在运行时实时调：

### `Viewshed3D`

-   `viewerPosition`
-   `distance`
-   `direction`
-   `pitch`
-   `horizontalFov`
-   `verticalFov`
-   `visibleAreaColor`
-   `hiddenAreaColor`
-   `lineColor`

### `Sightline`

-   `visibleColor`
-   `hiddenColor`
-   `lineWidth`
-   `mainCameraInvisibleDetection`
-   `indicatorColor`
-   `indicatorOutlineColor`
-   `indicatorOutlineWidth`
-   `indicatorSize`

## Performance Discipline

-   多个 `Viewshed3D` 并发成本明显高于普通图层
-   大屏项目中建议：
    -   当前激活对象 1~3 个
    -   历史分析结果做缓存或静态化
    -   非当前对象先隐藏，不要全部实时更新

## Common Failure Cases

-   用经纬度数组直接替代 `Vector3` 导致类型不一致
-   不断新建分析对象而不是复用已有对象
-   视域角度给成业务度数但没有校验边界
-   同时打开过多分析对象导致帧率不稳定
-   `Sightline` 目标点没有稳定 `name`，导致删改目标点时不好管理
-   鼠标预览点不及时移除，导致临时目标点残留

## Demo References

-   `demo/html/Viewshed3D.html`
-   `demo/html/Sightline.html`

## See Also

-   `minemap-business-video-projection`
-   `minemap-business-flood-and-shadow`
-   `minemap-terrain-and-analysis`
