---
name: minemap-business-particle-and-special-effects
description: MineMap 粒子系统与特效业务规范。用于火焰、烟雾、喷泉、告警特效、氛围效果。
---

# MineMap Business: Particle and Special Effects

## Architecture Positioning

MineMap 粒子系统不是 style layer，而是独立 3D 效果体系，典型入口有三类：

1. `new minemap.ParticleSystem(...)`
2. `new minemap.ParticleSystemLoader(map).load(json, callback)`
3. `map.addParticleSystem(...)` / `map.restartAllParticleSystems()`

因此它更接近“实时特效对象”，而不是普通数据图层。

## Source-backed Rules

-   粒子系统由 `ParticleSystem`、发射器、行为器、函数生成器共同构成
-   地图侧通过 `map.addParticleSystem()` 挂载，不走 `addSceneComponent()`
-   JSON 粒子可通过 `ParticleSystemLoader` 反序列化
-   非循环粒子在 JSON 方案下，业务往往要主动重启；demo 里就是通过 `map.restartAllParticleSystems()` 定时恢复播放
-   粒子也可以参与拾取；demo 里通过 `allowPick: true` + `map.getFeatureByGPUPick()` 进行点选

## Demo-backed Patterns

### 模式 1：发射器能力清单验证

对应 demo：`demo/html/ParticleSystemClick.html`

这个 demo 很有价值，因为它不是单一特效，而是直接把常见发射器都摆出来了：

-   `PointEmitter`
-   `RectangleEmitter`
-   `SphereEmitter`
-   `ConeEmitter`
-   `HemisphereEmitter`
-   `CircleEmitter`
-   `DonutEmitter`
-   `GridEmitter`

推荐把它当成“选型基线”：

-   向上喷发：`ConeEmitter`
-   面状持续释放：`RectangleEmitter` / `GridEmitter`
-   球状爆裂：`SphereEmitter`
-   环状告警：`CircleEmitter` / `DonutEmitter`

### 模式 2：程序化火焰

对应 demo：`demo/html/ParticleSystemFire.html`

这个 demo 展示了 MineMap 粒子系统真正的业务表达方式：

-   贴图：`TextureLoader.load()`
-   发射器：锥体
-   发射节奏：`emissionOverTime + emissionBursts`
-   初始属性：`startLife`、`startSpeed`、`startSize`、`startColor`
-   行为：
    -   `SizeOverLife`
    -   `ColorOverLife`
    -   `ApplyForce`
    -   `RotationOverLife`
    -   `TurbulenceField`

也就是说，业务上“火焰、烟雾、电弧、喷泉”并不是不同组件，而是同一组件在不同参数组合下的结果。

### 模式 3：JSON 装载特效

对应 demo：`demo/html/ParticleSystemLoader.html`

适合：

-   美术导出配置后交给前端接入
-   同一类特效多点复用
-   需要做配置库、模板库、预设库

## Recommended Workflow

1. 先选发射器形状
2. 再定世界坐标还是局部坐标
    - 固定在地理位置：通常 `modelMatrix + worldSpace: true`
3. 再定发射节奏
    - 常驻：`looping: true`
    - 爆发：`emissionBursts`
4. 再定视觉贴图与材质
5. 最后叠加行为器，控制生命周期内的尺寸、颜色、旋转、受力与扰动

## Strict Constraints

### 1. 粒子锚点要先转世界矩阵

demo 里统一通过 `minemap.Transforms.cartographicToFixedFrame(...)` 生成 `modelMatrix`。这说明业务上不要直接把经纬高数组硬塞给粒子系统本体。

### 2. `worldSpace` 要和业务语义一致

-   烟囱、喷泉、火点这类固定世界位置特效，通常要 `worldSpace: true`
-   如果希望效果随宿主局部变换联动，再考虑局部空间

### 3. 非循环 JSON 粒子默认不会自己无限播

`ParticleSystemLoader.html` 已明确演示：当配置里的 `looping` 为 `false` 时，需要业务自己重启。

### 4. 拾取不是默认免费能力

若要做粒子点击反馈，需要像 demo 一样显式打开 `allowPick: true`。

## Failure Cases

### 失败场景 1：拿粒子替代普通点图层

不合适。粒子系统擅长的是：

-   氛围
-   告警
-   动效强调

不擅长：

-   大规模静态点分布表达
-   属性驱动的海量要素渲染

### 失败场景 2：每个点都创建独立重特效

会迅速遇到：

-   更新压力大
-   贴图重复
-   GPU 片元开销高

### 失败场景 3：贴图和行为全都拉满

特别是透明、双面、湍流、受力叠加后，成本会显著上升。低端设备必须降配。

## Professional Parameter Guidance

-   `maxParticle`：先定预算，再做视觉妥协
-   `emissionOverTime`：控制持续密度
-   `emissionBursts`：做脉冲感、爆发感
-   `startLife`：决定拖尾与存留时间
-   `startSpeed`：决定喷射感、扩散感
-   `startSize`：决定主体存在感
-   `renderMode`：Billboard 是最常见方案

## Performance Discipline

-   同类特效优先复用贴图
-   粒子数优先按业务分层启停，不要全局常亮
-   大场景少做“所有对象都冒烟/发光/喷射”
-   用 JSON 方案时先做模板化，避免运行时反复拼复杂参数

## Demo References

-   `demo/html/ParticleSystemClick.html`
-   `demo/html/ParticleSystemFire.html`
-   `demo/html/ParticleSystemLoader.html`
-   `demo/html/ParticleSystemSmoke.html`
-   `demo/html/ParticleSystemRain.html`
-   `demo/html/ParticleSystemSnow.html`

## See Also

-   `minemap-primitives-and-materials`
-   `minemap-events-and-picking`
-   `minemap-performance-and-backend`
