---
name: minemap-plugin-echarts-integration
description: MineMap 与 ECharts 集成规范。用于散点、飞线、饼图、雷达图、3D 柱图等地图叠加可视化。
---

# MineMap Plugin: ECharts Integration

## Architecture Positioning

ECharts 集成不是 MineMap 内核导出的标准渲染对象，而是 demo 明确展示的一类“外部图表叠加层”。

当前 demo 至少给出了三条可靠路径：

1. 组件写法：复用 `demo/js/minemap-echarts-layer.js` 提供的 `MinemapEChartsLayer`
2. 内嵌写法：在业务页面内直接实现 `InlineEChartsLayer`
3. 手工 overlay 写法：业务页自己创建容器、`echarts.init(...)`、自行绑定地图同步事件

这些路径本质都依赖同一件事：

-   用 `map.project([lng, lat])` 把地理坐标转成屏幕像素
-   在地图事件触发时重算图表位置
-   把 ECharts 实际绘制挂到地图容器之上的绝对定位 DOM 层

其中：

-   `echarts.html` 属于“手工 overlay 写法”
-   `echarts-line-chart-overlay.html` 属于“页面内展开实现 layer”
-   `echarts-*-simple.html` 系列更多体现“复用 `MinemapEChartsLayer` + 业务图表分离”

## Supported Patterns Confirmed by Demo

`demo/js/minemap-echarts-layer.js` 已明确支持：

-   `scatter`
-   `effectScatter`
-   `lines`
-   `pie`

各自的地理输入约定为：

-   `scatter` / `effectScatter`: `value: [lng, lat, ...]`
-   `lines`: `data[i].coords = [[lng, lat], [lng2, lat2], ...]`
-   `pie`: `centerCoord: 'minemap'` 且 `center: [lng, lat]`

demo 还补充了典型业务形态：

-   航线迁移图
-   地图锚点折线图
-   地图锚点饼图
-   地图锚点雷达图
-   地图锚点柱状对比

## Recommended Workflow

1. 先初始化 MineMap
2. 等 `map.on('load', ...)`
3. 再创建 ECharts overlay layer
4. 图表数据保持地理坐标，不要预先手写像素值
5. 把同步责任交给 overlay layer，而不是业务手工监听每个相机事件
6. 页面销毁时主动 `dispose()`

## Demo-backed Patterns

### 模式 1：通用叠加层组件

对应文件：`demo/js/minemap-echarts-layer.js`

这是当前最值得复用的模式，因为它把这几个关键问题都封装了：

-   叠加容器创建
-   `echarts.init(...)`
-   `move` / `zoom` / `rotate` / `pitch` / `resize` 同步
-   `coordinateSystem: 'minemap'` 转 `cartesian2d`
-   Geo → Pixel 转换
-   `dispose()` 清理

如果业务只是要把 ECharts 叠到 MineMap 上，优先照这个组件型写法组织，而不是每个页面重复写一遍。

### 模式 2：原生展开版，便于调试

对应 demo：`demo/html/echarts-line-chart-overlay.html`

这个 demo 把 layer 逻辑直接展开在页面里，价值在于：

-   更容易调试坐标同步问题
-   更容易理解 `xAxis` / `yAxis` / `grid` 为什么要重建
-   更容易看清 `series.coordinateSystem = 'minemap'` 到 `cartesian2d` 的转换

适合首次排查：

-   相机变换后位置漂移
-   轴范围不一致
-   饼图 / 折线图中心点错位

### 模式 3：散点 / 飞线迁移图

对应 demo：`demo/html/echarts.html`

这是“地图主画布 + ECharts 整屏 overlay”的代表模式。适合：

-   航线网络
-   人口迁移
-   物流路径
-   城市联系图

这种模式通常不是每个点挂一个 DOM 图表，而是把整批图形一次性画在一个 ECharts canvas 上。

### 模式 4：地图锚点饼图

对应 demo：`demo/html/echarts-pie-chart-simple.html`

这里最关键的约束是：

-   饼图不能只靠 `coordinateSystem: 'minemap'`
-   要用 `centerCoord: 'minemap'`
-   `center` 必须仍然写地理坐标 `[lng, lat]`

而在 layer 里，饼图中心不走普通 y 翻转逻辑。

### 模式 5：地图锚点雷达图 + 反向联动

对应 demo：`demo/html/echarts-radar-chart-simple.html`

这个 demo 验证了另一件事：

-   可以通过 `echartsLayer.getEChartsInstance().on('click', ...)` 把图表交互反向驱动地图业务状态

也就是说，ECharts 叠加层不只是“显示”，也能作为交互入口。

### 模式 6：地图标记 + 旁路独立分析图表

对应 demo：

-   `demo/html/echarts-bar-chart-3d-simple.html`
-   `demo/html/echarts-radar-chart-simple.html`

这类 demo 采用“双图层职责分离”：

-   地图上的点位高亮：ECharts overlay
-   侧边或浮层分析图：普通 `echarts.init(container)`

这是很推荐的组织方式，因为地图同步层和复杂分析图层职责不同，不要强行混成一个实例。

## Strict Constraints

### 1. 不要误以为这是 MineMap 内核内建坐标系

`coordinateSystem: 'minemap'` 是 demo/plugin 层自己解释的协议，不是说明 MineMap 官方内核天然认识 ECharts。

### 2. 只有部分图表类型已被 demo 明确处理

当前可靠证据集中在：

-   `scatter`
-   `effectScatter`
-   `lines`
-   `pie`

超出这些类型时，要先补坐标转换逻辑，再谈复用。

### 3. 饼图与普通散点的坐标转换不同

`minemap-echarts-layer.js` 里明确区分了饼图：

-   饼图不做 y 轴翻转
-   其他图表通常要做 y 翻转以匹配 ECharts 坐标系

### 4. 相机变换必须同步

只在初始化时算一次像素坐标是不够的。至少要绑定：

-   `move`
-   `zoom`
-   `rotate`
-   `pitch`
-   `resize`

### 5. 页面销毁时必须清理

若不 `dispose()`：

-   地图事件监听会残留
-   ECharts canvas 会残留
-   容器 DOM 会残留

## Failure Cases

### 失败场景 1：把所有图表都做成 DOM Marker

当点位很多时，Marker + DOM 图表会很快变重。大批量情况下，优先整屏 ECharts overlay。

### 失败场景 2：业务层自己重复维护像素坐标

这会在缩放、旋转、倾斜后迅速失真。正确路径是始终保存地理坐标，把像素换算推迟到 overlay 层。

### 失败场景 3：图表能点，但地图不能动

这通常是 overlay 容器 `pointer-events` 策略没设计好。`minemap-echarts-layer.js` 已把它做成可配置项。

## Performance Discipline

-   高频相机事件更新一定要节流，demo 默认就是节流更新
-   批量点线尽量合并成单个 ECharts 实例
-   大图表分析面板与地图 overlay 分开维护
-   `resize()` 与位置更新一起做，避免容器尺寸和坐标基准不一致

## Demo References

-   `demo/js/minemap-echarts-layer.js`
-   `demo/html/echarts.html`
-   `demo/html/echarts-line-chart-overlay.html`
-   `demo/html/echarts-line-chart-simple.html`
-   `demo/html/echarts-pie-chart-simple.html`
-   `demo/html/echarts-radar-chart-simple.html`
-   `demo/html/echarts-bar-chart-3d-simple.html`

## See Also

-   `minemap-events-and-picking`
-   `minemap-performance-and-backend`
-   `minemap-business-airline-and-lines`
-   `minemap-fundamentals`
