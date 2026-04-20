---
name: minemap-threejs-integration
description: MineMap 与 threejs 融合规范。覆盖 ThreeLayer、自定义 three 场景接入、OBJ/FBX/IFC/VTK 等多种数据加载，以及 GeoJSON 驱动的道路/围栏/雷达特效。
---

# MineMap Threejs Integration

## Quick Start

```javascript
import * as THREE from "three";
import { ThreeLayer } from "../js/minemap-three-plugin/index.js";

map.on("load", () => {
	map.addSource("imagery", {
		type: "raster",
		tiles: ["https://example.com/satellite/{z}/{r}/{c}/{z}_{x}_{y}.jpg"],
		tileSize: 512
	});

	map.addLayer({
		id: "imagery",
		type: "raster",
		source: "imagery",
		"depth-test": true
	});

	const threeLayer = new ThreeLayer({
		id: "three-scene",
		refCenter: [120.301663, 28.576329]
	});

	map.addLayer(threeLayer);

	const mesh = new THREE.Mesh(
		new THREE.BoxGeometry(20, 20, 20),
		new THREE.MeshStandardMaterial({ color: 0x00aaff })
	);
	mesh.position.set(0, 0, 10);
	threeLayer.addObject(mesh);
});
```

## Architecture Positioning

### 1) 这不是 MineMap 核心三维对象体系，而是一条“custom layer + threejs”的融合路径

threejs 插件当前来自：

-   `demo/js/minemap-three-plugin/index.js`
-   `demo/js/minemap-three-plugin/index.d.ts`

它的定位要和 `SceneModel` / `SceneTileset` 分清：

-   `SceneModel` / `SceneTileset` 属于 MineMap 原生三维对象体系
-   `ThreeLayer` 属于自定义渲染层，把 threejs 场景挂到 MineMap 地图渲染管线里

因此推荐理解是：

-   需要原生 glTF / 3D Tiles / tileset / terrain 联动的标准三维对象，优先 `addSceneComponent()`
-   需要程序化特效、three 原生 loader、非 MineMap 原生格式或高度自定义 mesh 时，用 `ThreeLayer`

### 2) ThreeLayer 的真实渲染形态

从 `index.js` 和 `index.d.ts` 可确认：

-   `ThreeLayer.type = 'custom'`
-   `ThreeLayer.renderingMode = '3d'`
-   要通过 `map.addLayer(threeLayer)` 挂到地图里

也就是说它本质上是：

-   一个 MineMap custom layer
-   内部持有自己的 three `Scene`、`PerspectiveCamera`、`WebGLRenderer`
-   复用 MineMap 的 canvas 和 WebGL 上下文

### 3) 当前实现要求 WebGL2

源码里 `onAdd(map, gl)` 会直接检查：

-   `gl instanceof WebGL2RenderingContext`

如果不是 WebGL2：

-   会直接 `console.warn`
-   当前 threejs 融合层不会正常进入工作状态

所以文档里必须明确写：

-   这条路径当前按 shipped 代码看是 WebGL2-only

### 4) 坐标不是直接拿经纬度塞 three，而是以 `refCenter` 为基准做转换

`ThreeLayer` 的最关键参数是：

-   `refCenter`

作用：

-   threejs 场景以内插参考中心为原点
-   MineMap 经纬度 / 高程通过 world matrix 转成 three scene position

对外可用方法：

-   `getRefCenter()`
-   `setRefCenter(center)`
-   `toScenePosition(position, altitude?)`

这决定了 threejs 融合的基本工作流：

-   地理坐标交给 MineMap
-   three mesh 坐标放在 `refCenter` 的局部场景系里

## Public Surface Confirmed By Types

### `ThreeLayer` 已确认入口

-   `new ThreeLayer(options)`
-   `addObject(object)`
-   `removeObject(object)`
-   `getScene()`
-   `getSceneRoot()`
-   `getCamera()`
-   `getRenderer()`
-   `getWebGLRenderer()`
-   `getMap()`
-   `findObjectByName(name, root?)`
-   `toScenePosition(...)`

### `ThreeLayer` 参数

从 `index.d.ts` 当前可确认：

-   `id`
-   `slot?`
-   `refCenter?`
-   `envTexture?`
-   `envIntensity?`
-   `createLight?`

当前 shipped demo 里最稳定、最明确使用的是：

-   `id`
-   `refCenter`

`createLight` 在实现里也有明确使用；默认会创建基础光。

对 `envTexture` / `envIntensity`：

-   类型里已出现
-   但当前示例对它们的展示不足
-   skill 不把它们当 first-choice demo-backed 主能力

## Demo-backed Data Support

最近与当前仓库已确认的 threejs 融合 demo 包括：

-   `demo/html/Threejs-Box.html`
-   `demo/html/Threejs-OBJ.html`
-   `demo/html/Threejs-FBX.html`
-   `demo/html/Threejs-FBX-New.html`
-   `demo/html/Threejs-IFC.html`
-   `demo/html/Threejs-VTK.html`
-   `demo/html/Threejs-RoadEffect.html`

这说明 threejs 融合路径当前至少覆盖 3 类对象来源。

### A. three 原生 Object3D / Mesh

`Threejs-Box.html` 证明：

-   直接 new `THREE.Mesh(...)`
-   设置局部 `position`
-   `threeLayer.addObject(mesh)`

就可以进入 MineMap 场景。

这是最基础、最稳的融合路径。

### B. 通过 three loader 加载外部模型格式

当前 demo 已明确展示：

-   `OBJ + MTL`
-   `FBX`
-   `IFC`
-   `VTK`

这条能力非常关键，因为它说明 threejs 融合不是只会画程序化几何，而是真的能接更多外部格式。

#### OBJ + MTL

对应 demo：

-   `demo/html/Threejs-OBJ.html`

特点：

-   `MTLLoader` 先加载材质
-   `OBJLoader` 再加载模型
-   可做包围盒辅助显示和 GUI 变换调试

#### FBX

对应 demo：

-   `demo/html/Threejs-FBX.html`
-   `demo/html/Threejs-FBX-New.html`

特点：

-   `FBXLoader` 加载外部 FBX
-   运行时可调材质双面、阴影、缩放、旋转
-   新版 demo 还证明了可以通过 URL 动态追加 FBX 模型，而不只是固定本地路径

#### IFC

对应 demo：

-   `demo/html/Threejs-IFC.html`

特点：

-   使用 `web-ifc` + `web-ifc-three`
-   `IFCLoader` 加载 IFC
-   可关闭 `IFCSPACE` 等可选类别

这条路径说明：

-   threejs 融合对 BIM 交换格式也能形成补充入口

#### VTK

对应 demo：

-   `demo/html/Threejs-VTK.html`

特点：

-   `VTKLoader` 加载科学可视化 / 医工类常见几何
-   geometry 读入后再配 three material

### C. GeoJSON 驱动的程序化特效对象

对应 demo：

-   `demo/html/Threejs-RoadEffect.html`

这个 demo 很值得单独强调，因为它不是“再加载一个外部模型”，而是：

-   读取 GeoJSON / JSON 数据
-   生成围栏、流动道路 tube、雷达圈等程序化对象
-   让 threejs 成为 MineMap 的视觉增强层

它证明的不是“支持某一种文件格式”，而是：

-   支持把地理数据加工成 three mesh 表达

## Built-in Mesh Factories

从 `index.d.ts` 当前能确认，插件公开了 3 个程序化 mesh 工厂：

-   `createFenceMesh(...)`
-   `createTubeMesh(...)`
-   `createCircleMesh(...)`

### `createFenceMesh`

适用：

-   围栏
-   波浪墙
-   涟漪墙
-   淡入淡出边界墙

关键参数：

-   `layer`
-   `coordinates`
-   `material`
-   `color`
-   `opacity`
-   `num`
-   `speed`
-   `height`

### `createTubeMesh`

适用：

-   道路线特效
-   流动管线
-   发光轨迹线

关键参数：

-   `layer`
-   `coordinates: Position[][]`
-   `material`
-   `speed`
-   `tubeRadius?`
-   `tubularSegments?`
-   `radialSegments?`
-   `radius`

这里非常重要的一点是：

-   `coordinates` 支持多条线
-   这正是 `Threejs-RoadEffect.html` 能把多份 road GeoJSON 组织进一张 three layer 的原因

### `createCircleMesh`

适用：

-   雷达圈
-   扩散圈
-   ripple 点特效

已确认材质预设：

-   `ripple`
-   `spread`
-   `radar`

关键参数：

-   `layer`
-   `coordinates`
-   `material`
-   `color`
-   `opacity`
-   `radius`
-   `segments`
-   `num`
-   `speed`
-   `followWidth`

## Recommended Workflow

### 模式 1：底图仍由 MineMap 管，three 只做增强层

推荐组织：

1. `map.addSource(...)` / `map.addLayer(...)` 先铺影像或底图
2. 再 `map.addLayer(new ThreeLayer(...))`
3. 最后把 three object 往 `ThreeLayer` 里加

这条路径在现有 demo 里非常一致。

### 模式 2：地理坐标先进 MineMap，再转 scene 坐标

如果对象本身不是直接放在 `refCenter` 附近局部坐标里，优先用：

-   `threeLayer.toScenePosition(lnglat, altitude?)`

不要自己手写一套经纬度转 three world 的逻辑。

### 模式 3：three loader 负责格式，MineMap 负责地图环境

推荐职责分工：

-   three loader：OBJ / FBX / IFC / VTK 等外部格式
-   MineMap：底图、地理定位、地图交互、相机环境
-   `ThreeLayer`：桥接渲染和坐标转换

### 模式 4：GeoJSON 做输入，three mesh 做高表现力输出

如果需求是：

-   道路发光流动
-   围栏波纹
-   雷达扫描

优先思路不是把 GeoJSON 直接塞 style layer 里硬做，而是：

-   GeoJSON 作为输入
-   `createTubeMesh/createFenceMesh/createCircleMesh` 作为可视化输出

## Strict Constraints

### 1) 当前 threejs 融合路径不是 MineMap 原生 feature/layer 体系

这意味着你不能期待它天然拥有：

-   `feature-state`
-   原生 layer query
-   style spec 驱动
-   原生 source/layer 生命周期语义

### 2) 这条路径当前按 shipped 代码看依赖 WebGL2

不要默认把它当成 WebGL1 兼容能力写进公开文档。

### 3) threejs 融合不等于替代 `SceneModel` / `SceneTileset`

如果你加载的是：

-   glTF / glb
-   3D Tiles

除非你确实需要 three 专属能力，否则仍优先用 MineMap 原生三维对象体系。

### 4) 多格式支持来自 three loader 生态，不是 MineMap 原生统一 loader

这条边界要写清：

-   MineMap 提供的是融合层
-   OBJ / FBX / IFC / VTK 的格式解析实际依赖 three 及其扩展 loader

## Failure Cases

### 失败场景 1：把 `ThreeLayer` 当作普通 MineMap 图层来写

它是 custom layer，不是 style layer。

### 失败场景 2：直接把 three 场景坐标当经纬度用

没处理 `refCenter` 和坐标转换时，位置会完全错位。

### 失败场景 3：用 threejs 融合去替代所有标准图层

这样会丢掉很多 MineMap 原生数据组织、查询和交互优势。

### 失败场景 4：没开 `map.repaint = true`，却期待 three 动画持续更新

当前示例基本都显式开了：

-   `map.repaint = true`

对于流动材质、雷达圈、tube 动效，这通常是必要前提。

## Demo-backed References

-   `demo/html/Threejs-Box.html`
-   `demo/html/Threejs-OBJ.html`
-   `demo/html/Threejs-FBX.html`
-   `demo/html/Threejs-FBX-New.html`
-   `demo/html/Threejs-IFC.html`
-   `demo/html/Threejs-VTK.html`
-   `demo/html/Threejs-RoadEffect.html`
-   `demo/js/minemap-three-plugin/index.js`
-   `demo/js/minemap-three-plugin/index.d.ts`

## See Also

-   `minemap-fundamentals`
-   `minemap-style-and-data`
-   `minemap-2d-3d-overlay-and-classification`
-   `minemap-scene-components`
-   `minemap-primitives-and-materials`
-   `minemap-plugin-demo-editor`