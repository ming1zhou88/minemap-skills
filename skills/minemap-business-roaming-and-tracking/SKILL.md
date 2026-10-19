---
name: minemap-business-roaming-and-tracking
description: MineMap 轨迹动画、模型漫游、相机跟踪业务规范。用于巡检、车辆行驶、无人机路径回放、数字孪生漫游。
---

# MineMap Business: Roaming and Tracking

## Quick Start

```javascript
map.animationManager.animationTracking = new minemap.AnimationTracking({
	name: "car-1",
	mode: minemap.AnimationTrackingMode.LOCK,
	effects: minemap.AnimationTrackingEffects.PRECISE,
	distance: 100,
	enabled: true,
	headingPitchRoll: minemap.Math.HeadingPitchRoll.fromDegrees(0, 65, 0)
});
```

## Core Animation Entry Points

MineMap 轨迹与漫游有两套常用入口，不能混为一谈：

### 1. `map.animationManager`

适合：

-   模型轨迹动画
-   实例化模型动画
-   primitive 动画
-   extrusion 生长动画
-   JSON 配置驱动的业务回放

关键方法：

-   `load(animationJson)`
-   `add(animationJson)` -> 返回 `Promise`
-   `remove()` / `removeById()` / `removeAll()`

### 2. 跟踪配置

有两种来源：

-   `map.animationManager.animationTracking`
-   `map.trackingOptions`

前者更偏引擎统一跟踪配置；后者常见于业务动画配置和旧文档习惯。

## Supported Animation Targets

源码处理逻辑覆盖：

-   `model`
-   `modelInstance`
-   `primitive`
-   `primitiveInstance`
-   `extrusion`
-   billboard 轨迹对象

因此业务侧不要只把它理解成“车模移动”。

## Tracking Modes

-   `NONE`：关闭跟踪
-   `FREE`：自由观察，动画播放但相机不强锁
-   `FOLLOW`：跟随目标方向变化
-   `LOCK`：锁定目标视角，适合巡检、车载、飞行器回放

## Recommended Workflow

1. 先创建业务对象（通常是 `SceneModel`）
2. 确保对象已经可获取：`map.getSceneComponent(id)`
3. 准备动画 JSON 或关键帧 `AnimationClip`
4. `await map.animationManager.add(...)`
5. 设置 `animationTracking` 或 `map.trackingOptions`
6. 再调用 `play()`

## JSON-driven Animation Notes

`AnimationManager.add()` 支持：

-   完整 JSON 文档
-   单个 content 配置
-   JSON URL

业务上建议：

-   回放数据使用 JSON
-   即时生成演示可直接构造 clip
-   大批量动画统一走 JSON，便于留档和重放

## 实例化模型实时轨迹

增量推送的位置更新与 JSON 关键帧回放是两套接口。对不断收到车辆 / 设备状态的场景，使用 `map.animationManager.addRealtime()`，并让 `sourceId` 指向已初始化的 `3d-model-instance` 场景对象：

```javascript
const realtime = map.animationManager.addRealtime({
	id: "fleet-live",
	sourceId: "fleet-model",
	duration: 1600,
	maxInstances: 500
});

realtime.update([
	{ trackId: "car-01", position: [116.39, 39.90, 0], rotation: [90, 0, 90], timestamp: Date.now() }
]);
```

- `trackId` 是实例名；每次 `update()` 只传本轮变化的实例即可。
- 默认把 `position` 当作 `[longitude, latitude, height]` 并转成笛卡尔坐标；`update(items, { cartesian: true })` 用于已是 Cartesian 的位置。
- `duration` 使用毫秒，默认 1000；`easing` 可选 `linear`，其余取 `easeInOut`。
- `timestamp` 不大于该轨迹已接收时间戳的更新会被忽略，适合过滤乱序数据。
- 实例数最多 500；超出上限时更新会报错。需要退出时用 `map.animationManager.removeRealtime(id)`，它会销毁控制器并移除其管理的实例。
- 若启用跟随，把 `animationTracking.name` 设为对应 `trackId`，并使用 `FOLLOW` 模式。

不要对每条实时数据新建一个 JSON animation。也不要让同一 `trackId` 的实例同时由该控制器和业务代码独立销毁或复用；控制器移除轨迹时会从目标 collection 移除对应实例。

## Demo-backed Patterns

### 模式 1：批量动画内容一次加入

对应 demo：`demo/html/AnimationTrack.html`

这个 demo 证明 `map.animationManager.add(animationJson)` 返回的是 `Promise`，解析后可得到一个或多个动画对象，然后逐个 `play()`。

这意味着业务上可以把：

-   模型
-   模型实例
-   billboard
-   primitive

放进同一份 animation 文档里统一驱动。

### 模式 2：模型轨迹回放 + 跟踪

对应 demo：`demo/html/AutoHeightAnimation.html`

这个 demo 的关键结论：

-   自由模式不需要显式设置 `FREE`，直接 `animationTracking.enabled = false` 即可
-   跟随模式与锁定模式都可直接改写 `map.animationManager.animationTracking`
-   轨迹回放常配 `autoHeight: true`
-   `AnimationBlender` 支持 `play()`、`pause()`、`stop()`、`reset()`、`speed`、`currentTime`
-   `onPlaying` 可用于驱动时间轴 UI

### 模式 3：实例化车队稀疏更新

对应 demo：`demo/html/RealtimeAnimationTrack.html`。它展示 `addRealtime()`、按 `trackId` 稀疏更新、时间戳过滤、Marker 跟随同步和实例点击选择。这个 demo 的更新队列是业务示例；生产系统应由自己的消息通道提供状态，并按需节流推送。

## Professional Recommendations

### 巡检车 / 无人机回放

-   目标模型 id 要稳定，不要临时随机变更
-   相机模式优先 `LOCK`
-   姿态使用 `HeadingPitchRoll`，其中 `roll` 一般保持 0

### 城市运营漫游

-   主角模型做轨迹动画
-   其他环境对象静态保留
-   避免同时播放大量复杂骨骼动画模型

### 实例化车队轨迹

-   使用 `modelInstance`
-   `trackingModel` 要指向实例名 `instanceName`，不是根模型 id

### 时间轴回放

-   用 `onPlaying` 回调同步 UI
-   允许人工拖动 `currentTime` 做精确回看
-   切换场景前主动 `removeAll()` 清空动画状态

## Runtime Control Surface

从 demo 看，正式业务通常要暴露这些运行时控制：

-   `play()`
-   `pause()`
-   `stop()`
-   `reset()`
-   `speed`
-   `currentTime`
-   `animationTracking.enabled`
-   `animationTracking.mode`
-   `animationTracking.effect`
-   `animationTracking.headingPitchRoll`
-   `animationTracking.distance`

## Failure Cases to Avoid

-   未等模型加载完成就添加动画
-   `trackingModel` 填错：根 id 与实例 id 混用
-   动画很多但不及时 `removeAll()`，导致回放切场景时残留状态
-   把 glTF 自带动画与场景轨迹动画混为一层控制
-   只创建动画却不保存 `AnimationBlender`，后续无法暂停、调速、拖时间轴
-   自由模式与跟踪模式切换时忘记同步 `animationTracking.enabled`

## Minimal Safe Pattern

```javascript
const [blender] = await map.animationManager.add(animationJson);
map.animationManager.animationTracking.name = blender.id;
map.animationManager.animationTracking.mode = minemap.AnimationTrackingMode.FOLLOW;
map.animationManager.animationTracking.enabled = true;
blender.play();
```

## Demo References

-   `demo/html/AnimationTrack.html`
-   `demo/html/AutoHeightAnimation.html`
-   `demo/html/AutoHeightAnimationInstance.html`
-   `demo/html/RealtimeAnimationTrack.html`

## See Also

-   `minemap-transforms`
-   `minemap-math-foundations`
-   `minemap-business-oblique-bim`
-   `minemap-business-airline-and-lines`
-   `minemap-events-and-picking`
