---
name: minemap-business-gaussian-splatting
description: MineMap Gaussian Splatting 业务规范。用于高保真实景重建、轻量化点场景渲染、照片测量成果展示。
---

# MineMap Business: Gaussian Splatting

## Architecture Positioning

MineMap 里的高斯泼溅有两条落地路径，业务上不要混用概念：

1. `new minemap.GaussianSplatting(...)`
    - 面向单个 `.splat` 资产
    - 作为 `scene-object` 挂到场景里
    - 适合单体成果、室内扫描件、文保小场景、设备级演示
2. 高斯泼溅封装进 3D Tiles
    - demo 里已有 `3DGS-3DTiles.html`
    - 适合分块调度、城市级分区加载、与常规 tiles 统一管理

简单判断：

-   单体资产展示，优先 `GaussianSplatting`
-   需要分块调度、飞行定位、统一 LOD 管理，优先 3D Tiles 化

## Quick Start

```javascript
const gs = new minemap.GaussianSplatting({
	id: "gauss-1",
	data: "/data/model.splat",
	position: [116.39, 39.9, 0],
	rotation: [-Math.PI / 9, 0, 0],
	scale: [100, 100, 100]
});

map.addSceneComponent(gs);
```

## Source-backed Rules

-   `GaussianSplatting` 本质是 `scene-object`
-   当前公开数据入口以 `.splat` 文件地址为主，源码里通过 `getArrayBuffer()` 拉取
-   内部流程不是“直接画点”，而是：
    1. worker 传输原始 splat 数据
    2. worker 生成高斯纹理
    3. 按当前相机 `viewProj * modelView` 排序
    4. 把排序结果回写索引缓冲
-   因为存在视角相关排序，所以相机剧烈变化时会不断触发重排
-   `rotation` 与 `quaternion` 是二选一的旋转输入
-   `scale` 默认语义是 xyz 独立缩放，严禁写 0

## Demo-backed Usage Patterns

### 模式 1：空底图 + 单体高斯资产

对应 demo：`demo/html/3DGaussianSplatting.html`

适合：

-   文物件
-   工业零件
-   室内扫描单体
-   展陈型资产

这个 demo 的关键信号是：

-   用空 style 起图
-   先挂一个 raster 影像底图作参考
-   直接创建 `GaussianSplatting`
-   通过 `position + rotation + scale` 把资产对到地图坐标

### 模式 2：高斯内容做成 3D Tiles

对应 demo：`demo/html/3DGS-3DTiles.html`

适合：

-   大范围成果分块发布
-   需要 `sourceLoaded` 后自动飞行
-   需要 `maximumScreenSpaceError`、`skipLevelOfDetail` 这一类调度参数
-   需要后续与倾斜摄影、BIM、点云一起统一运维

## Recommended Workflow

1. 先确认资产边界
    - 单体成果：直接 `.splat`
    - 大场景成果：优先切成 3D Tiles
2. 先调 `position`
3. 再调 `rotation` 或 `quaternion`
4. 最后调 `scale`
5. 确认视角切换时的排序成本是否可接受
6. 如果业务是大屏持续漫游，评估是否改走 3D Tiles 方案

## Strict Constraints

### 1. `scale` 不能出现 0

源码注释已明确提醒不要随意设置成 0。业务上只要出现以下情形，就优先怀疑缩放：

-   模型完全不显示
-   包围盒异常
-   旋转后形态塌缩

### 2. 高频镜头运动会增加排序压力

高斯不是普通 mesh。它依赖视角排序，因此：

-   静态观察、缓慢浏览最合适
-   高频第一人称穿梭、大范围自动漫游，成本明显更高

### 3. 大成果不要长期以“多个 `.splat` 同屏”硬顶

如果业务已经出现：

-   多楼栋
-   多区块
-   多期成果并排对比

则应优先切换到分区调度方案，而不是继续堆多个 `GaussianSplatting` 实例。

## Failure Cases

### 失败场景 1：把高斯泼溅当成 BIM 替代

不合适。高斯适合“高保真展示”，不适合承担：

-   构件级拾取
-   精细业务属性挂接
-   规则化构件管理

这类场景仍然优先 `SceneModel` / `SceneTileset`。

### 失败场景 2：城市级成果仍然走单文件 `.splat`

会遇到：

-   初次加载重
-   相机排序压力大
-   无法像 3D Tiles 那样自然分块管理

### 失败场景 3：定位靠肉眼乱调

高斯资产落图时，最容易浪费时间的是“边看边猜”。正确做法是：

-   先确定成果真实锚点
-   再调旋转
-   最后只做小范围尺度修正

## Professional Decision Matrix

### 选 `GaussianSplatting`，如果你要：

-   最快展示单个高保真扫描件
-   控制少量资产的展陈效果
-   在 demo / 汇报 / 局部场景中强调视觉真实感

### 选高斯 3D Tiles，如果你要：

-   大范围分块加载
-   与常规 3D Tiles 调度统一
-   更可运营的发布链路

## Performance Discipline

-   一屏内高斯实例数量要严格受控
-   只在确实需要照片级外观时开启高斯成果
-   漫游镜头尽量减少高频急转和大幅抖动
-   大屏项目优先做分区启停，而不是让所有高斯内容常驻

## Demo References

-   `demo/html/3DGaussianSplatting.html`
-   `demo/html/3DGS-3DTiles.html`

## See Also

-   `minemap-business-oblique-bim`
-   `minemap-scene-components`
-   `minemap-performance-and-backend`
