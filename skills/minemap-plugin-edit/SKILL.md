---
name: minemap-plugin-edit
description: MineMap 编辑插件规范。用于要素绘制、选中、样式覆写、锁定、批量写入与叠加编辑。
---

# MineMap Plugin: Edit

## Architecture Positioning

`Edit` 不是 `source/index.js` 里的内建主干类，而是 demo 明确依赖的外挂编辑插件。

从 demo 看，它的真实接入方式是：

1. 额外引入编辑插件脚本
2. 部分 demo 还会额外引入编辑插件样式
3. 在地图 `load` 后执行 `new minemap.edit.init(map, options)`

注意：这些插件资源在 demo 里主要通过远程 / 内网 URL 引入，并不是当前仓库里统一打包好的正式源码入口。因此文档只能把它视为“demo 已验证的外部插件依赖”，不能把它描述成 MineMap 主包内建能力。

所以它更像“编辑工作台层”，而不是 style layer、scene component 或 primitive。

## Demo-backed Entry Surface

以下能力都已在 demo 中出现：

-   `new minemap.edit.init(map, options)`
-   `edit.onBtnCtrlActive(mode)`
-   `edit.setCustomStyle(style)`
-   `edit.setFeatures(featureCollection)`
-   `edit.draw.add(featureOrCollection)`
-   `edit.draw.getSelectedIds()`
-   `edit.draw.deleteAll()`
-   `edit.setFeaturePropertiesByIds(ids, properties)`
-   `edit.setLockByIds(ids, true)`
-   `map.on('edit.record.create', handler)`

这说明业务上应把它当作“独立插件实例 + 内部 draw 池”的模式使用。

## Recommended Workflow

1. 先初始化地图
2. 在 `map.on('load', ...)` 中创建 `edit`
3. 根据业务决定是否显示默认控件按钮
4. 先写入初始要素，再进入交互编辑
5. 样式覆写优先通过 `setCustomStyle(...)` 或 `setFeaturePropertiesByIds(...)`
6. 若业务有底图/三维叠加，最后再处理 `classificationType`、锁定与交互拾取冲突

## Demo-backed Patterns

### 模式 1：工具栏模式切换

对应 demo：`demo/html/Edit.html`

最基础的控制方式不是直接调用一堆 draw 子方法，而是统一走：

-   `edit.onBtnCtrlActive('point')`
-   `edit.onBtnCtrlActive('line')`
-   `edit.onBtnCtrlActive('polygon')`
-   `edit.onBtnCtrlActive('rectangle')`
-   `edit.onBtnCtrlActive('circle')`
-   `edit.onBtnCtrlActive('trash')`
-   `edit.onBtnCtrlActive('undo')`
-   `edit.onBtnCtrlActive('redo')`

这说明：如果业务要做自定义工具条，最稳的做法是“自己做按钮 UI，但把状态切换仍委托给插件”。

### 模式 2：批量灌入初始几何

对应 demo：`demo/html/LineRoundCap.html`

已有 `FeatureCollection` 时，优先：

-   `edit.setFeatures(fc)`

要在现有编辑池继续追加时，再用：

-   `edit.draw.add(featureCollection)`

这两个入口语义不同，不建议混成一个“万能导入函数”理解。

### 模式 3：按要素 ID 覆写样式

对应 demo：`demo/html/LineRoundCap.html`

插件允许在写入后按 `id` 批量改属性：

-   `edit.setFeaturePropertiesByIds(ids, {...})`

demo 已验证的属性包括：

-   `custom_style`
-   `lineWidth`
-   `lineColor`
-   `fillColor`
-   `fillOpacity`
-   `fillOutlineColor`
-   `fillOutlineWidth`
-   `fillOutlineDasharray`

因此，“编辑态样式”与“业务属性”是可以放在同一个 feature properties 体系里协同管理的。

### 模式 4：锁定底稿，仅允许编辑活动要素

对应 demo：`demo/html/VectorGeojsonOverlap.html`
和 `demo/html/FillLayerWithOverlapPolygon.html`

这里要分两种已在 demo 出现的写法：

1. `VectorGeojsonOverlap.html`

    - 先 `edit.draw.add(feature)`
    - 再 `edit.setLockByIds([activeId], true)`

2. `FillLayerWithOverlapPolygon.html`
    - 先 `edit.setFeatures(otherOrgFeature)` 放入底稿
    - 再通过 `edit.draw.add(...)` 继续追加当前活动要素与轮廓要素
    - 这个 demo 里虽然接收了 `setFeatures(...)` 的返回值，但没有显式调用 `setLockByIds(...)`

因此，能被明确证明的“显式锁定 API 用法”来自 `VectorGeojsonOverlap.html`；`FillLayerWithOverlapPolygon.html` 更适合当作“底稿 + 当前编辑要素分池组织”的证据。

这类模式适合：

-   行政区修编
-   红线修订
-   方案比对
-   只允许编辑新增层，不允许破坏基准层

### 模式 5：三维叠加场景下把编辑层抬到上面

对应 demo：`demo/html/VectorGeojsonOverlap.html`

在编辑初始化参数里，demo 出现了：

-   `classificationType: true`

从注释语义看，它用于让 edit 相关图层位于三维图层之上。业务只要涉及：

-   倾斜摄影
-   模型底图
-   三维覆盖物上方的 2D 编辑

都应优先检查这个开关。

## Strict Constraints

### 1. 必须把插件当外部依赖管理

`Edit` 不是 MineMap 主包自动可用对象。不要在没有额外引入插件脚本的前提下，直接假设 `minemap.edit` 一定存在。

### 2. 一定等地图 `load` 后再初始化

demo 都是在 `map.on('load', ...)` 中创建插件实例。编辑层本质上依赖地图内部样式与交互状态，过早初始化容易出时序问题。

### 3. 自定义样式要和 feature properties 体系一致

当你已经用 `custom_style`、`lineWidth`、`fillColor` 这类属性驱动样式时，就不要再在别处维护第二套脱节样式状态。

### 4. 锁定与可编辑要素要分开管理

底稿锁定后再追加活动要素，是 demo 支持的可靠路径。不要把“可展示”和“可编辑”混成一个 feature pool。

## Failure Cases

### 失败场景 1：把编辑插件当普通 GeoJSON source

不合适。它不是只负责渲染的 source/layer，而是带交互状态、选择态、撤销重做与编辑生命周期的插件。

### 失败场景 2：地图未加载完成就直接 `new minemap.edit.init(...)`

容易出现交互层、样式层或事件绑定状态不稳定的问题。

### 失败场景 3：在复杂三维场景中忽略层级冲突

若你有模型、地形、栅格、贴地面图层同时存在，编辑图形可能被遮挡；这时要复核：

-   `classificationType`
-   栅格 `depth-test`
-   二三维叠加层级策略

## Performance Discipline

-   长时间编辑会话优先复用一个 `edit` 实例，不要反复销毁重建
-   批量导入时优先整包 `FeatureCollection`
-   大量底稿优先锁定，不要全部维持在可编辑高频状态
-   三维重场景里先控制交互图层数量，再叠加复杂样式

## Demo References

-   `demo/html/Edit.html`
-   `demo/html/LineRoundCap.html`
-   `demo/html/VectorGeojsonOverlap.html`
-   `demo/html/FillLayerWithOverlapPolygon.html`

## See Also

-   `minemap-style-and-data`
-   `minemap-events-and-picking`
-   `minemap-2d-3d-overlay-and-classification`
-   `minemap-primitive-adapt-terrain`
