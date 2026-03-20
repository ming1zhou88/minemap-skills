---
name: minemap-business-airline-and-lines
description: MineMap 航线、飞线、动态流光线业务规范。用于空中航线、物流飞线、态势连线、流向表达。
---

# MineMap Business: Airline and Lines

## Quick Start

```javascript
const airline = new minemap.AirLine({
	id: "airline-1",
	data: [[116.39, 39.9, 121.47, 31.23, 1.5, 80]],
	"airline-width": 1,
	"airline-seg-group": 5,
	"airline-speed": 30,
	"airline-opacity": 1,
	"airline-color": {
		steps: [
			[0, "#e9ebee"],
			[50, "#0154f0"],
			[100, "#e70707"]
		],
		"color-index": 6
	}
});
map.addSceneComponent(airline);
```

## Architecture Positioning

`AirLine` 是专门的场景对象，不是普通 polyline layer。它一次性同时承担两件事：

1. 静态航迹线
2. 动态流动效果

因此它适合“关系网络 + 流向表达”，而不是一般性折线编辑。

## Source-backed Rules

-   `AirLine` 类型为 `scene-object`
-   `options.data` 必填
-   单条数据格式为 `[起点lng, 起点lat, 终点lng, 终点lat, 高度, ...业务属性]`
-   源码默认值：
    -   `airline-width = 1`
    -   `airline-seg-group = 5`
    -   `airline-speed = 60`
    -   `airline-opacity = 1.0`
-   如果启用 `airline-color`，则 `steps` 与 `color-index` 必须同时提供
-   `color-index` 超出数据列范围会直接抛错
-   源码注释给出关键事实：一个 `1° x 1°` 的航迹线会被拆为 10 段

## Demo-backed Usage Pattern

对应 demo：`demo/html/AirLine.html`

这个 demo 基本就是标准业务接法：

1. 拉取远程航线 JSON
2. 直接传给 `new minemap.AirLine(...)`
3. 用 `airline-color.steps` 做阈值着色
4. 用 GUI 控制添加和移除

这说明 `AirLine` 非常适合：

-   航线网络
-   物流干线与支线
-   跨区流向图
-   态势联动飞线

## Recommended Workflow

1. 先定义数据列语义
    - 前 5 列固定为起终点与高度
    - 后续列用于强度、等级、状态、类型等业务属性
2. 如果要做分段颜色映射，再确定 `color-index`
3. 根据视觉密度调 `airline-width`
4. 根据拖尾长度调 `airline-seg-group`
5. 根据节奏调 `airline-speed`

## Strict Constraints

### 1. `color-index` 指向的是原始数据列

它不是任意字段名，而是数组下标语义。数据结构一旦改列顺序，颜色映射就会错。

### 2. 业务阈值必须和 `steps` 对齐

如果 `steps` 取值分布与真实数据不匹配，最终效果会表现为：

-   大部分线都是同一颜色
-   关键线路不突出
-   图例失真

### 3. 大批量飞线不能无上限堆叠

`AirLine` 内部会做分段与动态效果处理，因此它比静态线要重。面向全国网络或超大关系图时，必须做分级展示和筛选。

## Failure Cases

### 失败场景 1：把 `AirLine` 用在普通道路网

不推荐。道路网络通常需要：

-   更稳定的静态表达
-   更强的交互筛选
-   更低成本的海量渲染

这类需求优先常规线图层或其他线性对象。

### 失败场景 2：所有关系都同时显示

关系图一旦全开，视觉上会直接变成“光污染”。必须做：

-   分层级
-   分区域
-   分主题
-   分时段

### 失败场景 3：高度写死但缺乏业务分层

若所有飞线高度都一样，线路会严重重叠。高度值应与层级、权重或类别联动。

## Business Recommendations

### 航空网络

-   主干航线：更高高度、更快速度、更亮色阶
-   支线：降低高度与宽度

### 物流流向

-   用业务量做 `color-index`
-   速度不要过快，避免误导成“实时位移”

### 态势联动

-   只高亮当前事件链路
-   其余线路弱化或隐藏

## Demo References

-   `demo/html/AirLine.html`
-   `demo/html/echarts.html`（如果业务只是想做二维迁移线联动，可评估是否交给 ECharts 叠加层处理）

## See Also

-   `minemap-business-roaming-and-tracking`
-   `minemap-primitives-and-materials`
