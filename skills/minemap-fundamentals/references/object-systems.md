# minemap-fundamentals 参数与模式参考

### 7) 核心数据结构：图层、模型、geometry 到底分别是什么

MineMap 里最容易混淆的，不是 API 名字，而是对象层级。

可以把常见结构分成 4 层：

#### A. style / source / layer：二维地图数据层

这是地图底图和常规专题图层的主数据结构。

```javascript
const style = {
	version: 8,
	glyphs: "minemap://fonts/{fontstack}/{range}",
	sprite: "minemap://sprite/sprite",
	sources: {
		poi: {
			type: "geojson",
			data: { type: "FeatureCollection", features: [] }
		}
	},
	layers: [
		{
			id: "poi-circle",
			type: "circle",
			source: "poi",
			paint: { "circle-radius": 6, "circle-color": "#2d8cf0" }
		}
	]
};
```

这里的职责是：

-   `style`：整张地图的样式树
-   `source`：数据来源
-   `layer`：把某个 source 画成具体视觉结果

适合：

-   点线面专题图
-   栅格底图
-   矢量瓦片
-   普通业务可视化

#### B. scene component：模型 / 3D Tiles / 场景对象层

这是 MineMap 4.x 推荐的三维主入口。

```javascript
map.addSceneComponent({
	id: "building",
	type: "3d-model",
	data: modelUrl,
	position: [116.39, 39.9, 10],
	rotation: [0, 0, 0],
	scale: [1, 1, 1]
});

map.addSceneComponent({
	id: "cityTiles",
	type: "3d-tiles",
	urls: [{ url: tilesetUrl, name: "city" }]
});
```

这里常见对象有：

-   `3d-model` → `SceneModel`
-   `3d-tiles` → `SceneTileset`
-   `scene-object` → `Earth` / `Skybox` / `Panorama` / `AirLine` 等

适合：

-   glTF / glb 模型
-   BIM
-   倾斜摄影
-   3D Tiles
-   场景级三维对象

#### C. primitive：低层三维绘制对象

`Primitive` 是比 scene component 更底层的一层。

它通常由：

-   `geometry`
-   `material`
-   `modelMatrix`

共同构成。

```javascript
const geometry = new minemap.Geometries.BoxGeometry({
	width: 100,
	height: 60,
	depth: 40
});

const primitive = new minemap.Primitive({
	geometry,
	material: minemap.StandardMaterial.fromType("Color", {
		color: "green",
		opacity: 0.8
	}),
	modelMatrix: matrix
});

map.addPrimitive(primitive);
```

适合：

-   分析辅助体
-   自定义三维标绘
-   低层材质实验
-   不想走完整 model / tileset 管线时的自定义对象

#### D. geometry：几何结构本体

`geometry` 只负责“形状”，不负责颜色、贴图、光照和挂载。

常见有：

-   `BoxGeometry`
-   `PlaneGeometry`
-   `PolygonGeometry`
-   `PolylineGeometry`
-   `CircleGeometry`
-   `TubeGeometry`

要点：

-   `geometry` 不是可直接上屏的最终对象
-   通常要和 `material` 一起放进 `Primitive`
-   如果你的需求是“加载模型文件”，就不该从 geometry 下手，而该走 scene component

可用一个简单判断法：

-   业务专题地图 → layer
-   外部三维资产（glTF/3DTiles）→ scene component
-   手工三维分析体 / 调试体 → primitive + geometry + material

### 8) DOM 覆盖物：`Marker` 与 `Popup`

`Marker` 和 `Popup` 现在单独拆到了独立 skill。

在 fundamentals 里只记住 4 条：

-   它们都是 DOM 覆盖物，不是 layer / primitive / scene component
-   两者都支持 `distanceDisplayCondition`
-   两者都支持通过 `setAltitude()` 表达相对地表高度
-   `Popup` 支持高度，但**不支持深度测试**

完整参数、三维高度联动、地形拖拽和 `Marker + Popup` 绑定细节，统一看 `minemap-marker-and-popup`。
