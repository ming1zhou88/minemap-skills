---
name: minemap-3d-tiles-runtime-control
description: MineMap 3D Tiles 运行时控制规范。用于 TileFeature 着色/显隐、实例拾取、加载完成检查与按子 URL 控制。
---

# MineMap 3D Tiles Runtime Control

## Architecture Positioning

这类能力不等于“普通加载 3D Tiles”。它专门处理运行时控制：

-   `sourceLoaded` / 加载完成检查
-   `setTileFeatureStyle(fn)`
-   TileFeature 属性查询与显隐/着色
-   `setShowByUrlName(...)`
-   3D Tiles 实例化拾取反馈

## Source-backed Entry Surface

-   `map.addSceneComponent({ type: '3d-tiles', ... })`
-   `sourceLoaded(tileset)`
-   `map.areTilesLoaded()`
-   `map.getFeatureByGPUPick()` / `map.getFeatureByGPUPickAsync()`
-   `SceneTileset#setTileFeatureStyle(fn|null)`
-   `SceneTileset#setShowByUrlName(value, name)`

## Demo-backed Patterns

### 模式 1：按 feature 属性做颜色/显隐

对应 demo：

-   `demo/html/3DTilesStyled.html`
-   `demo/html/3DTilesStyled1.html`

稳定做法：

1. 在样式函数里读取 `feature.getProperties()`
2. 只改 `feature.color` / `feature.show`
3. 通过 `tileset.setTileFeatureStyle(fn)` 应用
4. 关闭时传 `null`

不要直接改 `tileFeatureStyleUpdateFunction`。

### 模式 2：拾取实例并做高亮/属性面板

对应 demo：

-   `demo/html/3DTilesForInstances.html`
-   `demo/html/3DTilesNextPick.html`

稳定做法：

-   用 `getFeatureByGPUPick` 获取目标
-   用 `getProperty(...)` 或 `getProperties()` 读属性
-   运行时暂存旧颜色，再回写高亮颜色

### 模式 3：判断 tiles 是否真正加载完成

对应 demo：

-   `demo/html/3DTilesLoaded.html`

有两个层次：

-   `sourceLoaded`：源对象可用
-   `map.areTilesLoaded()`：当前视图瓦片是否完成

两者不要混成一个概念。

### 模式 4：多 URL 组合 tileset 的子对象开关

对应 demo：

-   `demo/html/3DTilesStyled1.html`

当 `urls` 模式启用后，可用：

-   `tileset.setShowByUrlName(show, name)`

它适合做分组显隐，不是逐 feature 样式替代品。

## Strict Constraints

### 1. TileFeature 样式函数只做轻量逻辑

源码会对选中瓦片内 feature 循环调用样式函数。不要在里面塞重计算或异步逻辑。

### 2. `setTileFeatureStyle(null)` 才是正式关闭方式

不要直接清内部状态。

### 3. 3D Tiles 查询优先 GPU pick

对 3D Tiles 不要退回二维 `queryRenderedFeatures()` 心智。

## Failure Cases

-   用私有字段替代 `getProperties()` / `getProperty()`
-   把 `sourceLoaded` 误当成全部瓦片可见完成
-   在样式函数里做网络请求或大计算

## Demo References

-   `demo/html/3DTilesForInstances.html`
-   `demo/html/3DTilesStyled.html`
-   `demo/html/3DTilesStyled1.html`
-   `demo/html/3DTilesLoaded.html`
-   `demo/html/3DTilesNextPick.html`

## See Also

-   `minemap-scene-components`
-   `minemap-events-and-picking`
-   `minemap-business-oblique-bim`
