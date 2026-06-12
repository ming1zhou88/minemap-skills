---
name: minemap-style-v8-differences
description: "MineMap style v8 与 Mapbox Style Specification v8 的差异检查。适用于编写或迁移 style/source/layer JSON，判断 MineMap 专有字段、Mapbox 专有字段、layer type/source type/paint/layout 差异，避免把 fill-extrusion、hillshade、sky、slot、model、raster-dem、fog、imports 等 Mapbox v8 字段误用于 MineMap。"
---

# MineMap Style V8 差异

## 使用原则

编写 MineMap style 时，以 `source/style/style-reference/style-spec/reference/v8.json` 为准。不要把 Mapbox 的 v8 文档当作 MineMap 的完整兼容清单。

两者同为 v8 风格结构，但 MineMap 已经独立演进：根字段、source type、layer type、三维场景能力、地形/后处理/视频投影入口都有差异。迁移样式时要逐字段核对，不要只看字段名相似。

一个背景信息只作理解来源：早期 0.49 版本曾沿用过 Mapbox 体系，后续受授权变化影响不再按该路线使用。这个历史不能作为当前字段兼容依据。

## 根字段差异

MineMap 专有或重点扩展的根字段：

- `IBL`
- `sceneComponent`
- `analysis`
- `postProcess`
- `videoProjection`
- `terrain`
- `camera`
- `lights`
- `projection`

Mapbox 当前 v8 中存在、但 MineMap 当前规范里不要直接照搬的根字段包括：

- `imports`
- `schema`
- `fragment`
- `featuresets`
- `models`
- `iconsets`
- `color-theme`
- `fog`
- `rain`
- `snow`
- `indoor`
- `light`
- `center`
- `zoom`
- `bearing`
- `pitch`

MineMap 的三维场景、光照、相机、后处理和天气效果不要按 Mapbox 的 `fog/rain/snow/light/models/imports` 思路配置；应走 MineMap 自己的 `lights`、`sceneComponent`、`postProcess`、`videoProjection`、`terrain` 等入口。

## Source Type 差异

MineMap 当前规范里的 source type：

- `vector`
- `raster`
- `geojson`
- `image`
- `3d-model`
- `3d-tiles`

Mapbox 当前 v8 里有但 MineMap 当前规范没有的 source type：

- `model`
- `raster_dem`
- `raster_array`
- `video`

不要把 Mapbox 的 `raster-dem` / `raster_dem` 写进 MineMap source。MineMap 地形按根字段 `terrain`、DEM 瓦片、地形分析相关入口组织；三维模型优先看 `sceneComponent` 或 MineMap 的 `3d-model` / `3d-tiles` 路径。

## Layer Type 差异

MineMap 当前规范里的 layer type：

- `background`
- `fill`
- `line`
- `symbol`
- `circle`
- `heatmap`
- `raster`
- `extrusion`
- `histogram`
- `dynamicLine`
- `sprite`
- `tracking`
- `symtracking`
- `3d-model`
- `3d-tiles`

Mapbox 当前 v8 中有但 MineMap 当前规范没有的 layer type：

- `fill-extrusion`
- `hillshade`
- `sky`
- `slot`
- `clip`
- `model`
- `building`
- `raster-particle`

最容易写错的是建筑拉伸：MineMap 使用 `type: "extrusion"`，paint 字段是 `extrusion-*`；不要写成 Mapbox 的 `type: "fill-extrusion"` 或 `fill-extrusion-*`。

```javascript
map.addLayer({
	id: "building-extrusion",
	type: "extrusion",
	source: "building",
	"source-layer": "building",
	paint: {
		"extrusion-color": "#7f8da3",
		"extrusion-height": ["get", "height"],
		"extrusion-base": 0,
		"extrusion-opacity": 0.9
	}
});
```

## 常见字段误用

不要从 Mapbox 当前 v8 直接搬这些字段到 MineMap：

- `fill-extrusion-*`
- `hillshade-*`
- `sky-*`
- `slot`
- `clip`
- `model`
- `raster-dem`
- `raster-array`
- `raster-particle`
- `imports`
- `schema`
- `fog`
- `rain`
- `snow`
- `light`
- `symbol-z-elevate`
- `symbol-elevation-reference`
- `line-emissive-strength`
- `fill-emissive-strength`

MineMap 专有或需要优先看本地规范的字段示例：

- `extrusion-*`
- `fill-water`
- `fill-water-texture`
- `fill-water-speed`
- `fill-water-wave-size`
- `fill-dynamic-image-url`
- `fill-dynamic-speed`
- `sunlight-color`
- `sunlight-direction`
- `sky-color`
- `line-border-opacity`
- `half-render`
- `line-normal-direction`
- `symbol-height-key`
- `symbol-height-offset`
- `symbol-pick-color`
- `background-pattern-fixed`
- `background-terrain-analysis`

## 迁移检查流程

1. 先读取 MineMap 本地 `v8.json`，确认 root/source/layer/type 是否存在。
2. 遇到 Mapbox 样式里的根字段，先判断 MineMap 是否有等价入口；没有就删掉或改成 MineMap 方案。
3. 将 `fill-extrusion` 改为 `extrusion`，并把 `fill-extrusion-*` 改成 MineMap 的 `extrusion-*`。
4. 将 Mapbox 的地形、天空、雾、雨雪、模型、导入片段等配置改成 MineMap 的 `terrain`、`sceneComponent`、`postProcess`、`lights` 等能力。
5. 修改后用 MineMap demo 或最小页面加载验证，重点看控制台 style 校验错误和图层是否实际渲染。

## 常见错误

- 看到 `$version: 8` 就默认 Mapbox v8 字段全兼容。
- 把 `fill-extrusion-height` 写进 MineMap，导致拉伸图层不生效。
- 把 Mapbox `sky` 或 `fog` 当成 MineMap 天空/天气入口。
- 把 Mapbox `model` source/layer 当成 MineMap 三维模型入口。
- 把 Mapbox `imports/schema/fragment` 作为 MineMap style 模块化方案。

## 对比依据

- MineMap：`source/style/style-reference/style-spec/reference/v8.json`
- Mapbox：`mapbox-gl-js/src/style-spec/reference/v8.json`
- 相关实现：`source/style/style.js`
- 图层系统：`source/style/style_layer/*`

## 相关技能

- `minemap-style-system`
- `minemap-style-and-data`
- `minemap-layer-system`
- `minemap-scene-components`
- `minemap-tile-url-and-data-security`
