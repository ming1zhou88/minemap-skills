---
name: minemap-business-flood-and-shadow
description: MineMap 淹没分析与阴影分析业务规范。用于防灾、排涝、日照评估、园区环境分析。
---

# MineMap Business: Flood and Shadow

## Quick Start

```javascript
const flood = new minemap.FloodAnalysisOffline({
	positions: [116.39, 39.9, 116.4, 39.9, 116.4, 39.91, 116.39, 39.91],
	height: 2,
	color: "red",
	opacity: 0.5
});
map.analysis.floodAnalysisOfflineCollection.add(flood);
```

## Available Analysis Containers

地图初始化后，分析对象统一挂在 `map.analysis`：

-   `floodAnalysisCollection`
-   `floodAnalysisOfflineCollection`
-   `shadowAnalysisCollection`

业务代码不要自行再维护一套平行容器。

## Demo-backed Distinction

MineMap 的淹没业务至少要区分两类，不要只写成一个概念：

### `FloodAnalysis`

对应 demo：`demo/html/FloodAnalysis.html`

特点：

-   依赖后端高程/体积服务
-   通过服务结果反推 `minHeight` 与不同阶段的 `height`
-   更适合“按水量/流量推演”的业务

### `FloodAnalysisOffline`

对应 demo：`demo/html/FloodAnalysisOffline.html`

特点：

-   不依赖同类后端填高服务
-   直接基于圈定区域、本地高度区间和速度进行演示
-   更适合离线预演、演示、教学和本地工具链

### `ShadowAnalysis`

对应 demo：`demo/html/ShadowAnalysis.html`

特点：

-   面向日照/遮阴率分布
-   与太阳时间、分析区域、分析高度范围强相关
-   更像“区域统计分析”，不是普通阴影开关

## Flood Analysis Guidance

### 更适合的场景

-   固定范围淹没推演
-   演练回放
-   风险分区预演

### 输入要求

-   `positions` 必填
-   `height` 是相对淹没高度
-   `color`、`opacity` 用于表达风险等级

### 运行特征

-   修改 `height` 会触发淹没区域几何更新
-   更适合“区域确定、反复调水位”的业务流

### Demo 实战结论

`FloodAnalysis.html` 不是单纯手动改一个水位，它体现的是完整业务流程：

1. 先画区
2. 再根据业务输入请求服务
3. 拿到不同阶段的高度数组
4. 通过滑条或自动播放推进 `floodAnalysis.height`
5. 同步更新图例

这说明在线淹没推演更适合“时间轴回放”，而不是单帧展示。

### `FloodAnalysisOffline` 适用边界

离线版更适合：

-   只关心水位上升过程演示
-   不需要实时体积换算
-   本地快速联调

## Shadow Analysis Guidance

### 核心输入

-   `positions`：闭合面，且至少 4 个点
-   `minHeight`
-   `maxHeight`
-   `date`
-   `colorRamp`
-   `colorRate`
-   `verticalScale`
-   `horizontalScale`

### 强约束

-   首尾点必须闭合，否则不能构成分析面
-   `colorRamp.length === colorRate.length`
-   `maxHeight` 若存在，应高于 `minHeight`

### 结果表达

阴影分析本质上是时间与区域采样后的颜色映射结果，`colorRamp` 与 `colorRate` 要直接对齐业务含义，不要只做视觉配色。

### Demo 实战结论

`ShadowAnalysis.html` 给出几个很关键的工程经验：

-   倾斜摄影未加载完成前，不要急着做分析
-   可通过 `map.areTilesLoaded()` 等待场景稳定
-   时间驱动要联动 `map.style.sunLight.setTime(...)`
-   分析面既可以参数写死，也可以交互绘制后闭合
-   运行时允许调 `minHeight`、`maxHeight`、`horizontalScale`、`verticalScale`

## Recommended Business Patterns

### 排涝 / 积水演练

1. 先圈定区域
2. 用离线淹没对象反复调整 `height`
3. 与地形同时使用时，先确认当前地形已经稳定加载

### 日照评估 / 遮阴评估

1. 用闭合多边形定义评估区
2. 明确日期
3. 用业务阈值反推 `colorRate`
4. 控制采样尺度，先粗后细
5. 等待 3D 场景稳定加载后再启动分析

## Performance Discipline

-   分析区不要一开始就铺满全场景
-   先做小范围验证，再扩展到生产范围
-   大范围阴影分析要控制 `verticalScale/horizontalScale`
-   分析时可临时关闭高成本后处理，保证结果更新稳定

## Common Failure Cases

-   淹没面输入未闭合
-   阴影色带与阈值数组长度不一致
-   分析区域过大且采样过密，导致性能快速恶化
-   地形未就绪就开始批量分析，结果不稳定
-   倾斜摄影还没加载完就开始 `ShadowAnalysis`
-   在线淹没和离线淹没混为一套交互，导致参数含义错位

## Demo References

-   `demo/html/FloodAnalysis.html`
-   `demo/html/FloodAnalysisOffline.html`
-   `demo/html/ShadowAnalysis.html`

## See Also

-   `minemap-business-visibility-analysis`
-   `minemap-terrain-and-analysis`
-   `minemap-performance-and-backend`
