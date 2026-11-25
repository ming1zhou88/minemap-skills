# minemap-marker-and-popup 参数与模式参考

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
