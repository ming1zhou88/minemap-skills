---
name: minemap-wmts-source
description: MineMap WMTS capabilities 接入。用于解析 XML、选择图层和矩阵集、配置 raster source，并排查 REST/KVP 模板与层级不匹配。
metadata:
  minemap-version: 4.28.2
---

# MineMap WMTS

## 公开流程

WMTSCapabilities.read → optionsFromCapabilities → raster source。不能写 type wmts，也不能把 capabilities URL 当作瓦片 URL。在 style 就绪后执行：

```javascript
const response = await fetch(capabilitiesUrl);
if (!response.ok) throw new Error(`WMTS capabilities: ${response.status}`);
const parsed = new minemap.WMTSCapabilities().read(await response.text());
const options = minemap.optionsFromCapabilities(parsed, {
  layer: layerIdentifier, matrixSet: matrixSetIdentifier, requestEncoding: "REST"
});
map.addSource("wmts-source", {
  type: "raster", scheme: "wmts", tiles: options.urls,
  capabilities: options, tileSize: 256
});
map.addLayer({ id: "wmts-layer", type: "raster", source: "wmts-source" });
```

## 选择和兼容边界

- layer 使用 Identifier；当前实现的 style 选择比较 Style.Title 后输出 Identifier，不套用其他 SDK 语义。
- 检查返回 options.matrixSet；匹配失败时当前实现可能退到第一个 link，不能把无异常当作匹配成功。
- 解析仅支持 EPSG:3857/4326，矩阵原点、轴顺序、分辨率和地图/source 投影仍须核对。
- TileID.wmtsUrl 以 z/x/y 替换 TileMatrix/TileCol/TileRow，不将 z 自动映射为任意矩阵 Identifier。带前缀或不连续的矩阵层级需要适配 URL/服务。
- KVP 解析分支可返回服务端点，但瓦片代码不会补齐 GetTile 参数。优先使用 REST 模板；KVP 必须由业务提供包含全部参数与占位符的完整模板。
- tileSize 按 TileMatrix 的 TileWidth/TileHeight 设置；256 是示例值。
- transformRequest 不会作用于应用自己的 fetch，capabilities 请求须单独配置鉴权。
- async 回调落地前检查页面仍挂载且 style 就绪；卸载时取消请求，先删 layer 再删 source。

## 证据

- `source/data-source/format/WMTSCapabilities.js`
- `source/core/math/geo/WMTS.js`
- `source/data-source/source-cache/RasterTileSource.js`
- `source/data-source/tile/TileID.js`
- `demo/html/WMTS.html`：旧 service WMTS 为兼容写法，新代码使用 scheme。
- [minemap-wms-source](../minemap-wms-source/SKILL.md)、[minemap-style-and-data](../minemap-style-and-data/SKILL.md)。
