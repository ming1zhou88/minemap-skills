---
name: minemap-geojson-clustering-and-feature-state
description: MineMap GeoJSON 聚合与 feature-state。用于大量点聚合、展开/分页查询、稳定要素 ID 和 hover 高亮，避免用 setData 重建数据表达选择状态。
metadata:
  minemap-version: 4.28.2
---

# GeoJSON Clustering and Feature State

## 聚合点

style 就绪后添加 Point 数据。聚合与非聚合层分开，聚合 ID 不能当成原始业务 ID：

```javascript
map.addSource("points", {
  type: "geojson", data: pointCollection,
  cluster: true, clusterRadius: 50, clusterMaxZoom: 15
});
map.addLayer({
  id: "clusters", type: "circle", source: "points",
  filter: ["has", "point_count"],
  paint: { "circle-radius": 18, "circle-color": "#22c7a9" }
});
map.addLayer({
  id: "single-points", type: "circle", source: "points",
  filter: ["!", ["has", "point_count"]],
  paint: { "circle-radius": 5, "circle-color": "#2878bd" }
});
function expandCluster(event) {
  const feature = event.features?.[0];
  if (!feature) return;
  const source = map.getSource("points");
  source.getClusterExpansionZoom(feature.properties.cluster_id, (error, zoom) => {
    if (error || map.getSource("points") !== source) return;
    map.easeTo({ center: feature.geometry.coordinates, zoom });
  });
}
map.on("click", "clusters", expandCluster);
// 卸载业务叠加时：map.off("click", "clusters", expandCluster)
```

getClusterChildren(clusterId, callback) 取直接子节点；getClusterLeaves(clusterId, limit, offset, callback) 分页取原始点。与 expansionZoom 都是错误优先回调 API，不能直接 await 返回值。数据更换或卸载后丢弃旧结果，实际业务需加 disposed/请求序号保护。

## 稳定 ID 和状态

GeoJSON Feature 使用稳定 id 或 promoteId。generateId 依赖数据顺序，setData 后不保证 ID 对应同一业务对象。MVT 状态还需 sourceLayer。

```javascript
map.setFeatureState({ source: "areas", id: areaId }, { hover: true });
map.setPaintProperty("area-fill", "fill-opacity", [
  "case", ["boolean", ["feature-state", "hover"], false], 0.8, 0.3
]);
map.removeFeatureState({ source: "areas", id: areaId }, "hover");
```

- feature-state 只用于 schema 允许的 paint 状态表达式，不用于 filter/layout。
- hover 切换先清旧 ID 再设新 ID；ID=0 有效，用 null/undefined 判断。
- 状态与 feature.properties 分开，不每次 mousemove 都 setData/setStyle。
- source 重建后重放业务选择状态；cluster_id 随缩放与数据变化，不能持久化。
- 自定义指标用 clusterProperties 并核对 map/reduce 表达式，point_count 只代表点数。

## 证据

- `source/data-source/source-cache/GeojsonSource.js`
- `source/api/map.js`
- `demo/html/PointClusterInfo.html`
- `demo/html/PointClusterByMultipleSource.html`
- `demo/html/GeoJSONWithHover.html`
- [minemap-events-and-picking](../minemap-events-and-picking/SKILL.md)、[minemap-style-and-data](../minemap-style-and-data/SKILL.md)。
