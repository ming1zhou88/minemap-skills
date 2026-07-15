---
name: minemap-s3m
description: MineMap S3M 倾斜摄影数据源。用于加载城市级三维模型（OSGB / S3M 切片 / 锦江示例）、S3M tileset 位置校正、太阳光联动与场景组件生命周期管理。
---

# MineMap S3M

## 概述

S3M 是 MineMap 4.25 公开支持的倾斜摄影 / 城市级三维模型数据格式。入口统一在 `map.addSceneComponent({ type: 's3m', ... })`，与已有的 3D Tiles / scene-object 体系保持一致。

`addSceneComponent` 会返回一个 S3M tileset 对象，具备 `readyPromise`，可用于在数据加载完成后定位相机。

## Quick Start

```javascript
const tileset = map.addSceneComponent({
	id: "cbd-s3m",
	type: "s3m",
	url: "../data/s3m/CBD/cbd.scp" // S3M 配置 / 服务地址
});

tileset.readyPromise.then(() => {
	// 加载完成后定位相机
	map.transform.activeCamera.setView({
		destination: new minemap.Math.Vector3(-2181968.890329965, 4385313.17843029, 4072712.8241634783),
		orientation: {
			heading: 3.1756648661534443,
			pitch: -0.3715184468182904
		}
	});
});
```

## Architecture Positioning

-   S3M 走的是 `scene-component` 体系，不是 source/layer
-   内部会加载 `url` 指定的 S3M 配置，解析 OSGB / crn 等切片资源
-   默认开启双面绘制，可应对建筑内壁 / 倾斜摄影背面观察
-   与 `enableLight` / `setSunLightTimestamp` 协同：S3M 材质本身依赖光照计算

## 关键参数

| 参数   | 说明                                                                  |
| ------ | --------------------------------------------------------------------- |
| `id`   | scene component 唯一 id，后续通过 `map.removeSceneComponent(id)` 移除 |
| `type` | 固定为 `'s3m'`                                                        |
| `url`  | S3M 服务的配置地址（`.scp` 文件或 REST 配置地址）                     |

## 公共生命周期

```javascript
const tileset = map.addSceneComponent({ id, type: "s3m", url });
tileset.readyPromise.then(() => {
	// 1. 调相机定位
	// 2. 读取 tileset 元数据
});
map.removeSceneComponent("cbd-s3m");
```

## 光照与时间联动

S3M 与全局光照系统共享同一份 sun 状态，因此可以用：

-   `map.enableLight()` / `map.disableLight()`
-   `map.setSunLightTimestamp(date.getTime())`
-   `map.setSunLightTime({ year, month, day, hours, minutes, seconds })`

推荐做法：

1. `load` 之后立即 `map.enableLight()`
2. 设置一个默认 `sunTime`（比如中午 12:00）
3. 用户切换时间时调用 `setSunLightTimestamp` 触发重渲染

## Demo-backed Patterns

### 模式 1：本地 S3M 数据（CBD 示例）

对应 demo：`demo/html/S3M.html`

```javascript
const tileset = map.addSceneComponent({
	id: "cbd",
	type: "s3m",
	url: "../data/s3m/CBD/cbd.scp"
});
```

特点：

-   用空 style 起图
-   添加一张 raster 影像底图作为参考
-   加载完成后再定位相机

### 模式 2：远端 S3M REST 服务

对应 demo：`demo/html/S3M_Pipeline.html`

```javascript
const tileset = map.addSceneComponent({
	id: "serve",
	type: "s3m",
	url: "https://.../3D-xxxx/rest/realspace/datas/PI_UV/config"
});
```

特点：

-   URL 指向 `realspace` 服务的 `datas/.../config` 端点
-   可同时挂多个 S3M tileset
-   每个 tileset 独立 `readyPromise`

### 模式 3：锦江示例（数据 + 城市）

对应 demo：`demo/html/S3M_JinJiang.html`

用于演示城市级倾斜摄影 + 地形 / 影像底图组合。

## Strict Constraints

### 1. 必须用 `addSceneComponent`

不要尝试用 `addSource({ type: 's3m' })` + `addLayer`。S3M 不是 style source/layer 模型。

### 2. `id` 唯一

重复 `id` 会替换现有 scene component，调用前确认 `map.getSceneComponent(id)` 行为。

### 3. 移除时使用 `removeSceneComponent(id)`

不要手动 `tileset.destroy()` 之类的私有方法。

### 4. S3M 依赖 crn 解析资源

部分 S3M 数据使用 crn 压缩纹理，需要在页面里配置 `window.minemapCDN`，让引擎能够拉取 wasm 解码器。

```javascript
window.minemapCDN = "https://your.cdn/minemap-CDN";
```

如果 CDN 没配，crn 纹理会失败并出现"紫黑方块"。

### 5. 默认双面绘制

4.25 默认开启 S3M 双面绘制，便于进入建筑内观察。如需关闭，参考对应 `setDoubleSided` / material 覆盖方式（按发布版本决定，不在主推路径）。

## Failure Cases

-   紫黑方块、纹理缺失 → 检查 `window.minemapCDN` 是否配置
-   数据加载后相机仍在地表 → 监听 `readyPromise` 后再 `setView`
-   多个 S3M 共存时定位错乱 → 给每个 S3M 单独的 `id`，并各自等待 `readyPromise`
-   S3M 看起来全黑 → 检查 `enableLight` 是否开启，并设置合理 `sunTime`

## See Also

-   `minemap-scene-components`
-   `minemap-3d-tiles-runtime-control`
-   `minemap-lighting-and-shadows`
-   `minemap-business-oblique-bim`
