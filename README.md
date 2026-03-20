# MineMap Skills（基于 MineMap 4.22 源码）

本目录按 `threejs-skills-main` 的结构，整理了 MineMap 地图引擎的技能规范文件（`SKILL.md`）。

目标：让智能编码助手在生成 MineMap 代码时，优先使用**源码一致**、**版本匹配**、**可直接落地**的 API 与模式。

当前整理状态：

-   基础技能包：已完成首轮整理
-   业务技能包：已完成源码校验与 demo 回填的主干版本
-   当前文档策略：优先写“推荐做法、禁用做法、失败案例、demo 证据”

## Purpose

与 `threejs-skills-main` 一样，这套技能库的目的不是泛泛介绍概念，而是补足编码助手在以下方面的上下文：

-   MineMap 公开 API 的真实入口
-   当前版本有效的参数结构
-   更推荐的新路径与应避免的旧路径
-   已被 `source/` 与 `demo/` 双重验证的交互模式

因此，这套文档默认优先回答三类问题：

1. 这个能力在 MineMap 里应该用哪个类或入口？
2. 这个能力有哪些真实约束和易错点？
3. 这个能力在 demo 里已经有哪些可直接复用的组织方式？

## 版本与校验基线

-   引擎版本：`minemap-3d-engine@4.22.1`
-   核心校验源码：
    -   `source/api/map.js`
    -   `source/style/style.js`
    -   `source/index.js`
    -   `source/util/createWebglContext.js`
    -   `docs/src-cn/*.js`（官方 API 注释源）

## Skills 列表

| Skill                                       | 说明                                                                                |
| ------------------------------------------- | ----------------------------------------------------------------------------------- |
| `minemap-fundamentals`                      | Map 初始化、生命周期、控件与相机基础                                                |
| `minemap-widget-and-controls`               | `IControl` 控件协议、内建 widget、调试/编辑控件与自定义控件组织                     |
| `minemap-style-and-data`                    | Style、Source、Layer、图像资源与数据更新                                            |
| `minemap-layer-system`                      | 图层类型、增删改移、过滤、缩放范围、渲染顺序与图层交互组织                          |
| `minemap-style-system`                      | style v8 结构、`setStyle()`、diff/keepUserInfo、sprite/glyphs 与样式工作流          |
| `minemap-2d-3d-overlay-and-classification`  | 二三维叠加、贴地、贴模型、自动赋高与分类控制                                        |
| `minemap-scene-components`                  | `addSceneComponent` 体系（3d-model / 3d-tiles / scene-object）                      |
| `minemap-3d-tiles-runtime-control`          | 3D Tiles 加载完成、TileFeature 样式、按属性显隐/着色、按子 URL 控制与实例拾取       |
| `minemap-model-runtime-debug`               | 模型爆炸视图、材质/纹理更新、线框显示、属性外挂、运行时调试与可见性控制             |
| `minemap-primitives-and-materials`          | Primitive、几何体、材质与低层三维对象                                               |
| `minemap-collision-and-bounding`            | AABB/OBB/包围球相交检测、BVH 碰撞、角色碰撞与包围体调试                             |
| `minemap-transforms`                        | 坐标系变换、锚点矩阵、姿态矩阵与固定坐标变换                                        |
| `minemap-math-foundations`                  | `Vector` / `Matrix` / `Quaternion` / `HeadingPitchRoll` / `Cartographic` 等数学基础 |
| `minemap-lighting-and-shadows`              | 光照开关、太阳时间、灯光对象与阴影控制                                              |
| `minemap-material-system-and-shading`       | Standard / Phong / Physical / Lambert / Polyline 等材质选型与调参                   |
| `minemap-primitive-adapt-terrain`           | Primitive 贴地、分类联动、栅格深度测试与低层贴地约束                                |
| `minemap-screen-space-reflection`           | SSR 开关、调参边界、与水面/模型反射的关系                                           |
| `minemap-water-surface-and-refraction`      | 水面材质、水体 fill layer、折射/反射控制与渲染模式                                  |
| `minemap-events-and-picking`                | 事件系统、图层事件代理、GPU 拾取                                                    |
| `minemap-terrain-and-analysis`              | 地形、DEM 查询、空间分析容器                                                        |
| `minemap-performance-and-backend`           | WebGL 后端选择、降级策略、性能实践                                                  |
| `minemap-post-process-and-render-pipeline`  | 后处理阶段、效果链编排与运行时启停                                                  |
| `minemap-anti-aliasing`                     | MSAA / FXAA / TAA / `fill-antialias` 的入口、取舍与画质约束                         |
| `minemap-camera-constraints-and-screenshot` | 相机近裁剪面阈值、区域限制、截图导出与相机交互约束                                  |
| `minemap-plugin-compare-swipe`              | 卷帘对比 Compare 插件、双地图同步与销毁约束                                         |
| `minemap-plugin-edit`                       | 外挂编辑插件 `minemap.edit.init(...)`、要素绘制编辑、样式覆写与锁定控制             |
| `minemap-plugin-echarts-integration`        | MineMap + ECharts 叠加层、地理坐标转屏幕坐标、同步更新与交互桥接                    |
| `minemap-plugin-demo-editor`                | demo `editor-map.html` 在线编辑器、Monaco + iframe 预览、WebGL 模式切换与试跑台组织 |

## `source/index.js` 对外导出覆盖结论

结论：**主干开发类已基本覆盖，但还不是“逐个导出类都已经有明确落点”的状态。**

当前覆盖情况可分三层理解：

### 已有明确技能落点

-   `Map`、生命周期、基础控件、`Marker`、`Popup`
-   style/source/layer 数据工作流
-   `SceneModel`、`SceneTileset`、`VideoProjection`
-   `Primitive`、常见 `Geometry` / `Material`
-   `Viewshed3D`、`Sightline`、`Measurement3D`
-   `FloodAnalysis`、`FloodAnalysisOffline`、`ShadowAnalysis`
-   `AnimationTracking`、`AnimationBlender`、关键帧轨迹体系
-   `ParticleSystem`、`ParticleSystemLoader`、发射器体系
-   `BeiDouGrid`、`AdminDivision`、`GaussianSplatting`、`AirLine`

### 已有主题覆盖，但此前不够显式

-   `ModelTransformationControl`、`BatchAddInstanceControl`、`EarthRotationControl`、`Thumbnail`、`FPSControl`
-   `ClippingPlane`、`ClippingPlaneCollection`
-   `Sun`、`Earth`、`Skybox`、`Panorama`、`TerrainLeveling`、`GradientLine`
-   `Highlight`、`CutFill`、`Excavation`、`ViewDome`、`Interference`、`InterferenceConjoined`、`Profile`、`Player`
-   `TextureLoader`、`TextureCubeLoader`、`Texture`、`TextureCube`
-   `WMTSCapabilities`、`optionsFromCapabilities`、`setRTLTextPlugin()`、`getRTLTextPluginStatus()`
-   `PostProcessStageLibrary`、`PostProcessStage`、`PostProcessStageComposite`

### 仍然只做附属说明，不单独拆包

以下导出属于“公开可用，但通常不是业务第一入口”，当前文档只做挂靠说明，不单独扩成独立 skill：

-   相机 / 视锥 / 包围体 / 数学工具族
-   光照类型族（`AmbientLight`、`DirectionalLight`、`PointLight`、`SpotLight`、`SunLight`）
-   `RenderState`、`WMTSCapabilities` 等低层或辅助对象
-   全局配置导出（`accessToken`、`domainUrl`、`dataDomainUrl`、`serverDomainUrl` 等）

本轮已把这些缺口补到现有基础包里，而不是继续制造过多碎片技能包。

## How It Works

可按请求主题加载对应技能包：

-   初始化地图、控件、生命周期 → `minemap-fundamentals`
-   控件体系、widget、自定义控件 → `minemap-widget-and-controls`
-   图层、数据源、样式更新 → `minemap-style-and-data`
-   图层系统与图层顺序控制 → `minemap-layer-system`
-   style 结构、切换与样式工作流 → `minemap-style-system`
-   二维图层与三维场景叠加 → `minemap-2d-3d-overlay-and-classification`
-   模型、3D Tiles、场景对象 → `minemap-scene-components`
-   3D Tiles 运行时着色、显隐与属性查询 → `minemap-3d-tiles-runtime-control`
-   模型运行时调试、线框、材质/纹理更新 → `minemap-model-runtime-debug`
-   Primitive、几何、材质 → `minemap-primitives-and-materials`
-   包围体、碰撞检测、BVH 角色碰撞 → `minemap-collision-and-bounding`
-   坐标变换与锚点矩阵 → `minemap-transforms`
-   数学对象与姿态表达 → `minemap-math-foundations`
-   光照、阴影 → `minemap-lighting-and-shadows`
-   材质选型与着色 → `minemap-material-system-and-shading`
-   Primitive 贴地 → `minemap-primitive-adapt-terrain`
-   屏幕空间反射（SSR） → `minemap-screen-space-reflection`
-   水面材质、水体图层、折射/反射 → `minemap-water-surface-and-refraction`
-   事件、hover、点击、GPU 拾取 → `minemap-events-and-picking`
-   地形、量算、分析容器 → `minemap-terrain-and-analysis`
-   后端、阴影、降级与性能 → `minemap-performance-and-backend`
-   后处理链、描边、夜视、雨雪、雾效 → `minemap-post-process-and-render-pipeline`
-   抗锯齿、边缘平滑与图像质量 → `minemap-anti-aliasing`
-   相机限制、近裁剪面与截图 → `minemap-camera-constraints-and-screenshot`
-   卷帘对比 → `minemap-plugin-compare-swipe`
-   编辑插件 → `minemap-plugin-edit`
-   ECharts 插件叠加 → `minemap-plugin-echarts-integration`
-   示例编辑器 / 在线试跑台 → `minemap-plugin-demo-editor`

业务问题再按专项技能包补充，例如：

-   视频投影 → `minemap-business-video-projection`
-   可视域 / 通视 → `minemap-business-visibility-analysis`
-   淹没 / 阴影 → `minemap-business-flood-and-shadow`
-   漫游 / 跟踪 → `minemap-business-roaming-and-tracking`

## 业务技能包规划（按源码能力拆分）

以下业务包不是拍脑袋归类，而是沿着公开导出类、`Map` 能力入口、`docs/src-cn` 注释源、核心实现以及 `demo/html` 真实案例整理出来的。

| Skill                                           | 业务方向                                             | 主要源码依据                                                                                            | 已纳入的 demo 证据                                                                                                                                     |
| ----------------------------------------------- | ---------------------------------------------------- | ------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `minemap-business-oblique-bim`                  | 倾斜摄影 / BIM / glTF / 3D Tiles 装载                | `source/api/map.js`、`docs/src-cn/SceneModel.js`、`docs/src-cn/SceneTileset.js`                         | `demo/html/3DTiles*.html`、BIM 相关示例                                                                                                                |
| `minemap-business-roaming-and-tracking`         | 轨迹动画、相机跟踪、巡检漫游                         | `docs/src-cn/AnimationManager.js`、`docs/src-cn/AnimationTracking.js`、`docs/src-cn/doc-map-migrate.js` | `AnimationTrack.html`、`AutoHeightAnimation*.html`                                                                                                     |
| `minemap-business-video-projection`             | 视频投射、监控视域、视频管理                         | `docs/src-cn/VideoProjection.js`、`docs/src-cn/VideoManager.js`                                         | `VideoProjection.html`、`VideoProjectionOnWall.html`、`VideoProjectionOnRaster.html`                                                                   |
| `minemap-business-visibility-analysis`          | 可视域、通视、视线判定                               | `docs/src-cn/Viewshed3D.js`、`docs/src-cn/Sightline.js`                                                 | `Viewshed3D.html`、`Sightline.html`                                                                                                                    |
| `minemap-business-flood-and-shadow`             | 淹没分析、阴影分析                                   | `docs/src-cn/FloodAnalysis.js`、`source/renderer/effects/ShadowAnalysis.js`                             | `FloodAnalysis.html`、`FloodAnalysisOffline.html`、`ShadowAnalysis.html`                                                                               |
| `minemap-business-spatial-analysis-suite`       | 地形分析、剖面、限高、挖填方、开挖、开敞度、干扰分析 | `source/api/map.js`、`source/renderer/effects/*.js`                                                     | `DemAnalysis.html`、`Profile.html`、`LimitHeightAnalysis.html`、`CutAndFillAnalysis.html`、`DiggingTerrain.html`、`ViewDome.html`、`Interference.html` |
| `minemap-business-particle-and-special-effects` | 粒子系统与特效编排                                   | `docs/src-cn/ParticleSystem.js`、`source/index.js`                                                      | `ParticleSystemClick.html`、`ParticleSystemFire.html`、`ParticleSystemLoader.html`                                                                     |
| `minemap-business-beidou-grid`                  | 北斗网格码调度与可视化                               | `docs/src-cn/BeiDouGrid.js`                                                                             | `BeiDouGridSceneObject.html`、`BeiDouCodecDemo.html`、`3DTilesWithBeiDouGrid.html`                                                                     |
| `minemap-business-admin-division`               | 行政区划拉伸、面墙边一体化展示                       | `docs/src-cn/AdminDivision.js`                                                                          | `ThematicMapOfAdministrativeDivisionsExt.html`                                                                                                         |
| `minemap-business-gaussian-splatting`           | Gaussian Splatting 高斯点渲染                        | `source/webgl/object/GaussianSplatting.js`                                                              | `3DGaussianSplatting.html`、`3DGS-3DTiles.html`                                                                                                        |
| `minemap-business-airline-and-lines`            | 航线、飞线、动态流动轨迹线                           | `docs/src-cn/AirLine.js`                                                                                | `AirLine.html`                                                                                                                                         |
| `minemap-business-measurement`                  | 三维量算、贴地面积、测距辅助                         | `docs/src-cn/Measurement3D.js`、`source/core/Measurement3D.js`                                          | `Measurement3D.html`、`ClampedMeasurement.html`                                                                                                        |

## Skill File Structure

参考 `threejs-skills-main` 的统一组织方式，MineMap 技能文件尽量遵循以下结构：

```markdown
---
name: skill-name
description: 触发该技能的典型场景
---

# Skill Title

## Quick Start

[最小可用示例]

## Core Concepts / Architecture Positioning

[真实入口、对象边界、参数结构]

## Common Patterns / Demo-backed Patterns

[结合源码与 demo 的落地用法]

## Performance Tips / Performance Discipline

[性能与降级建议]

## See Also

[相关技能包]
```

说明：

-   基础技能包更偏 `Core Concepts`
-   业务技能包更偏 `Architecture Positioning`、`Strict Constraints`、`Failure Cases`、`Demo-backed Patterns`
-   不强求标题字面完全一致，但要求结构职责一致

## 适用原则

1. **优先新接口**：3D 数据优先 `map.addSceneComponent()`，避免继续扩散废弃的 `3d-tiles` source/layer 写法。
2. **先判断加载状态**：关键操作挂到 `load` / `style.load` 后执行，避免时序错误。
3. **围绕渲染后端做兜底**：默认 `auto`，按 `webgpu -> webgl2 -> webgl1` 降级，针对特效做能力判断。
4. **优先可维护写法**：采用引擎公开类（`minemap.SceneModel`、`minemap.SceneTileset`、`minemap.Primitive` 等）而非内部私有对象。

## 推荐用法

将 `minemap-skills/skills` 作为技能库目录，在涉及 MineMap 需求时按主题加载对应 `SKILL.md`。

## 当前文档约束

1. 只推荐公开 API，不把 demo 中的私有字段调试写法当正式方案。
2. 只推荐新三维入口，不继续扩散历史废弃路径。
3. 凡是已有 demo 证据的能力，优先采用 demo 中已验证的参数组织和交互流程。

## Verification

本技能库当前的合理化处理原则如下：

-   **缺少的补充**：优先补源码真实入口、demo 已验证流程、失败案例、禁用写法
-   **冗余的删除**：删除与当前版本不符、重复表达、只停留在概念层但无助于编码的内容
-   **不合理的改掉**：修正错误源码依据、修正把私有字段当公开 API 的表述、修正旧路径继续扩散的问题

主要校验依据：

-   `source/api/map.js`
-   `source/index.js`
-   `source/style/style.js`
-   `source/core/Measurement3D.js`
-   `source/webgl/object/GaussianSplatting.js`
-   `source/renderer/effects/ShadowAnalysis.js`
-   `demo/html/*.html`

后续若继续扩充，仍按同一规则处理：**先核源码，再核 demo，最后入技能库**。
