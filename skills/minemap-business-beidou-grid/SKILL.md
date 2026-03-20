---
name: minemap-business-beidou-grid
description: MineMap 北斗网格码业务规范。用于网格编码可视化、动态调度、网格高亮与空间索引表达。
---

# MineMap Business: BeiDou Grid

## Quick Start

```javascript
const grid = new minemap.BeiDouGrid({
	autoGenerateGridData: false,
	gridData: [
		{
			id: "N048J00",
			color: new minemap.Color(0.8, 0.0, 0.0, 0.3),
			outlineColor: "black"
		}
	]
});
map.addSceneComponent(grid);
```

## Architecture Positioning

北斗网格不是普通 layer，而是专门的场景对象，类型为 `beiDouGrid`。源码与 demo 一致表明它有两条主路径：

1. 动态生成模式

-   `autoGenerateGridData !== false`
-   引擎按视锥和层级自动计算

2. 静态网格模式

-   `autoGenerateGridData: false`
-   由业务直接提供 `gridData`

不要把这两个模式混着设计需求。

## Source-backed Rules

-   运行时对象类型是 `beiDouGrid`
-   `maxLevel` 默认按源码取 `min(options.maxLevel || 6, 10)`
-   源码注释已明确：7 级及以后可能因数据过大导致性能下降甚至崩溃
-   `minHeight` / `maxHeight` 修改后会销毁并重建内部调度树
-   `freezeFrame` 只对自动生成模式有意义
-   静态模式下，如果 `gridData` 没有 `cartesianCorners`，引擎会尝试按 `id` 进行北斗码反算

## Demo-backed Business Patterns

### 模式 1：全图层动态网格对象

对应 demo：`demo/html/BeiDouGridSceneObject.html`

适合：

-   全域浏览
-   网格级定位
-   网格级点击查询

这个 demo 实际证明了标准交互链路：

1. `map.getPositionByGPUPick(e.point, { ignoreNoneAllowPick: true })`
2. `beiDouGrid.getPickedGridByPosition(pos)`
3. `beiDouGrid.setHighlights([picked])`
4. popup 展示 `id / level / lngLatHeightCorners / property`

### 模式 2：编码结果反显为 3D 网格

对应 demo：`demo/html/BeiDouCodecDemo.html`

适合：

-   调度结果复核
-   编码算法联调
-   指定网格码反查空间范围

这个 demo 很关键，因为它说明了静态模式的正确组织方式：

-   用 `minemap.BeiDouGrid.convertCodeToPoint(code)` 或 codec 工具拿到角点
-   组装 `gridData`
-   `autoGenerateGridData: false`
-   一个或多个 `BeiDouGrid` 组件加入地图
-   需要时通过 `map.removeSceneComponent(component.id)` 回收

### 模式 3：北斗网格覆盖 3D Tiles

对应 demo：`demo/html/3DTilesWithBeiDouGrid.html`

适合：

-   实景模型上的网格调度可视化
-   建筑、园区、城市块级编码结果叠加
-   服务端预计算网格边界回传后直接渲染

这个 demo 还给出一个很实用的做法：

-   如果页面中同时有多个北斗网格组件，可通过 `map.style.beiDouGridCollection.getAll()` 遍历统一控制显示与拾取

## Recommended Workflow

### 全域动态浏览

1. 创建自动网格对象
2. 限制 `maxLevel`
3. 通过 `minHeight / maxHeight` 收敛高度域
4. 在查询时临时高亮
5. 大屏静态展示时可开启 `freezeFrame`

### 编码结果展示

1. 服务端或业务侧产生北斗码
2. 转换为角点或直接传编码
3. 用静态 `gridData` 创建组件
4. 按批次管理组件 id
5. 及时移除无用组件

## Strict Constraints

### 1. `maxLevel` 不要盲目抬高

源码已给出明确信号：7 级以后风险明显增大。业务上没有充分验证前，不要把它当常规配置项开放给最终用户随意拉高。

### 2. 自动模式不适合承载大批固定业务网格

如果网格是：

-   告警网格
-   任务网格
-   指定清单网格

优先静态模式，不要让引擎每帧替你动态算一遍。

### 3. 高度上下限必须由业务兜底校验

demo 已明确做了 `maxHeight < minHeight` 的互斥修正。业务 UI 层也必须复制这类保护。

### 4. 颜色省略时会走随机色

这适合调试，不适合正式业务。正式项目应显式给出 `color` 和 `outlineColor`。

## Failure Cases

### 失败场景 1：全国级视图直接开高层级动态网格

极易导致：

-   结果过密
-   交互不可读
-   设备压力上升

### 失败场景 2：每个编码结果创建一个长期存活组件

如果编码结果很多，组件颗粒度过细会增加管理成本。应优先批量合并到少量 `BeiDouGrid` 组件里。

### 失败场景 3：用网格对象代替常规行政区表达

北斗网格强调编码体系与索引表达，不适合承担行政专题图的主要视觉职责。

## Demo References

-   `demo/html/BeiDouGridSceneObject.html`
-   `demo/html/BeiDouCodecDemo.html`
-   `demo/html/3DTilesWithBeiDouGrid.html`

## See Also

-   `minemap-business-admin-division`
-   `minemap-events-and-picking`
-   `minemap-business-oblique-bim`
