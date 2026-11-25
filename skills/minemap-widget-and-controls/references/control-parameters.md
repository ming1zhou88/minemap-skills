# minemap-widget-and-controls 参数与模式参考

### `Navigation`

定位：缩放按钮 + 指南针。

构造参数：

-   `showCompass = true`
-   `showZoom = true`
-   `visualizePitch = false`

特点：

-   不实现 `getDefaultPosition()`，所以默认 `top-right`
-   `visualizePitch = true` 时，指南针会同时可视化 pitch，并调用 `resetNorthPitch()`
-   默认模式下点击指南针调用 `resetNorth()`

### `Scale`

定位：比例尺。

构造参数：

-   `maxWidth = 100`
-   `unit = 'metric' | 'imperial' | 'nautical'`

公开方法：

-   `setUnit(unit)`

默认位置：

-   `bottom-left`

### `Fullscreen`

定位：切换地图容器全屏。

特点：

-   无构造参数
-   不实现 `getDefaultPosition()`，默认 `top-right`
-   会自动检测浏览器全屏能力
-   不支持时会隐藏按钮并给出警告

### `GeolocateControl`

定位：浏览器地理定位按钮。

构造参数：

-   `positionOptions = { enableHighAccuracy: false, timeout: 6000 }`
-   `fitBoundsOptions = { maxZoom: 15 }`
-   `trackUserLocation = false`
-   `showUserLocation = true`

特点：

-   不实现 `getDefaultPosition()`，默认 `top-right`
-   `trackUserLocation = true` 时会进入持续跟踪模式
-   `showUserLocation = true` 时会在地图上放一个用户位置 marker

### `Attribution`

定位：显示归因、数据来源、自定义版权片段。

构造参数：

-   `compact`
-   `customAttribution`：字符串或字符串数组

公开方法：

-   `addAttribution(attribution)`
-   `removeAttribution(attribution)`

默认位置：

-   `bottom-right`

### `Logo`

定位：Logo 水印控件。

特点：

-   默认位置 `bottom-left`
-   Logo 为引擎内部控件，source/index.js 没有公开导出；不能 new minemap.Logo。遵循初始化品牌配置，不复制内部构造路径
-   更偏引擎级 / 品牌级控件，业务层通常不需要手写管理

### `Thumbnail`

定位：鹰眼缩略图控件。

构造参数：

-   `width = 100`
-   `height = 100`
-   `url`
-   `clickCallback`

默认位置：

-   `bottom-right`

特点：

-   主图移动时会同步更新缩略图视口
-   点击可反向驱动主图定位

### `ModelTransformationControl`

定位：模型位置 / 旋转 / 缩放编辑控件。

构造参数：

-   `onTransform`
-   `onTransformEnd`

默认位置：

-   `top-right`

特点：

-   通过地图点击选择模型
-   回调返回 `{ position, rotation, scale }`
-   更适合编辑器和调试场景

### `BatchAddInstanceControl`

定位：实例化模型批量布设工具。

默认位置：

-   `top-right`

能力边界：

-   支持点 / 线 / 面等方式生成实例位置
-   可结合 `addModel(options)` 直接批量挂实例化模型
-   常见参数包括 `data`、`modelFolder`、`rotations`、`scales`、`positions`、`instanceNames`、`id`

### `FPSControl`

定位：帧率观测。

特点：

-   更适合本地调试
-   在 `map.js` 内部也按控件方式接入
-   无需把它当正式产品 UI

## Complete Control Selection Guide

### 线上常驻优先级

优先常驻：

-   `Navigation`
-   `Scale`
-   `Fullscreen`
-   `Attribution`

按业务决定：

-   `GeolocateControl`
-   `Thumbnail`

默认不常驻：

-   `FPSControl`
-   `ModelTransformationControl`
-   `BatchAddInstanceControl`

### 控件停靠位建议

-   `top-right`：操作性控件，如 `Navigation`、`Fullscreen`、`GeolocateControl`
-   `bottom-left`：环境类控件，如 `Scale`、`Logo`
-   `bottom-right`：归因、鹰眼、说明性控件

如果同时放多个控件：

-   不要把重交互控件和大面积面板全塞在同一个角落
-   `bottom-right` 已经有 `Attribution` 时，再放 `Thumbnail` 要注意遮挡

## Common Patterns

### 常规地图工具栏

```javascript
map.on("load", () => {
	map.addControl(new minemap.Navigation(), "top-right");
	map.addControl(new minemap.Fullscreen(), "top-right");
	map.addControl(new minemap.Scale({ unit: "metric" }), "bottom-left");
});
```

适合常规业务地图页面。`ScaleControl.html` 和常规 demo 都按这个模式组织。

### `Scale` 的稳定用法

`Scale` 是标准 `IControl`，默认位置是 `bottom-left`，支持：

-   `maxWidth`
-   `unit: 'imperial' | 'metric' | 'nautical'`
-   运行时 `setUnit(...)`

推荐：

-   国内业务默认 `metric`
-   海事或航运专题再用 `nautical`
-   不要自己手写比例尺 DOM 去同步缩放

### `Thumbnail` 作为鹰眼/导航 widget

`Thumbnail` 是一个完整控件，不是 demo 私有脚本。

源码和 demo 已验证的关键参数：

-   `url`
-   `width`
-   `height`
-   `clickCallback`

典型用途：

-   显示全球或区域缩略图
-   点击缩略图后联动主地图 `easeTo(...)`
-   在主图移动时同步更新视口指示

```javascript
const thumbnail = new minemap.Thumbnail({
	url: thumbnailUrl,
	clickCallback: ({ lnglat }) => {
		map.easeTo({ center: lnglat, zoom: 0, pitch: 0, bearing: 0, duration: 2000 });
	}
});

map.addControl(thumbnail, "bottom-right");
```

### `ModelTransformationControl` 作为模型编辑 widget

这个控件适合运行时调整三维对象的位置、旋转、缩放。

源码与 demo 共同表明：

-   支持 `onTransform(result)`
-   支持 `onTransformEnd(result)`
-   默认停靠位是 `top-right`
-   应配合可拾取目标使用，至少要保证被编辑对象能被交互命中

```javascript
const transformControl = new minemap.ModelTransformationControl({
	onTransformEnd(result) {
		console.log(result.position, result.rotation, result.scale);
	}
});

map.on("load", () => {
	map.addSceneComponent({
		id: "building",
		type: "3d-model",
		data: modelUrl,
		position: [116.39, 39.9, 10],
		allowPick: true
	});

	map.addControl(transformControl, "top-left");
});
```

适用场景：

-   模型摆位校准
-   编辑器内姿态修正
-   调试阶段记录 position / rotation / scale 快照

### `GeolocateControl` 的真实使用边界

推荐用法：

-   移动端或外勤业务做“定位到我”
-   需要持续跟踪时开 `trackUserLocation`

不推荐：

-   在无 HTTPS / 无权限策略的环境里把它当必定可用功能
-   把它当高精度测量入口

### `Attribution` / `Logo` 的职责分工

建议区分：

-   `Attribution`：数据来源、归因、版权片段
-   `Logo`：品牌跳转与固定 Logo 呈现

不要把两者混成一个自定义 DOM 去重写，除非你明确接管全部合规和样式责任。

不建议直接当最终用户常驻控件。

### `BatchAddInstanceControl` 作为批量布点工具

这个控件本质上是“实例模型批量生成面板 + 地图交互工具”。

已验证能力：

-   默认位置 `top-right`
-   支持点/线/面三类生成方式
-   可通过 `getData()` 读取工具生成的实例结果
-   可与 `ModelTransformationControl` 组合成“先批量生成，再局部校准”的流程

推荐工作流：

1. `map.addControl(new minemap.BatchAddInstanceControl())`
2. 用工具在地图上绘制生成区域/路径/点位
3. 用 `getData()` 导出 position / rotation / scale
4. 需要微调时再挂 `ModelTransformationControl`

### `FPSControl` 只用于性能观察

`FPSControl` 适合本地调试：

-   看帧率波动
-   粗看不同后处理/阴影/模型量级下的性能变化

不建议：

-   作为正式产品 UI 一部分上线
-   用它代替真实 profiling
