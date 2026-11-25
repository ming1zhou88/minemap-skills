---
name: minemap-wms-source
description: MineMap WMS 接入与请求排查。用于 raster source 的 scheme wms、GetMap URL、投影和范围；不承诺自动解析 WMS capabilities。
metadata:
  minemap-version: 4.28.2
---

# MineMap WMS

## 当前入口

4.28.2 的 source 注册表没有 wms 类型。使用 type raster、scheme wms 和完整 GetMap URL；不会从 GetCapabilities 自动挑图层，也不会把顶层 layers/crs/transparent 等字段拼入请求。

## 最小接入

style 就绪后添加，下面明确使用 EPSG:3857 服务，地址和图层需替换：

```javascript
map.addSource("wms-source", {
  type: "raster", scheme: "wms", projection: "MERCATOR",
  tiles: ["https://your-domain.example/wms?SERVICE=WMS&VERSION=1.1.1&REQUEST=GetMap&LAYERS=city&STYLES=&FORMAT=image/png&TRANSPARENT=TRUE&SRS=EPSG:3857&BBOX={bbox}&WIDTH=256&HEIGHT=256"],
  tileSize: 256, bounds: [112, 28, 114, 30]
});
map.addLayer({ id: "wms-layer", type: "raster", source: "wms-source" });
```

## 边界与排查

- TileID.wmsUrl 将 {bbox} 替换为 Web Mercator 米制范围，不能仅把 URL 的 SRS 改成 EPSG:4326；不会自动转换 BBOX 或处理 WMS 1.3.0 的轴顺序。
- bounds 为经纬度显示范围，BBOX 为逐瓦片请求范围，单位不同。
- tileSize 与 URL WIDTH/HEIGHT 一致，业务端显式设置；服务参数放在 URL 中。
- 兼容字段 service WMS 会覆盖 scheme，新代码只写 scheme，避免冲突。
- 检查最终 URL、XML 错误响应、CORS 和鉴权，不将服务 XML 错误当作有效 PNG。
- 栅格不提供矢量属性查询；GetFeatureInfo 由应用按服务协议单独请求。
- image source 的 wms true 是另一条按视口取图路径，源码有未完善说明，不能套用 raster 瓦片参数。
- 移除先 removeLayer，再 removeSource。

## 证据

- source/data-source/source-cache/source.js
- source/data-source/source-cache/RasterTileSource.js
- source/data-source/tile/TileID.js
- source/style/style-reference/style-spec/reference/v8.json
- 当前 checkout 无独立 WMS demo，示例是源码核对结果，未对外部服务联调。
- [minemap-wmts-source](../minemap-wmts-source/SKILL.md)、[minemap-tile-url-and-data-security](../minemap-tile-url-and-data-security/SKILL.md)。
