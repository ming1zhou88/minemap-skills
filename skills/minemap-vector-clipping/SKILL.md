---
name: minemap-vector-clipping
description: MineMap style 矢量图层的 Polygon / MultiPolygon 区域裁剪。适用于按地理边界显示或排除矢量图层内容；不用于裁剪栅格、3D Tiles 或独立场景对象。
metadata:
  minemap-version: 4.28.2
---

# MineMap Vector Clipping

## 适用范围

`Map` 的 `vectorClip` 构造选项及 `setVectorClip()` 系列方法会把 GeoJSON 多边形转换为屏幕遮罩，作用于引擎支持的 style 矢量图层。裁剪几何使用经纬度坐标，接受 `Polygon`、`MultiPolygon`、`Feature` 和 `FeatureCollection`；Polygon 内环可表示孔洞。

```javascript
const map = new minemap.Map({
	container: "map",
	style,
	vectorClip: {
		geometry: boundaryFeature,
		mode: "include",
		layers: ["roads", "buildings"]
	}
});
```

也可以在地图创建后更新：

```javascript
map.setVectorClip({
	geometry: boundaryFeature,
	mode: "exclude",
	excludeLayers: ["clip-outline"]
});

// 暂时停用但保留当前几何；或清除配置并释放遮罩资源
map.setVectorClipEnabled(false);
map.clearVectorClip();
```

## 选择图层模式

- `mode: "all"`：对支持的矢量图层统一裁剪，忽略图层 id 列表。
- `mode: "include"`：只裁剪 `layers` 中列出的 style layer id。
- `mode: "exclude"`：裁剪其他支持的矢量图层；通过 `excludeLayers`（兼容 `layers`）列出不裁剪的 style layer id。
- `enabled: false`：保留几何和模式，但暂时关闭裁剪。

这里填写的是 style 图层 id，不是 MVT 的 `source-layer` 名称。若要绘制裁剪边界自身，通常把边界轮廓图层放进排除列表，否则轮廓也会被遮罩。

## 更新与清理

- `setVectorClip(options)` 替换当前裁剪配置。
- `setVectorClipGeometry(geometry)` 只替换几何，并保留当前模式和启用状态。
- `setVectorClipEnabled(enabled)` 只切换启用状态。
- `getVectorClip()` 读取当前配置；未配置时返回 `null`。
- `clearVectorClip()` 清除配置并触发重绘。

以上方法返回当前 `Map` 实例，便于链式调用。变更几何时使用公开方法，不要直接修改 `painter.vectorClipManager` 等内部对象。

## 边界与排查

裁剪仅覆盖实现登记的 style 矢量图层类型，例如 `fill`、`line`、`circle`、`symbol`、`heatmap`、`extrusion`、`tracking`、`symtracking`、`dynamicLine`、`histogram` 和 `sprite`。栅格图层、`3d-model`、`3d-tiles` 与独立 scene component 不在此遮罩范围内。需要裁剪这些对象时，选择各自的数据处理或几何方案。

常见问题：

- 裁剪方向反了：确认 `mode`，并区分 `include` 与 `exclude`。
- 部分图层没有变化：确认其类型受支持，并传 style layer id。
- 边界线消失：把轮廓 layer 加入排除列表。
- 多边形为空或报错：确认输入是受支持的 GeoJSON 类型，坐标顺序为 `[longitude, latitude]`。

## 依据

- `source/api/map.js`（构造选项与 `setVectorClip*` 方法）
- `source/renderer/vector-clip/VectorClipManager.js`
- `demo/html/VectorPolygonClip.html`

## 相关技能

- `minemap-style-and-data`
- `minemap-layer-system`
- `minemap-events-and-picking`
