# Spark + MineMap demo adapter

本参考只覆盖仓库 demo 中的集成方式。`@sparkjsdev/spark` 是外部依赖；`SparkThreeLayer` 位于 `demo/js/minemap-three-plugin/`，不是 `source/index.js` 导出的 MineMap 公共类。

当前 demo 的结构是：

1. 创建 WebGL2 地图，并把 `ThreeLayer` 适配层加到 map。
2. 从适配层取得 Three.js renderer 和 scene root。
3. 使用共享的 renderer 创建 `SparkRenderer`，再把它挂到 scene root。
4. 使用 Spark 的 `SplatMesh` 加载业务自己的 SPLAT / SPZ 资源。
5. Spark 内容变更时调用 `SparkThreeLayer.requestSparkUpdate()`，由适配层请求地图重绘。
6. 地图移除时停止自建动画循环并释放 Spark 资源。

不要把这一流程写成 `map.addSceneComponent({ type: "spark" })` 或称为 MineMap 内建 3DGS loader。Spark 的 renderer、mesh、LOD 选项和版本兼容性由 Spark 及项目依赖管理；MineMap 负责与地图渲染上下文和相机同步。

示例 `SparkDemoCommon.js` 当前关闭了 `enableLod` 和 `enableDriveLod`，因此不能据此声称该 demo 已启用 Spark LOD。用户需要 LOD 时，应按其安装的 Spark 版本检查官方选项并单独验证。

使用时还应：

- 保持同一个 WebGL2 上下文，不要另建 renderer 接管 MineMap 的 canvas。
- 不要在 MineMap 的同步 render 回调中递归触发 `map.triggerRepaint()`；由适配层的 dirty/update 流程安排重绘。
- 移除 layer 时恢复外部 `onDirty` 回调，并释放 Spark render target、mesh 与 renderer。
- 用自己的可访问资源地址和配置替换 demo 数据与凭据；不要复制示例里的服务地址或 key。

源码依据：`demo/js/minemap-three-plugin/SparkThreeLayer.js`、`SparkDemoCommon.js`、`demo/html/Threejs-3DGS-Spark.html`。
