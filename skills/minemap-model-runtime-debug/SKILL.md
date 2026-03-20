---
name: minemap-model-runtime-debug
description: MineMap 模型运行时调试规范。用于爆炸视图、材质/纹理更新、线框显示、属性外挂、运行时查看与距离可见性控制。
---

# MineMap Model Runtime Debug

## Architecture Positioning

这类能力主要来自 query/interaction demo，但并不全是内核“业务主 API”。需要分三层看：

1. 正式公开能力：`SceneModel`、材质、纹理、`distanceDisplayCondition`
2. 正式可用但偏调试/展示：`accurateBounding`、线框开关
3. demo 组织手法：爆炸视图、模型浏览器、属性外挂面板

## Source-backed Entry Surface

-   `map.addSceneComponent({ type: '3d-model', ... })`
-   `loaded(model)`
-   `SceneModel#accurateBounding`
-   `SceneModel#distanceDisplayCondition`
-   `SceneModel#showEmbeddedWireframe`
-   `SceneModel#generateWireframe`
-   `SceneModel#showGeneratedWireframe`
-   `material.setOptions(...)`
-   `minemap.TextureLoader.load(...)`

## Demo-backed Patterns

### 模式 1：运行时改模型材质/纹理

对应 demo：

-   `demo/html/UpdateGLTFTexture.html`
-   `demo/html/material-update.html`

推荐顺序：

1. 拿到 `feature.material`
2. 优先 `material.setOptions(...)`
3. 纹理通过 `TextureLoader.load(...)` 异步加载后再设置

### 模式 2：线框显示优先用公开开关

对应 demo：

-   `demo/html/ModelWireframeRender.html`

源码已公开：

-   `showEmbeddedWireframe`
-   `generateWireframe`
-   `showGeneratedWireframe`

不要把 demo 中的私有遍历、节点调试逻辑当成唯一入口。

### 模式 3：动画模型剔除问题用 `accurateBounding`

对应 demo：

-   `demo/html/DistanceBasedVisibilityCulling.html`
-   `demo/html/CameraRegionLimiter.html`

`accurateBounding` 的定位很明确：

-   更准的包围盒
-   更高的运行时成本

只在动画/变形模型确实出现剔除异常时启用。

### 模式 4：爆炸视图与模型查看器属于 demo 工作流

对应 demo：

-   `demo/html/3DModelExplodedView.html`
-   `demo/html/3DModelViewer.html`

这类页面说明 MineMap 支持：

-   按部件组织模型交互
-   运行时做部件显隐/抽离
-   用拾取结果驱动说明面板

但 demo 中存在私有字段访问，技能库不把它写成正式推荐 API。

### 模式 5：属性外挂展示本质是“外部属性表 + 拾取结果映射”

对应 demo：

-   `demo/html/Fbx-PropAttr-WriteShow.html`
-   `demo/html/BIM2GLTF-AttachInfo.html`

结论：

-   外挂属性不是引擎自动建模能力
-   仍需用拾取标识把外部属性映射回当前构件

## Strict Constraints

### 1. 材质更新优先公开 `material` 接口

不要把 `primitive._material` 之类私有字段写成标准方案。

### 2. 线框能力可用，但属于调试/表达增强

不要默认给全部模型开启 `generateWireframe`。

### 3. `distanceDisplayCondition` 是距离显示控制，不是 LOD 系统

它只负责近远可见范围。

## Failure Cases

-   每次点击都重新整模型加载，只为改颜色
-   为了解决剔除问题，默认全局打开 `accurateBounding`
-   把 demo 私有字段访问当成正式 API 复制到业务代码

## Demo References

-   `demo/html/UpdateGLTFTexture.html`
-   `demo/html/material-update.html`
-   `demo/html/ModelWireframeRender.html`
-   `demo/html/3DModelExplodedView.html`
-   `demo/html/3DModelViewer.html`
-   `demo/html/Fbx-PropAttr-WriteShow.html`
-   `demo/html/DistanceBasedVisibilityCulling.html`
-   `demo/html/PBR-GLTF-3DTiles.html`

## See Also

-   `minemap-scene-components`
-   `minemap-material-system-and-shading`
-   `minemap-transforms`
-   `minemap-business-oblique-bim`
