---
name: minemap-business-video-projection
description: MineMap 视频投射与视频管理业务规范。用于监控视域、摄像头覆盖模拟、视频贴投到模型/地表/3D Tiles。
---

# MineMap Business: Video Projection

## Quick Start

```javascript
const videoProjection = new minemap.VideoProjection({
	location: [116.39, 39.9],
	altitude: 80,
	fov: 45,
	aspect: 16 / 9,
	near: 0.1,
	far: 200,
	heading: 0,
	pitch: 0,
	roll: 0,
	used3DModel: true,
	used3DTiles: true,
	usedRaster: true,
	usedPolygon: true,
	videoTextureFun: () => videoElement,
	debugVideo: true
});

map.videoManager.add(videoProjection);
```

## Core Model

视频投射在 MineMap 中不是普通贴图替换，而是一套“投影相机 + 视频纹理 + 命中目标类型筛选”的业务能力。

关键对象：

-   单实例：`VideoProjection`
-   管理器：`map.videoManager`

## Required Inputs

### 必填/强约束

-   `location`：必须是经纬度数组
-   `videoTextureFun`：返回视频源对象的函数

### 常用姿态参数

-   `altitude`
-   `heading`
-   `pitch`
-   `roll`
-   `fov`
-   `aspect`
-   `near`
-   `far`

## Projection Target Switches

源码中支持单独启停：

-   `used3DModel`
-   `used3DTiles`
-   `usedPolygon`
-   `usedRaster`

业务建议：

-   室内 BIM 监控：优先 `used3DModel`
-   倾斜园区：优先 `used3DTiles`
-   二三维叠合监管：保留 `usedRaster`

## Demo-backed Patterns

### 模式 1：标准视频投影对象

对应 demo：`demo/html/VideoProjection.html`

这个 demo 已经给出最标准的业务链路：

1. 创建隐藏 `video` DOM
2. `videoTextureFun` 返回该 video
3. 创建 `VideoProjection`
4. `map.videoManager.add(videoProjection)`
5. 用 GUI 实时调 `location / altitude / heading / pitch / roll / near / far / fov / aspect / alpha`
6. 按目标类型实时切换 `used3DModel / used3DTiles / usedPolygon / usedRaster`

另外它还说明，接收投影的自定义 `Primitive` 必须带上：

-   `usedVideoProjection: true`

否则不会按预期参与投影。

### 模式 2：立面投影

对应 demo：`demo/html/VideoProjectionOnWall.html`

适合：

-   建筑外立面投影
-   室内墙面投影
-   宣传屏 / 大屏幕落墙模拟

这个 demo 的高价值点在于：

-   切换投影位前先 `map.videoManager.removeAll()`
-   通过业务几何先构造投影接收平面
-   投影目标只开 `usedPolygon: true`
-   一个投影位对应一套参数缓存，可重复进入复用

### 模式 3：raster 投影与抖动排查

对应 demo：`demo/html/VideoProjectionOnRaster.html`

业务含义不是“推荐直接改私有字段”，而是说明：

-   raster 投影容易在瓦片拼接处暴露问题
-   `debugVideo` 和调试视锥对排查非常关键
-   投影位姿一旦频繁变化，更需要先验证接收目标是否正确

## Zoom Gating

`VideoProjection` 自带 `minzoom/maxzoom` 门限。

这是业务上非常重要的成本控制手段：

-   远景不显示，避免无意义投影
-   近景再启用，减少纹理与计算压力

强约束：`minzoom` 不能大于 `maxzoom`。

## Debug Strategy

开启 `debugVideo` 后，引擎会显示调试视锥体对象。

适用场景：

-   排查朝向错位
-   排查近远裁剪不合理
-   排查镜头覆盖范围与真实业务不一致

不要在正式大屏长期开启。

## Receiver Preparation

投影永远不是“只创建投影对象就完了”，还必须确认接收体已经具备接收资格：

-   自定义 `Primitive`：需要显式标记 `usedVideoProjection: true`
-   立面投影：优先先构造明确接收面，再让投影命中该面
-   业务切换投影位时，优先先清空旧实例，再添加新实例

如果接收体没有准备好，先检查接收体，而不是先怀疑视频流。

## Video Manager Practice

### 添加顺序

```javascript
map.videoManager.add(videoProjection, 0);
```

### 管理顺序

-   `raise(video)`
-   `lower(video)`
-   `remove(video)`
-   `removeById(uniqueId)`
-   `removeAll()`

### 全局开关

```javascript
map.videoManager.enabled = false;
```

适合临时关闭所有监控投射能力。

业务上还应把它当作“实例生命周期管理器”：

-   场景切换：`removeAll()`
-   单路替换：`remove(video)` 后再 `add(video)`
-   多路并发：控制顺序与启停，不要默认全部常驻

## Professional Recommendations

### 安防监控

-   一个摄像头对应一个 `VideoProjection`
-   摄像头安装位统一用业务台账坐标驱动
-   开发联调期统一打开 `debugVideo`，上线关闭

### 园区视频上图

-   优先把静态底座放到 `3d-tiles`
-   每个视频实例精确控制目标类型，避免无关表面被投射

### 立面广告 / 墙面投影

-   接收目标优先做成单独 polygon，而不是把整个场景都开放给投影
-   切换投影点位时同步切换接收面与视频源

### raster 贴投

-   只在确有需要时启用 `usedRaster`
-   先排查瓦片拼接、相机位姿、近远裁剪，再谈画质

## Pitfalls

-   `videoTextureFun` 返回值不稳定，会造成纹理更新异常
-   视频路数过多会占用纹理单元，需分级启停
-   不要把视频投射当作大规模覆盖渲染手段，它更适合重点监控位
-   demo 中出现的 `_location`、`_altitude` 这类私有字段改写只可视作调试手段，正式业务应优先走公开属性 `location`、`altitude`、`heading`、`pitch`、`roll`

## Demo References

-   `demo/html/VideoProjection.html`
-   `demo/html/VideoProjectionOnWall.html`
-   `demo/html/VideoProjectionOnRaster.html`

## See Also

-   `minemap-business-visibility-analysis`
-   `minemap-business-oblique-bim`
-   `minemap-performance-and-backend`
