---
name: minemap-business-admin-division
description: MineMap 行政区划业务规范。用于区域拉伸、边界墙体、专题区划展示。
---

# MineMap Business: Admin Division

## Quick Start

```javascript
const adminDivision = new minemap.AdminDivision({
	divisions: [
		{
			hierarchy: {
				positions: [116.39, 39.9, 116.4, 39.9, 116.4, 39.91, 116.39, 39.91, 116.39, 39.9],
				holes: []
			},
			height: 500
		}
	]
});
map.addSceneComponent(adminDivision);
```

## Architecture Positioning

`AdminDivision` 不是简单的填充面，而是一个复合场景对象。源码里它会为每个区划组合生成：

-   区划面
-   墙体
-   外边框

因此它适合“专题区域起鼓表达”，不适合拿来替代所有普通 polygon 图层。

## Source-backed Rules

-   `AdminDivision` 是 `scene-object`
-   构造参数核心为 `divisions`
-   每个 division 需要 `hierarchy` 和 `height`
-   `hierarchy` 里包含：
    -   `positions`
    -   `holes`
-   内部会缓存两套 geometry：
    -   `planed`
    -   `extruded`
-   可显式调用：
    -   `planing()`
    -   `extruding()`
-   墙体默认会使用引擎资产纹理 `minemap://assets/division-edge.png`

## Demo-backed Usage Pattern

对应 demo：`demo/html/ThematicMapOfAdministrativeDivisionsExt.html`

这个 demo 说明了两个业务上很实用的策略：

1. 行政区对象可以一次加多个实例，分别表达不同区域
2. 为了让区划起鼓更明显，可以同步压暗底图，例如调整 raster 的 `raster-brightness-max`

## Recommended Workflow

1. 先清洗行政区数据
    - 外环闭合
    - 孔洞结构完整
2. 根据业务层级确定 `height`
3. 生成 `divisions`
4. 添加到地图
5. 若需要与镜头状态联动，可在合适时机在平铺态和拉伸态之间切换

## Strict Constraints

### 1. 输入结构必须是 `hierarchy.positions + hierarchy.holes`

当前实现不是只吃一串 positions。业务数据整理阶段必须明确外环与洞。

### 2. 环必须尽量闭合、方向稳定

虽然几何库会尽量处理，但正式项目里不应依赖运行时“碰运气修补”。

### 3. `height` 是业务视觉参数，不是越高越好

高度过大时会带来：

-   墙体过于夸张
-   与周边三维模型相互遮挡
-   镜头近距离下阅读困难

## Failure Cases

### 失败场景 1：拿 `AdminDivision` 绘制海量基层网格

不合适。它更适合少量重点行政区、责任区、专题区域，而不是海量细碎面。

### 失败场景 2：孔洞数据不完整

会导致：

-   面填充错误
-   墙体包裹错误
-   外边框与真实区域不一致

### 失败场景 3：只做起鼓，不做视觉陪衬

如果底图和周边场景太亮，区划起鼓效果会不明显。demo 已证明，必要时应联动底图亮度或周边样式。

## Business Recommendations

### 适合的业务

-   省市区专题图
-   园区责任区
-   选区高亮
-   多区域对比展示

### 不适合的业务

-   常规行政边界底图
-   海量细碎面要素渲染
-   强交互编辑型 polygon 工具

## Demo References

-   `demo/html/ThematicMapOfAdministrativeDivisionsExt.html`
-   `demo/html/ThematicMapOfAdministrativeDivisions.html`

## See Also

-   `minemap-business-measurement`
-   `minemap-primitives-and-materials`
