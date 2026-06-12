---
name: minemap-s3m-tiles
description: "MineMap S3M/S3MB/SCP 三维瓦片加载与运行时控制。适用于接入 SuperMap S3M 数据、.scp/config 地址、OSGBCacheFile、S3M 选择高亮、LOD 范围、显隐控制、readyPromise 后相机跳转、CRN 压缩纹理 CDN/WASM 依赖，以及通过 map.addSceneComponent({ type: 's3m' }) 排查 S3M 加载问题。"
---

# MineMap S3M 瓦片

## 快速开始

```javascript
map.on("load", () => {
	const s3mLayer = map.addSceneComponent({
		id: "s3m-cbd",
		type: "s3m",
		url: "../data/s3m/CBD/cbd.scp"
	});

	s3mLayer.readyPromise.then(() => {
		map.transform.activeCamera.setView({
			destination: new minemap.Math.Vector3(-2181968.890329965, 4385313.17843029, 4072712.8241634783),
			orientation: {
				heading: 3.1756648661534443,
				pitch: -0.3715184468182904
			}
		});
	});
});
```

移除时继续走场景组件入口：

```javascript
map.removeSceneComponent("s3m-cbd");
```

## 架构位置

在业务代码里使用 `map.addSceneComponent({ type: "s3m" })`，不要直接 `new S3MTilesLayer(...)`。

当前运行路径是：

- `Map#addSceneComponent(...)`
- `Style#addSceneComponent(...)`
- `Style#addS3M(...)`
- `S3MTilesLayer`
- `S3MCollection`

`S3MCollection` 在渲染管线里和 `modelCollection`、`tilesetCollection`、`primitiveCollection`、`sceneObjectCollection` 同级更新，因此 S3M 更接近三维场景组件，不是普通 style layer。

## 支持的数据形态

S3M 主要接受两类配置：

- S3MB JSON 配置，常见为 `.scp` 文件或 SuperMap `.../config` 服务地址。
- 旧版 XML SCP，其中 OSG 文件入口会从 `.osgb` 归一到 `.s3m`。

加载器会用资源地址推导基础路径，再按相对路径解析瓦片和附属文件。SuperMap realspace 服务会把内部瓦片路径改写到 `/rest/realspace/.../data/path/...` 这一类地址下。

## Demo 已验证写法

### 本地 SCP

`demo/html/S3M.html` 使用：

```javascript
const tileset = map.addSceneComponent({
	id: "serve",
	type: "s3m",
	url: "../data/s3m/CBD/cbd.scp"
});
```

本地回归优先用这个 demo，因为数据位于 `demo/data/s3m/CBD`。

### 远程管线数据

`demo/html/S3M_Pipeline.html` 添加两个独立的 S3M 场景组件：

```javascript
map.addSceneComponent({
	id: "pipe",
	type: "s3m",
	url: "https://www.supermapol.com/realspace/services/3D-0725RVM-2/rest/realspace/datas/PI_UV/config"
});

map.addSceneComponent({
	id: "other",
	type: "s3m",
	url: "https://www.supermapol.com/realspace/services/3D-0725RVM-2/rest/realspace/datas/Other_UV/config"
});
```

多份数据要使用不同 `id`。移除某个 `id` 只会移除对应组件。

### 加载完成后相机跳转

依赖数据范围或根节点状态的相机跳转放到 `readyPromise` 后执行，不要用固定 `setTimeout` 猜加载时机。

## 运行时控制

`map.addSceneComponent({ type: "s3m" })` 返回的是 `S3MTilesLayer`。当前确认有用的控制包括：

- `ready` 与 `readyPromise`
- `show` / `visible`
- `rectangle`
- `totalMemoryUsageInBytes`
- `maximumMemoryUsage`
- `lodRangeScale` 或 `setLodRangeScale(value)`
- `selectedColor`
- `selectEnabled`
- `setSelection(ids)`
- `releaseSelection()`
- `setVisibleInViewport(index, visible)`
- `getVisibleInViewport(index)`

`_rootTiles`、`_requestTiles`、`_style3D`、`_objsOperationList` 这类下划线字段只当内部实现细节，不写进业务方案。

## 纹理与 CDN 要求

S3M 已包含 CRN 纹理解码路径。相关 demo 会设置：

```javascript
window.minemapCDN = "http://192.168.152.123/minemap-engine/minemap-CDN";
```

部署环境需要提供等价 CDN 路径，确保 CRN/WASM 依赖可被请求到。若几何已出现但纹理缺失、图层长时间不显示，先查 `window.minemapCDN`、CRN/WASM 网络请求和 CORS。

S3M 内的 KTX2 会识别为 `S3MPixelFormat.KTX2`。通用 KTX2 mip 链、WebGL1 降级和黑纹理排查看 `minemap-compressed-texture-runtime`。

## 渲染注意

近期源码让 S3M render entity 默认创建 `doubleSided: true` 材质。不要在业务层先加背面渲染补丁，除非某个具体数据集仍然证明需要定制处理。

S3M 更新受 `CRNTranscoder.wasmReady` 控制。首帧里 `S3MTilesLayer.update(...)` 可能只是初始化转码器并提前返回，瓦片渲染会在依赖就绪后继续。

## 常见错误

- 在业务代码里直接 `new S3MTilesLayer(...)`。
- S3M 数据含 CRN 纹理时忘记配置 `window.minemapCDN`。
- 把 `readyPromise` 理解成所有子瓦片都已驻留；它更偏根配置/根瓦片就绪。
- 多个 S3M 数据复用同一个 `id`。
- 用 `queryRenderedFeatures` 做 S3M 拾取；S3M 走三维场景组件路径，交互要按场景拾取模式处理。

## 源码依据

- `source/style/style.js`
- `source/style/S3MCollection.js`
- `source/renderer/RenderGroup.js`
- `source/third-party/s3m/S3MTiles/S3MTilesLayer.js`
- `source/third-party/s3m/S3MTiles/Core/CRNTranscoder.js`
- `source/third-party/s3m/S3MParser/S3ModelParser.js`

## Demo 依据

- `demo/html/S3M.html`
- `demo/html/S3M_Pipeline.html`
- `demo/html/S3M_JinJiang.html`
- `demo/data/s3m/CBD/cbd.scp`

## 相关技能

- `minemap-scene-components`
- `minemap-3d-tiles-runtime-control`
- `minemap-model-runtime-debug`
- `minemap-compressed-texture-runtime`
