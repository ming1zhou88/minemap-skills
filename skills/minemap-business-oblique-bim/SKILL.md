---
name: minemap-business-oblique-bim
description: MineMap 倾斜摄影、BIM、glTF 与 3D Tiles 业务规范。用于园区、城市实景、厂区 BIM、单体模型装载。
---

# MineMap Business: Oblique and BIM

## Quick Start

```javascript
map.on("load", () => {
	map.addSceneComponent({
		id: "factory-bim",
		type: "3d-model",
		data: "/assets/factory.glb",
		position: new minemap.Math.Vector3(116.39, 39.9, 0),
		rotation: [90, 0, 0],
		scale: [1, 1, 1],
		allowPick: true
	});

	map.addSceneComponent({
		id: "city-oblique",
		type: "3d-tiles",
		urls: [{ url: "/tiles/city/tileset.json", name: "city" }],
		maximumScreenSpaceError: 8,
		maximumMemoryUsage: 512,
		allowPick: true
	});
});
```

## Architecture Positioning

该业务包严格对应 MineMap 的三维组件主入口 `map.addSceneComponent()`。

源码层面的稳定事实：

-   `3d-model` 走 `SceneModel`
-   `3d-tiles` 走 `SceneTileset`
-   历史 `addSource(type="3d-tiles")`、`addLayer(type="3d-tiles")` 已被明确标注为废弃提示

因此，新业务代码应统一采用：

1. `Map` 初始化
2. `load` 或 `style.load` 后添加三维组件
3. 用 `getSceneComponent()` 管理对象生命周期

## Decision Matrix

### 何时用 `3d-model`

适用：

-   单栋 BIM
-   单设备 / 单构件 / 单车体 / 单机房模型
-   需要实例化批量摆放的重复模型
-   需要精确控制 `matrix`、局部动画、最小屏幕像素尺寸时

优先能力：

-   `minimumPixelSize`
-   `activeAnimation`
-   `accurateBounding`
-   `distanceDisplayCondition`
-   `positions/rotations/scales/instanceNames` 实例化
-   `allowWaterReflect / allowWaterRefract`

### 何时用 `3d-tiles`

适用：

-   倾斜摄影
-   大范围 BIM 金字塔调度
-   点云场景
-   需要 LOD 调度、内存淘汰、按屏幕误差控制精度时

优先能力：

-   `maximumScreenSpaceError`
-   `maximumMemoryUsage`
-   `schedulingPolicy`
-   `skipLevelOfDetail`
-   `immediatelyLoadDesiredLevelOfDetail`
-   `pointSize`、`customizedColor`（点云）
-   `geoBoundsOptions`

## `SceneModel` Professional Guide

### 变换优先级

-   如果传 `matrix`，则应认为 `position / rotation / quaternion / scale` 不再作为主控制入口
-   若要做复杂 BIM 姿态修正，优先：
    -   先用 `Transforms.cartographicToFixedFrame()` 建锚点矩阵
    -   再叠加旋转 / 缩放矩阵
    -   最后把总矩阵传给 `matrix`

### 推荐参数组合

#### 1. 单体 BIM

```javascript
map.addSceneComponent({
	id: "plant-bim",
	type: "3d-model",
	data: "/bim/plant.glb",
	position: new minemap.Math.Vector3(116.39, 39.9, 0),
	rotation: [90, 0, 0],
	scale: [1, 1, 1],
	allowPick: true,
	minzoom: 14,
	maxzoom: 24
});
```

#### 2. 重复设备实例化

-   `positions` 必填
-   `rotations/scales/colors/shows/instanceNames` 选填
-   重复设备不要拆成大量单独 `3d-model`

### 需要谨慎开启的项

-   `accurateBounding=true`：可解决动画引起的包围盒不准，但会增加运行时成本
-   `minimumPixelSize`：能保住远景可见性，但会改变真实尺度感知
-   `activeAnimation`：只针对 glTF 自带动画，不等于场景轨迹动画

## `SceneTileset` Professional Guide

### 精度与成本

-   `maximumScreenSpaceError` 越小越清晰，网络与内存压力越高
-   业务初始建议：
    -   倾斜摄影：`8 ~ 16`
    -   大范围城市：`12 ~ 20`
    -   点云精看：`6 ~ 12`

### 内存策略

-   `maximumMemoryUsage` 是核心控制项
-   源码默认值在 `SceneTileset` 中为 `256` MB 级别逻辑
-   移动端 / 低端设备要显式收紧

### 调度策略

支持：

-   `SchedulingPolicyType.EFFECTIVENESS_FIRST`
-   `SchedulingPolicyType.BALANCED`
-   `SchedulingPolicyType.EFFICIENCY_FIRST`

建议：

-   默认先用 `BALANCED`
-   网络差但显存尚可：偏 `EFFECTIVENESS_FIRST`
-   大场景稳定浏览：偏 `EFFICIENCY_FIRST`

### 不建议默认开启的项

-   `skipLevelOfDetail`
-   `immediatelyLoadDesiredLevelOfDetail`

原因：源码注释已明确提示，这类参数虽然可能提高某些命中率，但会带来模糊或移动时加载变慢的问题。

### 多份 3D Tiles 飞行定位

如果一次添加多份 `urls`：

-   `geoBoundsOptions.index`：指定飞向哪一份
-   `geoBoundsOptions.combine`：合并边界后飞行

该逻辑直接影响 `flyTo/jumpTo/easeTo` 针对 tileset 的定位结果。

## Recommended Business Workflow

### 厂区 / 园区 BIM

1. 主建筑主体用 `3d-model`
2. 重复设备用实例化 `3d-model`
3. 大范围实景底座若存在，用 `3d-tiles`
4. 交互选中统一通过 `allowPick + getFeatureByGPUPick()` 做

### 城市倾斜

1. 倾斜主体用 `3d-tiles`
2. 楼宇专题高亮或补模用 `3d-model`
3. 控制 `maximumScreenSpaceError` 与 `maximumMemoryUsage`
4. 对远景区域用 `minzoom/maxzoom` 控显

## Hard Constraints and Pitfalls

-   `minzoom/maxzoom` 只控制显示，不等于立即释放资源
-   `backCulling=false` 只在确有背面缺失问题时开启
-   点云 `customizedColor` 只适用于点云，不应滥用到普通 mesh 业务描述中
-   `allowPick=false` 的对象，如果业务依赖拾取，会直接失效
-   旧版 source/layer 方式不要继续扩散到新页面和新模板中

## See Also

-   `minemap-scene-components`
-   `minemap-events-and-picking`
-   `minemap-performance-and-backend`
