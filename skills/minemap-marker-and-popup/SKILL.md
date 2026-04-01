---
name: minemap-marker-and-popup
description: MineMap 的 Marker 与 Popup 规范。涵盖 DOM 覆盖物、三维高度、distanceDisplayCondition、拖拽、内容 API、地形场景与常见失败场景。
---

# MineMap Marker And Popup

## Quick Start

```javascript
const marker = new minemap.Marker({
	color: "#ff4d4f",
	draggable: true,
	offset: [-13.5, -35]
})
	.setLngLat([116.39, 39.9])
	.setAltitude(120)
	.addTo(map);

const popup = new minemap.Popup({
	closeOnClick: false,
	closeButton: false,
	offset: [0, -40]
})
	.setText("我是带三维高度的 DOM 标注")
	.setLngLat([116.39, 39.9])
	.setAltitude(120);

marker.setPopup(popup).togglePopup();
```

## Architecture Positioning

### 1) `Marker` 和 `Popup` 都是 DOM 覆盖物

它们的定位很明确：

-   挂在地图容器 DOM 上
-   不是 layer
-   不是 primitive
-   不是 `SceneModel` / `SceneTileset`

因此它们的优点和边界也很明确：

-   优点：适合业务 UI、编辑把手、说明卡片、少量交互点位
-   边界：它们不参加真正的 3D 深度测试

对 `Popup` 来说，这个约束尤其重要：

-   支持三维高度语义
-   不支持深度测试
-   因为它本质仍然是 DOM，而地图主体是 canvas

这条结论对 `Marker` 也成立：

-   可以做“看起来有高度”的 DOM 锚点
-   但不要把它当成真正参与遮挡排序的三维对象

### 2) 适合什么，不适合什么

适合场景：

-   少量业务点位
-   人工标注、编辑锚点
-   点击反馈卡片
-   调试把手
-   需要直接塞 HTML / DOM 的说明 UI

不适合场景：

-   成千上万的点
-   需要真实 3D 遮挡顺序
-   需要像图层一样做批量样式驱动

这类场景应改用：

-   `circle` / `symbol` layer
-   `PointPrimitiveCollection`
-   `SceneModel` / `SceneTileset`

## Quick Decision Rules

### 用 `Marker` 的时候

-   需要一个少量、可交互、可拖拽的点位 DOM
-   需要绑定 popup
-   需要自定义点位元素、按钮、状态角标

### 用 `Popup` 的时候

-   需要点击或 hover 的说明框
-   需要放文本、HTML 或复杂 DOM 内容
-   需要一个近景信息提示，而不是图层对象

### 不要继续往 DOM 覆盖物方向走的时候

-   点太多
-   需要稳定的高性能批量渲染
-   需要严格贴地、遮挡、材质或分类渲染

## Marker Core Concepts

### 构造方式

`Marker` 支持两种创建方式：

1. 传自定义 `HTMLElement` + 配置
2. 只传配置对象，让引擎创建默认 marker

### 常用构造参数

-   `element`：自定义 DOM
-   `scale`：仅默认 marker 生效，默认 `1`
-   `anchor`
-   `offset`
-   `color`：默认 marker 颜色，默认 `#3FB1CE`
-   `draggable`：默认 `false`
-   `clickTolerance`
-   `rotation`：默认 `0`
-   `rotationAlignment`：默认 `auto`
-   `pitchAlignment`：默认 `auto`，内部会跟随 `rotationAlignment`
-   `distanceDisplayCondition`

### `anchor` 和 `offset`

`anchor` 决定的是：

-   `setLngLat()` 指定的经纬度落在 marker 元素的哪个锚点上

可选值：

-   `center`
-   `top`
-   `bottom`
-   `left`
-   `right`
-   `top-left`
-   `top-right`
-   `bottom-left`
-   `bottom-right`

`offset` 是像素偏移，计算原点是元素中心。

默认 marker 特别要注意：

-   如果想让默认图钉的尖头落在真实坐标点上，常用偏移是 `[-13.5, -35]`

### 位置与高度

基础位置：

-   `setLngLat([lng, lat])`

三维高度：

-   `setAltitude(heightToSurface)`

高度语义是：

-   相对地表的高度
-   让 marker 在视觉上“抬起来”

常见写法：

```javascript
marker.setLngLat([116.39, 39.9]).setAltitude(80);
```

### 地形上的三维高度与拖拽

这是 MineMap 里 `Marker` 很实用的一点。

源码里的拖拽逻辑会区分 terrain：

-   非 terrain：按带高度椭球面反算位置
-   terrain 打开时：用 GPU pick 取地表位置
-   如果 pick 到 terrain，高度会在拖拽中自动刷新

也就是说：

-   在 terrain 场景下拖 marker，不只是平面挪位置
-   高度也可能被同步改掉

这很适合：

-   山地选点
-   地形表面交互锚点
-   编辑态落点修正

### Popup 绑定

`Marker` 最常见的工作流：

-   `setPopup(popup)`
-   `togglePopup()`

关键细节：

-   marker 绑定 popup 后，点击 marker 会切换 popup
-   如果 popup 没显式给 `offset`，源码会为默认 marker 自动推导一套合适偏移
-   自定义 marker 没给 popup 偏移时，会复用 marker 的 `offset`

### 拖拽相关方法

-   `setDraggable(true | false)`
-   `enableDragging()`
-   `disableDragging()`
-   `isDraggable()`
-   `getDraggable()`

使用建议：

-   业务编辑态再开拖拽
-   展示态尽量关掉

### 旋转与地图倾斜

-   `setRotation(rotation)`
-   `getRotation()`
-   `setRotationAlignment(alignment)`
-   `getRotationAlignment()`
-   `setPitchAlignment(alignment)`
-   `getPitchAlignment()`

对齐语义：

-   `rotationAlignment = 'map'`：更像固定在地图平面
-   `rotationAlignment = 'viewport'`：更像始终朝向观察者
-   `pitchAlignment = 'map'`：贴地图平面
-   `pitchAlignment = 'viewport'`：立在视口前

### 距离显隐

`Marker` 支持 `distanceDisplayCondition`。

作用：

-   根据相机与 marker 的距离决定显隐

适合：

-   中近景信息点
-   防止远景时 DOM 点位满屏

### 常用实例方法

-   `addTo(map)`
-   `remove()`
-   `getLngLat()`
-   `setLngLat()`
-   `getAltitude()`
-   `setAltitude()`
-   `getElement()`
-   `setPopup()`
-   `getPopup()`
-   `togglePopup()`
-   `getMap()`

## Popup Core Concepts

### 构造参数

-   `closeButton`：默认 `true`
-   `closeOnClick`：默认 `true`
-   `closeOnMove`：默认 `false`
-   `focusAfterOpen`：默认 `true`
-   `anchor`
-   `offset`
-   `className`
-   `maxWidth`：默认 `240px`
-   `distanceDisplayCondition`

### 基础内容 API

-   `setText(text)`：安全文本，不插原始 HTML
-   `setHTML(html)`：直接插 HTML，只能用于可信内容
-   `setDOMContent(node)`：直接放 DOM 节点
-   `setMaxWidth(width)`

推荐规则：

-   普通业务文本优先 `setText()`
-   可信模板字符串再用 `setHTML()`
-   复杂表单、按钮面板、交互卡片优先 `setDOMContent()`

### 位置与高度

-   `setLngLat([lng, lat])`
-   `setAltitude(heightToSurface)`
-   `getAltitude()`

三维高度写法：

```javascript
popup.setLngLat([116.39, 39.9]).setAltitude(120);
```

重要边界：

-   `Popup` 支持高度语义
-   但不支持深度测试

所以它更适合：

-   悬浮信息牌
-   高处设备说明框
-   模型上方的 UI 标识

而不适合：

-   需要被模型真实遮挡裁切的三维面板

### 指针跟随模式

`trackPointer()` 会把 popup 绑定到鼠标指针位置，而不是固定经纬度。

它会替代 `setLngLat()` 的常规定位行为。

推荐搭配：

-   `closeOnClick: false`
-   `closeButton: false`

典型场景：

-   hover 提示框
-   鼠标跟随说明卡片

### offset 策略

`offset` 支持三种写法：

1. 数字
2. `PointLike`
3. 按 anchor 分别配置的对象

对象写法适合复杂气泡：

-   不同朝向使用不同偏移
-   避免 tip 和 marker 重叠

### 类名控制

-   `addClassName(className)`
-   `removeClassName(className)`
-   `toggleClassName(className)`

这是做业务皮肤最稳的入口，不要直接依赖内部 DOM 结构做过深选择器耦合。

### 生命周期方法

-   `addTo(map)`
-   `remove()`
-   `isOpen()`

### 距离显隐

`Popup` 也支持 `distanceDisplayCondition`。

这在三维场景里很有用：

-   近景打开 popup
-   远景自动隐藏

## Common Patterns

### 模式 1：普通点位说明

```javascript
const popup = new minemap.Popup({ closeOnClick: false }).setText("站点 A");

new minemap.Marker({ color: "#ff0000" })
	.setLngLat([116.39, 39.9])
	.setPopup(popup)
	.addTo(map);
```

### 模式 2：带三维高度的业务点

```javascript
const marker = new minemap.Marker({ element: el, offset: [-12, -38] })
	.setLngLat([116.39, 39.9])
	.setAltitude(60)
	.addTo(map);

const popup = new minemap.Popup({ offset: [0, -50], closeOnClick: false })
	.setHTML("<div>高空设备</div>")
	.setLngLat([116.39, 39.9])
	.setAltitude(60);

marker.setPopup(popup);
```

关键点：

-   `Marker` 和 `Popup` 高度最好同时设
-   否则视觉上会错层

### 模式 3：地形上拖拽选点

```javascript
const marker = new minemap.Marker({ draggable: true })
	.setLngLat([116.39, 39.9])
	.addTo(map);
```

terrain 开启时，拖拽会走地表 pick，高度可能自动更新。

### 模式 4：hover 跟随提示

```javascript
const hoverPopup = new minemap.Popup({
	closeButton: false,
	closeOnClick: false
}).trackPointer();

map.on("mousemove", "poi-layer", (e) => {
	hoverPopup
		.setText(e.features[0].properties.name)
		.addTo(map);
});
```

这类场景不要手动每帧 `setLngLat()` 去模拟跟随。

## Failure Cases

### 失败场景 1：拿 `Marker` 充当大量点图层

大量 DOM 会明显拖慢交互和布局。大批量点位应改用：

-   `circle`
-   `symbol`
-   `PointPrimitiveCollection`

### 失败场景 2：只给 marker 设高度，不给 popup 设高度

结果是 marker 在高处，popup 还贴在地面参考位置附近，看起来像断开了。

### 失败场景 3：把 popup 当真实三维面板

`Popup` 不支持深度测试，不能指望它像 mesh 一样被模型自然裁掉。

### 失败场景 4：对不可信内容使用 `setHTML()`

源码不会替你做 HTML 清洗，`setHTML()` 只能喂可信内容。

### 失败场景 5：把 hover popup 写成固定 `setLngLat()`

如果需要鼠标跟随，应优先用 `trackPointer()`，而不是自己不断 patch 经纬度。

### 失败场景 6：展示态长期保留可拖拽 marker

这会增加误操作和多余的事件处理。拖拽能力应只在编辑态开放。

## Performance Tips

-   DOM 覆盖物数量控制在少量交互对象范围内
-   远距离信息点使用 `distanceDisplayCondition`
-   展示态关闭拖拽
-   大量说明内容优先复用 popup 实例，而不是每个点永久常开
-   复杂内容优先延迟创建 DOM，而不是初始化时一次性全部挂上

## Official Docs And Example Lookup

### 官方入口

`Marker` / `Popup` 相关公开资料建议一起查：

-   官网首页：`https://minedata.cn/`
-   3D Ultra 开发指南：`https://minedata.cn/nce-support/guide-3D-Ultra`
-   3D Ultra API 参考：`https://minedata.cn/nce-support/api-3D-Ultra`
-   3D Ultra 示例中心：`https://minedata.cn/nce-support/demoCenter-3D-Ultra`
-   平台登录入口：`https://map.minedata.cn/main-user/login`

当前公开页面能确认：

-   有登录入口
-   有试用入口
-   有帮助中心与示例中心

当前不能确认：

-   面向开发者的公开自助注册流程

所以在项目 onboarding 里，应该写成“账号登录 / 试用 / 开通申请”，不要写成“官网直接注册”。

### `Marker` / `Popup` 资料怎么查

推荐顺序：

1. 先看示例中心里的“覆盖物 / 点标记 / 信息窗体”分类
2. 再看 API 参考，确认类名、构造参数、实例方法
3. 再回本地 `source/api/Marker.js`、`source/api/Popup.js` 核对真实行为
4. 最后结合本地 `demo/` 搜更接近当前版本的案例

不要只看官网的原因：

-   官网 API 参考版本可能落后于当前仓库版本
-   三维高度、terrain 拖拽、popup offset 自动推导等细节，源码更完整
-   `Popup` 不支持深度测试这类边界，应以源码注释和实现为准

### 示例中心里优先查哪些栏目

最相关的官方栏目通常是：

-   覆盖物
-   点标记
-   信息窗体
-   地图事件

如果官网示例只展示基本用法，没有覆盖三维高度、距离显隐或 terrain 联动：

-   直接以本 skill 的源码结论为主
-   再看 `minemap-official-resources-and-onboarding`
-   最终以本地 `source/` + `demo/` 收敛

## See Also

-   `minemap-official-resources-and-onboarding`
-   `minemap-fundamentals`
-   `minemap-widget-and-controls`
-   `minemap-events-and-picking`
-   `minemap-scene-components`
-   `minemap-terrain-and-analysis`
