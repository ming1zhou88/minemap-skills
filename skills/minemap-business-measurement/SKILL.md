---
name: minemap-business-measurement
description: MineMap 三维量算业务规范。用于面积、贴地面积、辅助测量与规划校核。
---

# MineMap Business: Measurement

## Quick Start

```javascript
const area = minemap.Measurement3D.area([
	new minemap.Math.Vector3(116.39, 39.9, 0),
	new minemap.Math.Vector3(116.4, 39.9, 0),
	new minemap.Math.Vector3(116.4, 39.91, 0),
	new minemap.Math.Vector3(116.39, 39.91, 0)
]);
console.log(area);
```

## Architecture Positioning

`Measurement3D` 是静态工具类，不是可直接挂图的对象。真正的业务落地通常要分成两层：

1. 几何交互层

-   采点
-   动态绘制辅助线/面

2. 量算层

-   `Measurement3D.area()`
-   `Measurement3D.distance()`
-   `Measurement3D.height()`
-   `Measurement3D.clampedArea()`
-   `Measurement3D.clampedDistance()`

demo 已经完整体现了这个分层思路。

## Source-backed Rules

-   `area(positions)`：空间面积
-   `distance(positions)`：空间距离
-   `height(startPoint, endPoint)`：三角高差，返回水平距、垂距、斜距
-   `clampedArea(positions, map, mostDetailed = false, sampleRatio = 400, minArea = 400)`：贴地面积
-   `clampedDistance(positions, map, mostDetailed = false, minDistance = 30)`：贴地距离
-   贴地量算依赖当前地形源；找不到 terrain 会报错
-   贴地面积与贴地距离内部都会做采样，并对点数做上限保护，超限会直接抛错

## Demo-backed Patterns

### 模式 1：空间量算

对应 demo：`demo/html/Measurement3D.html`

这个 demo 给出了最标准的交互约束：

-   面积：左键加点，右键结束
-   距离：左键加点，右键结束
-   高度：左键起点，左键终点，第二次点击即结束

同时它还说明一个重要实践：

-   `Measurement3D` 本身只负责算值
-   动态过程要靠 `Primitive + PolygonGeometry + PolylineGeometry` 自己画辅助图形

### 模式 2：贴地量算

对应 demo：`demo/html/ClampedMeasurement.html`

这个 demo 的要点比空间量算更重要：

-   绘制辅助 geometry 时要打开 `adaptTerrain`
-   结束后使用 `clampedArea()` / `clampedDistance()`
-   函数不仅返回数值，还返回贴地采样后的 geometry，可直接重新绘制结果

## Recommended Workflow

### 空间量算

1. 鼠标采点
2. 用 GPU pick 得到空间点
3. 动态绘制辅助线/面
4. 调用 `area` / `distance` / `height`
5. 输出文本结果

### 贴地量算

1. 确认地形已启用
2. 采集地表区域或路径
3. 调用 `clampedArea` / `clampedDistance`
4. 用返回 geometry 渲染最终贴地结果

## Strict Constraints

### 1. 输入类型必须正确

-   `area` / `distance` / `height` 主要吃 `Vector3`
-   贴地方法支持 `Vector2 | Vector3`

不要把 screen point、LngLat 对象、普通数组直接混进去。

### 2. 贴地量算不是无限细采样

源码里已有明确保护：

-   贴地面积采样点过多会抛错
-   贴地距离采样点过多也会抛错

因此大范围复杂区域必须先裁剪。

### 3. `height()` 不是“地形高程查询”

它算的是两个三维点之间的三角测量结果，不要和 DEM 高程读取混淆。

## Failure Cases

### 失败场景 1：在未开启地形时做贴地量算

结果不可靠，甚至直接报错。贴地量算前必须先确认 terrain 可用。

### 失败场景 2：复杂大面直接做贴地面积

会遇到采样点暴涨。正确做法是：

-   先裁剪
-   再分区
-   再汇总

### 失败场景 3：把量算函数当 UI 工具

`Measurement3D` 不管理交互状态，也不负责界面提示。业务层必须自己实现开始、撤销、结束、重置。

## Business Recommendations

### 用空间量算，如果你在做：

-   建筑屋顶面积估算
-   构筑物间距测算
-   设备到设备的空间高差

### 用贴地量算，如果你在做：

-   山地范围评估
-   巡检路线贴地长度
-   地表真实覆盖面积

## Demo References

-   `demo/html/Measurement3D.html`
-   `demo/html/ClampedMeasurement.html`

## See Also

-   `minemap-terrain-and-analysis`
-   `minemap-business-admin-division`
