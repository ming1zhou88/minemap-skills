---
name: minemap-post-process-and-render-pipeline
description: MineMap 后处理阶段与渲染效果链规范。用于夜视、黑白、雨雪、雾效、描边、天际线等屏幕后处理。
---

# MineMap Post-process and Render Pipeline

## Quick Start

```javascript
map.on("load", () => {
	const nightVision = minemap.PostProcessStageLibrary.createNightVisionStage();
	map.postProcessStages.add(nightVision);
});
```

## Core Concepts

### 1) 公开入口

MineMap 后处理的正式开发入口是：

-   `map.postProcessStages`
-   `minemap.PostProcessStageLibrary`
-   `minemap.PostProcessStage`
-   `minemap.PostProcessStageComposite`

推荐优先级：

1. 先用 `PostProcessStageLibrary`
2. 再用 `PostProcessStageComposite`
3. 最后才自己构造底层 `PostProcessStage`

### 2) 运行位置

后处理是**屏幕后处理链**，它作用在当前帧渲染结果之上，不是：

-   style layer
-   scene component
-   primitive

因此它更适合：

-   观感增强
-   氛围效果
-   特定视觉分析表达

不适合承担基础业务表达。

### 3) 与性能/后端的关系

后处理必须放在渲染预算后面考虑：

-   先让主场景稳定渲染
-   再叠加后处理
-   `webgl1` 下要更保守
-   多阶段组合不要默认常驻全开

## Demo-backed Patterns

### 模式 1：单阶段直接启用

对应 demo：`demo/html/PostProcessNightVision.html`、`demo/html/PostProcessBlackAndWhite.html`

共同特点：

-   直接通过 `PostProcessStageLibrary.createXXXStage()` 创建
-   再 `map.postProcessStages.add(...)`

这是最推荐的起点，适合：

-   夜视
-   黑白
-   单一观感效果

### 模式 2：天气型后处理

对应 demo：`demo/html/PostProcessRain.html`、`demo/html/PostProcessSnow.html`、`demo/html/PostProcessFog.html`

这类效果说明：

-   MineMap 把雨、雪、雾都放进了 post-process 家族
-   它们更像“全屏气氛层”，不是场景内真实几何对象
-   业务上更适合做模式切换，不适合在多个效果间长期叠加

### 模式 3：描边与天际线

对应 demo：`demo/html/PostProcessOutline.html`、`demo/html/PostProcessSkyline.html`

这类效果比黑白/夜视更接近“分析辅助表达”：

-   `createOutlineStage(...)` 适合强调目标对象
-   `createSkyline(...)` 适合城市轮廓、地平线强化

其中描边 demo 直接使用：

-   `PostProcessStageLibrary.createOutlineStage(map.painter, params)`

说明这类效果和渲染器内部状态耦合更深，应避免自己手搓替代实现。

## Recommended Workflow

1. 先确认业务目标是不是“后处理”而不是“场景对象特效”
2. 优先查 `PostProcessStageLibrary` 是否已有现成 stage
3. 先单阶段验证视觉和性能
4. 再考虑多阶段组合
5. 最后才进入自定义 shader stage

## Strict Constraints

### 1. 后处理不是默认模板项

不要在地图初始化时把所有视觉效果一起打开。

### 2. 多阶段链要控制叠加数量

后处理链越长：

-   显存和 framebuffer 压力越大
-   每帧 full-screen pass 越多
-   调试复杂度越高

### 3. 优先用库函数，不优先自定义 stage

`PostProcessStageLibrary` 已经覆盖了 MineMap 常用能力，优先复用官方阶段，少写自定义 fragment chain。

### 4. 后处理不替代真实业务状态

例如：

-   描边可以强调对象
-   但不能替代真正的选中状态管理
-   雾效可以增强氛围
-   但不能替代真实气象数据表达

## Failure Cases

### 失败场景 1：把后处理当成所有特效的统一解

不对。很多效果应优先考虑：

-   粒子系统
-   材质动画
-   scene object
-   图层表达

后处理只解决屏幕空间效果。

### 失败场景 2：低端设备多阶段常驻

夜视、雨雪、雾、描边同时打开，会明显抬高成本。

### 失败场景 3：业务层直接依赖 `map.painter` 泛化开发

只有像描边这类源码和 demo 已明确这样组织的能力，才按官方模式用 `map.painter`；不要把它扩成普通业务默认入口。

## Common Patterns

### 夜视

```javascript
const stage = minemap.PostProcessStageLibrary.createNightVisionStage();
map.postProcessStages.add(stage);
```

### 黑白

```javascript
map.postProcessStages.add(minemap.PostProcessStageLibrary.createBlackAndWhiteStage());
```

### 雨雪雾效

```javascript
const rainStage = minemap.PostProcessStageLibrary.createRainStage({
	density: 0.6,
	speed: 1.0
});
map.postProcessStages.add(rainStage);
```

### 清理和切换

切换模式时优先：

-   保存 stage 引用
-   按对象启停或 remove
-   场景销毁时统一 `map.postProcessStages.removeAll()`

不要每次切换都只增不减。

## Performance Discipline

-   先测基础帧率，再启后处理
-   单次只开最必要的 1~2 个阶段
-   天气、描边、夜视不要长期无差别叠加
-   若后端解析为 `webgl1`，优先关闭复杂组合阶段

## Demo References

-   `demo/html/PostProcessNightVision.html`
-   `demo/html/PostProcessBlackAndWhite.html`
-   `demo/html/PostProcessRain.html`
-   `demo/html/PostProcessSnow.html`
-   `demo/html/PostProcessFog.html`
-   `demo/html/PostProcessOutline.html`
-   `demo/html/PostProcessSkyline.html`

## See Also

-   `minemap-performance-and-backend`
-   `minemap-anti-aliasing`
-   `minemap-business-particle-and-special-effects`
-   `minemap-terrain-and-analysis`
