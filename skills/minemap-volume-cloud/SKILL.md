---
name: minemap-volume-cloud
description: MineMap 气象规则网格三维体渲染。适用于加载体数据 JSON 或内存网格、设置颜色映射和阈值、裁切或沿线剖切体数据。
metadata:
  minemap-version: 4.28.2
---

# MineMap VolumeCloud

风速 U/V 规则网格与粒子轨迹使用 [minemap-wind-field](../minemap-wind-field/SKILL.md)；本专题处理标量体数据，不替代风场 renderer。

## 入口

`VolumeCloud` 是基于规则网格数据的自定义图层，通过 `Map` 的体数据方法管理。它与将一组二维图片堆叠成体的 `VolumeShellSceneComponent` 是不同的数据路径。

```javascript
const volume = map.addVolumeCloud({
	id: "weather-volume",
	url: "https://example.com/weather-grid.json",
	threshold: 25,
	steps: 500,
	densityScale: 1
});

await volume.readyPromise;
```

地图也提供 `getVolumeCloud(id)` 和 `removeVolumeCloud(idOrVolume)`；移除时引擎会释放该对象的 GPU 资源。也可向 `addVolumeCloud()` 传入 `VolumeCloud` 实例，或传 `data` 直接使用内存数据。

## 输入数据

当前 `VolumeCloudData` 需要 `rows`、`cols`、`heights`、`values` 以及 `xmin/xmax`、`ymin/ymax`、`zmin/zmax` 范围。`values` 长度必须等于 `rows * cols * heights`，排列次序是 height、row、column；`-1` 是默认空值。通过 `url` 加载时，服务返回的数据应符合这套结构。

```javascript
const volume = map.addVolumeCloud({
	id: "weather-volume",
	data: {
		rows,
		cols,
		heights,
		values,
		xmin, xmax,
		ymin, ymax,
		zmin, zmax
	},
	threshold: 25,
	steps: 500
});
```

`url` 资源由该图层自行请求。若服务需要鉴权或自定义 fetch 选项，通过 `requestInit` 设置请求头等参数；不要假设地图的 `transformRequest` 会自动改写该请求。跨域服务还需允许浏览器访问。

## 显示与裁切

常用构造选项包括 `show`、`threshold`、`steps`、`opacity`、`densityScale`、`clip`、`colorSteps` / `colors` 或 `transferFunction`。`steps` 控制射线采样次数，体素数量和采样次数都会影响渲染成本，应从较低分辨率和适中步数开始。

需要沿地理线查看体数据时，可使用 `polylineSlice`：传入至少两个 `[longitude, latitude]` 坐标，并设置大于零的 `width`。`showHelper` 和 `helperColor` 用于控制辅助几何显示。

```javascript
const volume = map.addVolumeCloud({
	id: "weather-volume",
	url: "https://example.com/weather-grid.json",
	clip: { z: [-0.3, 0.4] },
	polylineSlice: {
		positions: [[116.2, 39.7], [116.6, 40.0]],
		width: 500,
		enabled: true,
		showHelper: true
	}
});
```

`clip` 的 x/y/z 值是体数据局部范围内的归一化区间，端点限制在 `[-0.5, 0.5]`。如业务需要交互式更新裁切或剖切参数，先读取当前版本 `VolumeCloud` 的公开方法，再按需更新；不要通过修改内部 shader 或 GPU 资源实现。

## 运行约束

- `VolumeCloud` 要求 WebGL 2；WebGL 1 环境会失败，不存在自动降级到 WebGL 1 的渲染路径。
- 三维纹理尺寸受设备 `MAX_3D_TEXTURE_SIZE` 限制。构造失败时先检查三轴尺寸和数据内存，再提高设备要求。
- `readyPromise` 完成后再依赖加载结果；处理拒绝状态并显示可理解的错误。
- 大体积数据优先在服务端裁剪、降采样或按需生成数据，避免把超大 `values` 数组直接送入浏览器。
- 气象网格体渲染不要写成 `addSource({ type: "volume" })`；这是自定义 layer，不是 style source。

## 依据

- `source/api/map.js`（`addVolumeCloud`、`getVolumeCloud`、`removeVolumeCloud`）
- `source/renderer/effects/volumeCloud/VolumeCloud.js`
- `demo/html/VolumeCloud.html`

## 相关技能

- `minemap-volume-rendering`（图片切片堆叠的 VolumeShell）
- `minemap-clouds-and-atmosphere`
- `minemap-performance-and-backend`
